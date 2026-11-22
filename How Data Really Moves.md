# How Data Really Moves

*A friendly, slightly irreverent tour from the wire to your code, in the spirit of Beej’s guides*

Complete draft: Parts I–VI, an epilogue, and Appendices A–F

---

## Preface

Hey, you!

You’ve been typing `cin >> x`, `scanf`, `open()`, `socket()`, and `print()` for a while now. They work. Data goes in, data comes out. But somewhere in the back of your brain there’s a little voice saying: *“Okay, but what is actually happening?”*

Every explanation you’ve found says something like “a stream is a continuous flow of data,” which is about as helpful as saying “a car is a thing that goes.” You asked for a feeling and got a definition. That’s annoying, and I’m sorry.

This guide is the fix. We’re going to take things you already use (streams, files, sockets, the keyboard) and peel them apart until you can *see* the machinery: the buffers, the pointers, the walls between your code and the operating system, the little hardware engines that move bytes while the CPU does something else.

Here’s the deal:

- I’ll explain in plain English first, then show you the real mechanism.
- There will be ASCII diagrams. Lots of them. They are beautiful. Do not make fun of them.
- There will be C code you can compile and run. Run it. Reading about this stuff without poking it is like reading about swimming.
- When something is a simplification, I’ll say so. When real systems differ (Linux vs Windows vs macOS), I’ll mostly describe Linux, because it’s the easiest to poke at.

### Who is this for?

Anybody who writes code and wants to know what’s underneath it. You should be able to read basic C (pointers, arrays, structs). If you know Python or Java instead, you’ll survive: the ideas are identical, only the names change.

### How to read this thing

Start at Chapter 1 and go in order. Each chapter builds on the last. If a chapter ends with a “Try it” section, actually try it.

---

## Roadmap: The Whole Book at a Glance

The book has six parts, an epilogue with ten rules to remember, and six appendices.

**Part I: The Big Illusion**

1. What Is a “Stream,” Really?
2. Buffers and Ring Buffers: The Water Pipe Made of RAM
3. Why Hardware Is Chunky, Slow, and Rude

**Part II: The Walls and the Doors** *(Chapters 4–6)*

4. User Space, Kernel Space, and the Privilege Wall
5. System Calls: Paying the Toll
6. Double Buffering: Why There Are Two Buffers

**Part III: Hardware Gets Involved** *(Chapters 7–9)*

7. Interrupts: The Tap on the Shoulder
8. DMA: Moving Bytes Without Bothering the CPU
9. Polling vs Interrupts vs DMA: A Cage Match

**Part IV: The Three Streams** *(Chapters 10–12)*

10. Input Streams: Keyboards and Files (the Page Cache)
11. Output Streams: `printf`, `flush`, and Why Your Output Shows Up Late
12. Network Streams: Packets, TCP, and the Biggest Illusion of All

**Part V: Pressure, Waiting, and Scale** *(Chapters 13–15)*

13. Blocking vs Non-Blocking I/O
14. Backpressure and Flow Control
15. `select`, `poll`, `epoll`: Watching 100,000 Pipes at Once

**Part VI: Going Fast** *(Chapters 16–18)*

16. Memory-Mapped Files (`mmap`)
17. Zero-Copy and `sendfile`
18. IOMMU, Cache Coherency, and Other Gremlins

**Appendices:** cheat sheets, glossary, `strace` recipes, further reading.

---

# Part I: The Big Illusion

## Chapter 1. What Is a “Stream,” Really?

### 1.1 The lie you’ve been told

Here is the first and most important thing in this entire book:

> **A stream is not a physical flow of data. It is a software illusion built on top of buffers.**

No water is flowing. There’s no pipe. There are only chunks of memory, a couple of pointers, and some careful bookkeeping that makes it *feel* like a smooth, continuous trickle.

Why bother with the illusion? Because it’s *useful*. When you write:

```c
int c;
while ((c = getchar()) != EOF) {
    process(c);
}
```

you don’t want to think about disk sectors, network packets, USB polling intervals, or whether the next byte has arrived yet. You want to say “give me the next byte” and have the universe sort it out. A stream is the abstraction that lets you say that.

### 1.2 The one-sentence definition

> **A stream is an ordered sequence of bytes, consumed (or produced) one piece at a time, with a hidden buffer sitting between the thing that makes the bytes and the thing that uses them.**

Three words matter:

- **Ordered.** Byte 5 always comes after byte 4. You can’t skip around (not in a *pure* stream).
- **Consumed.** Once you read a byte, it’s gone from the stream’s point of view. Its slot can be reused.
- **Buffer.** The secret ingredient. Everything else in this book is about buffers.

### 1.3 Producer and consumer

Every stream has two ends: the **producer** and the **consumer**.

```
   PRODUCER                  BUFFER                  CONSUMER
 (makes bytes)         (holds bytes in RAM)       (uses bytes)

  keyboard    ─────►   [ ... bytes ... ]   ─────►   your program         (input stream)
  your program ───►    [ ... bytes ... ]   ─────►   screen / file / NIC  (output stream)
  remote server ──►    [ ... bytes ... ]   ─────►   your program         (network stream)
```

For an *input* stream, **you** are the consumer. For an *output* stream, **you** are the producer. A network socket is both at once: you consume one stream and produce another, running in opposite directions. (That’s what “bidirectional” means: two streams wearing one trench coat.)

### 1.4 The key insight: producers and consumers don’t keep the same pace

If producer and consumer ran at exactly the same speed, perfectly in sync, you wouldn’t need a buffer. You could hand bytes over directly, like passing a bucket in a fire line. But they never do:

- A keyboard produces maybe 5 bytes a second if you’re typing fast. Your CPU could consume billions.
- A network card might dump 600 KB in 5 milliseconds, then go silent for 50.
- Your program might be paused by the OS for a few milliseconds while it runs your music player.

Producer and consumer are *decoupled in time*. The buffer is what makes that okay. It’s a shock absorber.

### 1.5 Try it: watch a stream refuse to be a stream

```c
#include <stdio.h>

int main(void) {
    int c;
    while ((c = getchar()) != EOF) {
        printf("got: %d\n", c);
    }
    return 0;
}
```

Run it and type `hi`, but **don’t press Enter**. Nothing happens. Why? In a normal terminal, the *terminal driver in the kernel* collects your keystrokes in a line buffer (so backspace works!) and only hands them to your program when you press Enter. You just met your first hidden buffer, and it wasn’t even in your program.

Press Enter. All three bytes (`h`, `i`, and newline, value `10`) arrive at once. Your “stream of keystrokes” turned out to be a *chunk*.

---

## Chapter 2. Buffers and Ring Buffers: The Water Pipe Made of RAM

### 2.1 What a buffer is

A **buffer** is a region of memory used to hold data temporarily while it moves from one place to another. Not magic. An array:

```c
unsigned char buffer[4096];
```

Congratulations, you’ve written a buffer. Now let’s make it behave like a stream.

### 2.2 The naive version and why it’s annoying

Suppose the producer writes from the start of the array and the consumer reads from the start. When the producer reaches the end, you’d have to shift everything left (slow) or stop (bad). The elegant fix: **make the array a circle.**

### 2.3 The ring buffer (circular queue)

Imagine the array bent into a ring, so after the last slot comes the first again. We keep two markers:

- **Write pointer** (*head*): where the producer puts the next byte.
- **Read pointer** (*tail*): where the consumer takes the next byte.

```
              Read Ptr (consumer takes from here)
                  │
                  ▼
   index:   0     1     2     3     4     5     6     7
          ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
          │  __ │ 'H' │ 'i' │ '!' │  __ │  __ │  __ │  __ │
          └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
                                      ▲
                                      │
                          Write Ptr (producer puts next byte here)
```

Bytes `H`, `i`, `!` are *in flight*: written but not yet read. Slots marked `__` are free. The rules:

1. **Producer writes:** put the byte at the write pointer, advance it.
2. **Consumer reads:** take the byte at the read pointer, advance it.
3. **Wrap-around:** past the last slot, a pointer goes back to slot 0.
4. **A byte that’s been read is “gone.”** Its slot can be overwritten later.

That last rule is the one-way, moving-window behavior that creates the *feeling* of flow. The data region is a window sliding around the ring.

### 2.4 The two edge cases (where “pressure” comes from)

**Empty buffer.** The consumer wants a byte but read pointer equals write pointer. Nothing to read, so the consumer must *wait*. In real systems this is **blocking**: the OS puts your thread to sleep until the producer adds something.

**Full buffer.** The producer wants to write but there’s no free slot, so it must *wait* (or drop data, or get an error). This is the origin of **backpressure**: a fast producer gets slowed by a slow consumer. In Chapter 14 we’ll see this reach all the way across the Internet.

```
  EMPTY:  read == write           FULL: every slot occupied
  ┌───┬───┬───┬───┐                ┌───┬───┬───┬───┐
  │ _ │ _ │ _ │ _ │                │ A │ B │ C │ D │   (producer must wait)
  └───┴───┴───┴───┘                └───┴───┴───┴───┘
  (consumer must wait)
```

### 2.5 The sneaky problem: empty and full look the same

If you only check “read == write,” a completely **empty** ring and a completely **full** ring look identical. Classic fixes:

- Keep a separate **count** of bytes in the buffer.
- **Waste one slot** (call it full when `write + 1 == read`).
- Use **free-running counters** that are never wrapped, and wrap only when *indexing*. This is my favorite, and the one we’ll code.

### 2.6 Let’s write one

The free-running-counter trick works beautifully when the capacity is a power of two, because wrapping becomes a bitwise AND instead of a modulo.

```c
#include <stdint.h>
#include <stddef.h>
#include <stdio.h>

#define CAP 8            /* must be a power of two */

typedef struct {
    uint8_t data[CAP];
    size_t  head;        /* total bytes ever written (write counter) */
    size_t  tail;        /* total bytes ever read    (read counter)  */
} ring_t;

static size_t ring_count(const ring_t *r) {
    return r->head - r->tail;          /* bytes currently stored */
}

/* Returns 0 on success, -1 if full. */
static int ring_put(ring_t *r, uint8_t b) {
    if (ring_count(r) == CAP) return -1;     /* FULL: producer must wait */
    r->data[r->head & (CAP - 1)] = b;        /* & (CAP-1) is the wrap-around */
    r->head++;
    return 0;
}

/* Returns 0 on success, -1 if empty. */
static int ring_get(ring_t *r, uint8_t *b) {
    if (ring_count(r) == 0) return -1;       /* EMPTY: consumer must wait */
    *b = r->data[r->tail & (CAP - 1)];
    r->tail++;
    return 0;
}

int main(void) {
    ring_t r = {0};
    const char *msg = "HELLO, STREAM!";   /* 14 chars: more than CAP */
    const char *p = msg;
    uint8_t b;

    while (*p) {
        /* producer: push as many bytes as will fit */
        while (*p && ring_put(&r, (uint8_t)*p) == 0) p++;
        printf("[buffer holds %zu bytes] ", ring_count(&r));

        /* consumer: drain a few bytes */
        for (int i = 0; i < 5 && ring_get(&r, &b) == 0; i++)
            putchar(b);
        putchar('\n');
    }
    while (ring_get(&r, &b) == 0) putchar(b);   /* drain the rest */
    putchar('\n');
    return 0;
}
```

Why does `head - tail` still work after the counters get huge? They’re unsigned (`size_t`), and unsigned arithmetic in C wraps predictably. Even if `head` has overflowed past the maximum and `tail` hasn’t yet, the difference is still the right count. Cute, huh?

Run it. You’ll see the producer fill the 8-byte buffer, the consumer drain 5, the producer refill, and so on. That’s a stream. That’s *the whole trick*.

### 2.7 Ring buffers are everywhere

Once you see them, you can’t unsee them:

- The **kernel’s TCP receive buffer** for each socket.
- The **keyboard driver’s** queue of key events.
- **Audio playback buffers** (underrun = that awful crackle).
- **Network card descriptor rings**, which hold pointers to buffers rather than raw bytes.
- **Linux’s `io_uring`**, named for exactly what it is.

(Real kernel buffers are often fancier, with chunk lists or dynamic resizing, but the *behavior*, a moving window with two cursors, is the same.)

### 2.8 Try it

1. Change `CAP` to 4 and see how the output changes.
2. Make the consumer slower than the producer. Watch `ring_put` start failing. That *is* backpressure.
3. Make the consumer faster than the producer. Watch `ring_get` return `-1`. That *is* blocking.

---

## Chapter 3. Why Hardware Is Chunky, Slow, and Rude

We’ve established that streams are built on buffers. But *why* are buffers necessary? Couldn’t a program just catch each byte as it arrives? Nope, and the reason is physical. Hardware misbehaves in two ways: it moves data in **chunks**, and it runs on a **wildly different clock** than your CPU.

### 3.1 Hardware moves chunks, not bytes

Your code might say “give me one byte,” but the physical world doesn’t work at that granularity.

**Storage devices (SSDs, hard drives):** they read and write fixed units, such as sectors or pages, commonly **512 bytes**, **4 KB**, or larger. Want one byte? The drive still reads the whole unit. It’s like buying the entire egg carton to make one omelet.

**Network cards:** Ethernet and Wi-Fi send discrete **frames/packets**, typically up to about **1,500 bytes** of payload on ordinary Ethernet. They arrive irregularly: your NIC might sit idle for 12 ms, then get several packets back-to-back within microseconds.

**Keyboards:** a USB keyboard doesn’t stream characters. The host controller *polls* it at a fixed interval (commonly 1–8 ms), and the keyboard reports which keys are currently pressed in a small report (8 bytes in the classic boot-protocol layout). Most polls say “nothing new”; then you press a key and there’s a sudden event.

```
  Time ─────────────────────────────────────────────────────►

  What the hardware does:
  ▇▇▇▇        ▇▇                              ▇▇▇▇▇▇▇▇▇▇▇▇
  burst       burst        (silence)          BIG burst

  What your program WANTS to see:
  ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇ ▇
  smooth, steady, one byte at a time
```

The buffer converts the top picture into the bottom picture.

### 3.2 The speed mismatch (a thought experiment)

A modern CPU cycle is roughly **0.3 nanoseconds**. Let’s *scale time* so that one CPU cycle takes **one human second**:

| Event | Real time | Scaled (1 cycle = 1 sec) |
| --- | --- | --- |
| 1 CPU cycle | \~0.3 ns | 1 second |
| RAM access | \~60 ns | \~3 minutes |
| NVMe SSD read | \~50 µs | \~2 days |
| Network packet, same country | \~20 ms | \~2 years |
| Gap between two keystrokes | \~150 ms | \~15 years |

(All numbers are ballpark; the *ratios* are what matter.)

Read that table again. If your CPU were a person waiting for the next keystroke, they’d wait **fifteen years** between letters, and could have done hundreds of millions of other things in the meantime *if allowed to*. So the CPU and the I/O hardware must be **asynchronous**: each runs on its own timeline, and they can never stay in lockstep.

### 3.3 What breaks if there’s no buffer?

Imagine we ripped out every buffer and wired `read()` directly to the hardware. Two disasters can happen, depending on who’s faster at that instant.

**Disaster A: constant stalling (hardware is too slow).** Your program asks for the next byte; the network won’t deliver for another 15 ms (“1.5 CPU years” on our scaled clock). Without a buffer and OS scheduling, the CPU must either **busy-wait** (spin asking “is it here yet? now? now?”, burning power for nothing) or simply freeze.

**Disaster B: dropped data (your program is busy).** Now flip it. Your program is parsing JSON, or the OS paused it for 5 ms to run your browser. A gigabit link delivers roughly 125 MB/s, which is about **600 KB in 5 ms**. The network card has only a small on-board memory (size varies by card). If nobody drains it, it fills fast and the next packet has nowhere to go: a **receive overrun**. Those bytes are lost forever, and your code never knew they existed.

```
   DISASTER A: hardware too slow         DISASTER B: program too slow

   CPU: "byte? byte? byte? byte?..."      NIC: [#######FULL#######]
   NIC: (silence for 15 ms)              NIC: "new packet!" ──✗ nowhere to put it
   CPU: burns 100% doing nothing         CPU: busy elsewhere, data lost
```

### 3.4 The shock absorber: three parts

Modern computers solve both disasters with one trick: put a big, patient buffer in RAM between the hardware and your code, and have the hardware fill it **without** bothering the CPU.

```
 CHAOTIC HARDWARE            SHOCK ABSORBER (RAM)           SMOOTH SOFTWARE
 [NIC / SSD / USB]                                          [Your Program]

  |--burst--| ...silence... |--burst--|
       │                          │
       ▼  (1. DMA writes chunks)  ▼
   ┌─────────────────────────────────────┐
   │ [pkt1][pkt2][pkt3][pkt4][  ][  ]    │ ◄── Kernel ring buffer
   └─────────────────────────────────────┘
       │
       │ (2. one interrupt: "new data landed!")
       ▼
   [OS kernel] ── wakes your thread ──► (3. your code reads at its own pace)
```

1. **DMA (Direct Memory Access):** the device writes whole bursts straight into RAM, no CPU copying required. (Chapter 8.)
2. **Interrupt:** once a batch is safely in RAM, the device sends one electrical “hey!” to the CPU. (Chapter 7.)
3. **Decoupled reading:** even if your program is frozen for 10 ms, the RAM buffer quietly absorbs everything. When you wake up, you drain it at your own pace. Smooth stream achieved.

That’s the shape of the whole book. Every later chapter zooms in on one piece of this picture.

### 3.5 Recap

- A stream is an illusion made of a **buffer + two cursors**.
- Hardware produces **bursts in chunks**, on a timescale wildly different from the CPU’s.
- Without buffers you get **stalling** (too slow) or **data loss** (too fast, or you’re busy).
- Buffers live in RAM, are filled by hardware via **DMA**, announced by **interrupts**, and drained by your code at its own pace.

### 3.6 Try it

Count how often your program really talks to the kernel (we’ll decode this fully in Chapter 5):

```c
#include <stdio.h>

int main(void) {
    FILE *f = fopen("/etc/services", "r");   /* any biggish text file */
    if (!f) { perror("fopen"); return 1; }
    long n = 0;
    int c;
    while ((c = fgetc(f)) != EOF) n++;      /* one byte at a time! */
    printf("%ld bytes read\n", n);
    fclose(f);
    return 0;
}
```

Then:

```
$ gcc read1.c -o read1
$ strace -c ./read1
```

You read hundreds of thousands of bytes one at a time, yet `strace` will show only a few dozen `read` system calls (the exact number depends on the file and your C library). The C library’s hidden **user-space buffer** is absorbing the difference. That’s what Chapters 4 and 6 are about.

---

*End of Part 1. Next up: Part II, where we build the wall between your program and the kernel, and learn what it costs to cross it.*

---

*Draft 2 (Part II: Chapters 4–6)*

# Part II: The Walls and the Doors

In Part I we learned that a stream is a buffer with two cursors. But we kept waving our hands about *where* those buffers live and *who* is allowed to touch them. Now we fix that. There’s a wall in your computer, and every byte that reaches your program has to pass through a door in it.

---

## Chapter 4. User Space, Kernel Space, and the Privilege Wall

### 4.1 Two zones

Your RAM is not one big happy field. The CPU itself enforces a split into two zones:

- **Kernel space:** privileged memory where the operating system and device drivers live. Code running here can do *anything*: talk to hardware, change memory maps, read any process’s data.
- **User space:** restricted memory where your programs live. Chrome, VS Code, your C++ code, your Python script: all of them are guests here.

This isn’t an operating-system *policy* that a clever program could talk its way around. It’s baked into the **CPU hardware**. On x86 the privilege levels are called **rings**: the kernel runs in **Ring 0** (most privileged) and normal programs run in **Ring 3** (least privileged). ARM calls its levels *exception levels* (user code runs at EL0, the kernel at EL1), but the idea is the same.

```
  HARDWARE            KERNEL SPACE (Ring 0)           USER SPACE (Ring 3)

  [SSD / NIC]  <--->  [ OS + drivers ]   ||   [ your program ]
                      [ kernel buffers ] ||   [ your buffers ]
                                         ||
                                  PRIVILEGE WALL
                       (enforced by the CPU, not by politeness)
```

### 4.2 What the wall actually forbids

When your program runs in Ring 3, the CPU simply refuses to:

1. **Execute privileged instructions** (like talking directly to I/O ports, halting the CPU, or changing the page tables that define memory maps). Try, and the CPU raises a fault and the OS kills your process.
2. **Touch kernel memory.** Every memory page has permission bits. Kernel pages are marked “supervisor only,” so a user-mode load or store to them faults.
3. **Touch other processes’ memory.** Each process gets its own *virtual* address space, and the kernel only maps in what that process is allowed to see.

That means your program **cannot** read a disk, send a packet, or even read a keypress by itself. It has to *ask the kernel* to do it, politely, through a controlled doorway (Chapter 5).

### 4.3 Why have a wall at all?

Three excellent reasons:

- **Protection:** a buggy or malicious program can’t read your password manager’s memory or overwrite the OS.
- **Stability:** when your program crashes, only *your program* dies, not the whole machine.
- **Sharing:** dozens of programs want the same disk and network card. Someone has to referee. The kernel is the referee, and it can only referee if nobody else is allowed to touch the hardware.

### 4.4 Where the buffers live now

The wall explains why a stream needs *two* kinds of buffers. Data coming from hardware lands first in kernel memory (because only the kernel and drivers may touch hardware), and your program’s own buffer lives on the user side:

```
  HARDWARE              KERNEL SPACE                      USER SPACE

  [SSD / NIC] --DMA--> [ Kernel buffer ]  ==copy==>  [ User buffer ] --> your loop
                       (page cache,         (via a      (stdio, cin,       reads 1 byte
                        socket buffer)      syscall)     BufferedReader)   at a time
                                    |
                              Privilege wall
```

### 4.5 Try it: bump into the wall

On x86-64 Linux, this tries to write to an address in the kernel’s half of the address space:

```c
#include <stdio.h>

int main(void) {
    int *p = (int *)0xffff800000000000UL;   /* a kernel-half address */
    printf("about to poke the kernel...\n");
    fflush(stdout);
    *p = 42;                                /* boom */
    printf("you will never see this\n");
    return 0;
}
```

You’ll get `Segmentation fault`. That isn’t your C library being grumpy: the **CPU** refused the access, the kernel’s fault handler noticed, and it sent your process the `SIGSEGV` signal. That’s the wall doing its job.

---

## Chapter 5. System Calls: Paying the Toll

### 5.1 The doorway

If the wall is absolute, how does `read()` ever work? Through the one sanctioned doorway: the **system call** (a.k.a. *syscall*).

A syscall is a special request that says: “Kernel, please do this privileged thing on my behalf.” The kernel provides a numbered menu of services: `read`, `write`, `open`, `close`, `socket`, `mmap`, and hundreds more. Your program picks a number, passes some arguments, and asks the CPU to switch into the kernel.

### 5.2 What happens during a syscall

On x86-64 Linux, the dance goes like this:

1. Your code puts the **syscall number** in a register (`rax`) and the **arguments** in other registers (`rdi`, `rsi`, `rdx`, and friends).
2. Your code executes the `syscall` instruction.
3. The CPU **switches from Ring 3 to Ring 0** and jumps to a kernel entry point that the kernel set up at boot. You don’t get to choose where it lands. That’s the whole point.
4. The kernel saves your registers, **validates** the arguments (is that pointer really inside your memory? do you have permission to read this file?), does the work, and puts the result in `rax`.
5. The kernel executes a return instruction (`sysret` or equivalent), the CPU drops back to **Ring 3**, and your code resumes as if nothing happened.

```
   USER MODE (Ring 3)                  KERNEL MODE (Ring 0)

   read(fd, buf, 4096)
      |  rax = 0 (read)
      |  rdi = fd, rsi = buf, rdx = 4096
      |  execute `syscall`  ------------>  entry point
      |                                      save registers
      |                                      check arguments
      |                                      copy data from kernel buffer
      |                                      to your buf
      |  <-----------------------------   `sysret`, rax = bytes read
      v  carry on
```

The exact syscall numbers and register conventions differ per CPU architecture and OS, but every OS has this shape.

### 5.3 Making a raw syscall yourself

The familiar `write()` function is a thin wrapper around this. You can bypass it and call the kernel by number:

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <sys/syscall.h>

int main(void) {
    const char msg[] = "hello from a raw syscall\n";
    syscall(SYS_write, 1, msg, sizeof msg - 1);   /* fd 1 = stdout */
    return 0;
}
```

No `printf`, no buffer, no stdio: just you, a number, and the kernel.

### 5.4 Why the toll is expensive

Crossing the wall is *far* more costly than calling a normal function. The CPU has to switch privilege modes, the kernel has to save and restore state and validate your arguments, and caches and branch predictors get disturbed. Depending on the CPU and on security mitigations (the fixes for bugs like Spectre and Meltdown made syscalls pricier), a trivial syscall can cost from under a hundred nanoseconds to a microsecond or more, which is **hundreds to thousands of CPU cycles**. A normal function call costs a handful of cycles.

So the golden rule of I/O performance is:

> **Make fewer, bigger syscalls.**

### 5.5 Try it: count the toll

Here’s a program that reads a file one byte at a time with raw `read()` calls, and one that reads 4 KB at a time:

```c
/* slow.c: one syscall per byte */
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
    if (argc < 2) { fprintf(stderr, "usage: %s file\n", argv[0]); return 1; }
    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }
    char c;
    long n = 0;
    while (read(fd, &c, 1) == 1) n++;      /* 1 byte per trip across the wall */
    printf("%ld bytes\n", n);
    close(fd);
    return 0;
}
```

```c
/* fast.c: one syscall per 4096 bytes */
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
    if (argc < 2) { fprintf(stderr, "usage: %s file\n", argv[0]); return 1; }
    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }
    char buf[4096];
    long n = 0;
    ssize_t got;
    while ((got = read(fd, buf, sizeof buf)) > 0) n += got;
    printf("%ld bytes\n", n);
    close(fd);
    return 0;
}
```

Make a 1 MB test file and compare:

```
$ head -c 1048576 /dev/urandom > big.bin
$ gcc slow.c -o slow && gcc fast.c -o fast
$ time ./slow big.bin
$ time ./fast big.bin
$ strace -c ./slow big.bin 2>&1 | head
$ strace -c ./fast big.bin 2>&1 | head
```

Expect `slow` to make over a million `read` calls and `fast` to make about 256 (plus a handful for startup). The speed difference is typically dramatic. (Note that running under `strace` itself slows things down a lot. Use `time` without `strace` for fair timing.)

---

## Chapter 6. Double Buffering: Why There Are Two Buffers

### 6.1 The question everyone asks

“Wait. The data already sits in a kernel buffer in RAM. Now we copy it to *another* buffer in RAM, in my program? Isn’t that wasteful?”

It’s a fair question, and the answer is that the two buffers solve **two different problems**.

> **The kernel buffer protects you from slow hardware.** **The user buffer protects you from slow system calls.**

### 6.2 Meet the two buffers

**The kernel-space buffer** is a chunk of RAM controlled by the OS.

- *Who fills it:* hardware (via DMA) and device drivers.
- *Examples:* the **page cache** (for file data coming from disk) and **socket buffers** (for TCP data).
- *Its job:* absorb bursty hardware, reassemble out-of-order packets, and keep recently used disk blocks in RAM so other programs can reuse them without touching the SSD again.

**The user-space buffer** is an ordinary array in your program’s memory, usually created for you by your language’s library.

- *Who fills it:* the kernel, copying data in whenever your program makes a `read()`/`recv()` syscall.
- *Its job:* let your code consume *one byte or one line at a time* without making a syscall for each.

```
  HARDWARE          KERNEL SPACE                       USER SPACE

  [SSD] --DMA--> [ page cache ] ---read() 4 KB--->  [ stdio buffer ] --> getc(), getc(), getc()...
                  (shock absorber                     (toll absorber:
                   for hardware)                       1 syscall feeds
                                                       thousands of getc calls)
```

### 6.3 The numbers

Reading a 1 MB file, one byte at a time, in a loop:

| Approach | What happens | Syscalls |
| --- | --- | --- |
| No user buffer (raw `read(fd, &c, 1)`) | Every iteration crosses the wall | 1,048,576 |
| 4 KB user buffer | One syscall fetches 4,096 bytes; the next 4,095 reads are plain memory loads | 256 |
| 8 KB user buffer | Same idea, bigger gulp | 128 |

Those inner reads are just *pointer increments inside your own memory*: no privilege switch, full CPU speed. That’s the entire reason `fgetc`, `cin`, Python’s `open()`, and Java’s `BufferedReader` all keep a hidden buffer.

### 6.4 Same idea, different names

You’ll find this pattern in every language, with typical buffer sizes in the 4–8 KB range (the exact defaults vary by library and version):

| Language / API | Where the user buffer hides |
| --- | --- |
| C | `stdio` `FILE*` (`fopen`, `fgets`, `printf`) |
| C++ | `std::filebuf` behind `ifstream`/`ofstream`, and `cin`/`cout` |
| Python | `io.BufferedReader` / `BufferedWriter` behind `open()` |
| Java | `BufferedReader`, `BufferedInputStream`, etc. |

### 6.5 Three buffering modes in C

The C library doesn’t buffer everything the same way. Each `FILE*` is in one of three modes:

- **Fully buffered:** flush only when the buffer is full. This is the default for ordinary files and pipes.
- **Line buffered:** flush when a newline is written. This is the default when stdout is a **terminal**.
- **Unbuffered:** every write goes out immediately. This is the default for `stderr`.

You can change the mode with `setvbuf`:

```c
#include <stdio.h>

int main(void) {
    static char mybuf[65536];
    setvbuf(stdout, mybuf, _IOFBF, sizeof mybuf);   /* fully buffered, 64 KB */
    for (int i = 0; i < 100000; i++)
        printf("line %d\n", i);
    return 0;                                       /* buffer flushed at exit */
}
```

### 6.6 Try it: see output buffering with your own eyes

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("step 1... ");     /* no newline! */
    sleep(3);
    printf("step 2\n");       /* newline: line-buffered terminals flush here */
    return 0;
}
```

Run it in a terminal: “step 1...” does **not** appear for 3 seconds, because it’s sitting in the user-space buffer waiting for a newline. Add `fflush(stdout);` after the first `printf` and it appears immediately. Now run `./a.out | cat`: stdout is a pipe, so it’s *fully* buffered, and nothing appears until the program exits. We’ll dissect this in Chapter 11.

### 6.7 The honest downside

Double buffering isn’t free. Data gets **copied twice**: hardware to kernel buffer (by DMA, no CPU involved), then kernel buffer to user buffer (by the CPU, during the syscall). When you’re moving gigabits per second, those copies burn CPU cycles and memory bandwidth. High-throughput systems (web servers, Kafka) use **zero-copy** tricks like `mmap` and `sendfile` to skip the user buffer entirely. That’s Part VI.

For now, hold on to the picture:

```
  HARDWARE ==DMA==> KERNEL BUFFER ==syscall copy==> USER BUFFER --> YOUR CODE
               (no CPU)         (CPU copies)         (plain memory reads)
```

### 6.8 Recap

- The CPU splits memory into **kernel space** and **user space**, and enforces the wall in hardware.
- The only legal doorway is the **system call**, which costs hundreds to thousands of cycles.
- So we use **two buffers**: the kernel buffer shields us from slow hardware, and the user buffer shields us from slow syscalls.
- **Fewer, bigger syscalls** is the golden rule.

---

*End of Part II. Next up: Part III, where we meet interrupts and DMA, the machinery that lets hardware fill those kernel buffers without the CPU lifting a finger.*

---

*Draft 3 (Part III: Chapters 7–9)*

# Part III: Hardware Gets Involved

So far the kernel buffer has been a magic box that “just fills up.” Time to open the box. Who fills it? How does the CPU find out that something arrived? Who does the copying? Three answers: **interrupts**, **DMA**, and a clever compromise between polling and both.

---

## Chapter 7. Interrupts: The Tap on the Shoulder

### 7.1 The “how do I know?” problem

The kernel has a buffer. A network card is about to drop a packet into it. How does the CPU find out?

There are really only two strategies:

- **Polling:** the CPU repeatedly asks the device, “Anything yet? Anything yet? Anything yet?” Simple, but it burns CPU cycles *every time you ask*, including all the times the answer is “no.” (Remember the 15-year wait between keystrokes from Chapter 3.)
- **Interrupts:** the CPU goes about its business, and the **device taps it on the shoulder** when something happens.

Most of the time, modern systems use interrupts. Polling is the awkward kid at the party who keeps asking “are we there yet?” from the back seat.

### 7.2 What an interrupt actually is

An **interrupt** is an electrical signal (or, on modern PCIe devices, a special memory write called an MSI/MSI-X message) that tells a CPU core: “Stop what you’re doing; something needs attention.”

What happens next, in broad strokes:

1. The device raises the interrupt.
2. The CPU **finishes its current instruction** (it doesn’t stop mid-instruction).
3. The CPU **saves a bit of state** (instruction pointer, flags) so it can resume later.
4. It looks up the interrupt number in a table the kernel set up at boot (on x86, the **IDT**, Interrupt Descriptor Table) and **jumps to the registered handler**, which runs in kernel mode.
5. The handler does its (very short) job and returns.
6. The CPU **resumes your program** exactly where it left off. Your program never knows it was interrupted.

```
   CPU running your program               Device
   ------------------------               ------
   instr 1
   instr 2
   instr 3   <------ IRQ ------------     "data arrived!"
   [save state, look up IDT, jump]
     handler: note the event, ack the device
   [restore state]
   instr 4  (continues as if nothing happened)
```

### 7.3 Handlers must be quick: top half and bottom half

While an interrupt handler runs, other work on that core is delayed, so handlers must be **tiny**. Linux splits the job into two phases:

- **Top half (hard IRQ):** do the bare minimum: acknowledge the device, grab critical info, schedule the rest.
- **Bottom half (softirq / tasklet / workqueue):** do the heavier work shortly afterward, with interrupts re-enabled. For networking this is where packets get processed through the protocol stack.

Think of the top half as the receptionist who writes down “Mr. Packet is here” and the bottom half as the staff who actually deal with him.

### 7.4 How an interrupt wakes your sleeping program

Remember blocking from Chapter 2? Your thread called `read()`, the buffer was empty, and the kernel put your thread to **sleep** on a *wait queue* attached to that buffer. Your thread isn’t running, isn’t spinning, isn’t using CPU. It’s just parked.

When data arrives:

```
  1. Device DMAs bytes into the kernel buffer      (Chapter 8)
  2. Device raises an interrupt
  3. Top half runs, schedules the bottom half
  4. Bottom half processes the data, appends it to the buffer
  5. Kernel finds your thread on the wait queue, marks it RUNNABLE
  6. Scheduler picks your thread when a core is free
  7. read() returns; your code continues
```

That’s the entire reason blocking I/O doesn’t waste CPU: the *interrupt* is what wakes you, not you checking.

### 7.5 Interrupt storms and coalescing

One interrupt per packet sounds fine at 100 packets per second. At a million packets per second, the CPU would spend all its time servicing interrupts and have no time left to do anything with the data (a famous failure mode called *receive livelock*).

The fix is **interrupt coalescing** (also called interrupt moderation): the NIC deliberately waits until several packets have arrived, or a tiny timer has expired, and *then* raises one interrupt for the whole batch. You trade a little latency for a lot of throughput.

### 7.6 Try it: spy on the interrupts

On Linux:

```
$ cat /proc/interrupts
$ watch -n1 cat /proc/interrupts
```

You’ll see a table with one row per interrupt source (keyboard, NIC queues, NVMe queues, timers…) and one column per CPU core, with counters ticking upward. Now type on the keyboard, copy a big file, or load a web page, and watch which counters jump. You’re eavesdropping on the tap on the shoulder.

For bottom halves, try `cat /proc/softirqs` and look at the `NET_RX` row while downloading something.

---

## Chapter 8. DMA: Moving Bytes Without Bothering the CPU

### 8.1 The problem DMA solves

Before DMA, getting data from a device into RAM was done by the CPU itself, a method called **Programmed I/O (PIO)**:

1. Read a few bytes from a device register into a CPU register.
2. Write those bytes from the CPU register into RAM.
3. Repeat. Thousands of times. Per transfer.

For a gigabit network card or an NVMe SSD, this would trap a multi-GHz CPU at 100% doing nothing but mindless copy-paste.

```
  WITHOUT DMA (PIO): CPU is the middleman for every chunk
  [ SSD / NIC ] ---> [ CPU registers ] ---> [ RAM ]
                       (busy, busy, busy)

  WITH DMA: CPU sets it up, walks away
  [ SSD / NIC ] ====================================> [ RAM ]
   (has its own DMA engine; CPU is free to run your code)
```

### 8.2 What “DMA” means

**DMA** stands for **Direct Memory Access**: a mechanism that lets an I/O device read from or write to RAM **without the CPU copying each byte**.

On modern machines, high-speed devices (NVMe drives, NICs, GPUs) have their own built-in DMA engines connected over **PCIe**. They act as **bus masters**: they can initiate memory reads and writes on their own, as if they were tiny CPUs that only know how to move data.

### 8.3 The sequence of a DMA transfer

Here’s what happens when a stream pulls data from hardware:

1. **The CPU gives orders.** The device driver (kernel code) writes a small instruction, a **descriptor**, telling the device three things: *where* the data is (or should go), *which RAM address* to use, and *how many bytes* to move.
2. **The CPU walks away.** It goes back to running your program or other kernel work.
3. **The device’s DMA engine does the transfer.** It sends memory-write transactions across PCIe, through the memory controller, straight into the kernel buffer in RAM.
4. **The device rings the bell.** When the 4 KB page or 1,500-byte packet has fully landed, the device raises an **interrupt**: “Done; the buffer is ready.” (This is where Chapter 7 plugs in.)

```
               1. "Copy 4 KB to RAM address X"
   [ CPU ] ----------------------------------> [ DMA engine in device ]
      ^                                                 |
      |                                      2. direct transfer over PCIe
      |                                                 v
      |                                       [ RAM: kernel buffer at X ]
      |                                                 |
      +------------- 3. interrupt: "done!" -------------+
```

### 8.4 Descriptor rings (ring buffers strike again)

In real network cards and NVMe drives, the “instructions” aren’t passed one at a time. They sit in a **ring buffer of descriptors**, the very structure from Chapter 2, living in RAM and shared between the driver and the device.

For receiving packets on a NIC, roughly:

- At startup, the driver allocates many empty packet buffers and fills an **RX ring** with descriptors pointing at them: “put the next incoming packet in this buffer.”
- When a packet arrives, the NIC grabs the next free descriptor, DMAs the packet into the buffer it points to, and marks the descriptor **done**.
- The NIC raises an interrupt (or the driver notices while polling).
- The driver processes completed descriptors, hands the packets up the stack, and posts fresh empty buffers back into the ring.

```
   RX descriptor ring (shared by driver and NIC)

   +--------+--------+--------+--------+--------+--------+
   | DONE   | DONE   | EMPTY  | EMPTY  | EMPTY  | EMPTY  |
   | -> bufA| -> bufB| -> bufC| -> bufD| -> bufE| -> bufF|
   +--------+--------+--------+--------+--------+--------+
       ^                  ^
    driver reads       NIC writes next
    completed          packet into bufC
```

It’s ring buffers all the way down.

### 8.5 Two real-world catches

Because hardware now writes into RAM *behind the CPU’s back*, subtle problems appear.

**Catch 1: stale CPU caches.** If the CPU recently read that RAM address, its fast L1/L2/L3 cache might still hold the *old* bytes. On x86, hardware keeps DMA and CPU caches coherent for ordinary memory. On some other architectures, drivers must explicitly flush or invalidate caches around DMA (Linux has a whole “DMA mapping” API for this). Either way, the kernel makes sure you see fresh bytes, not a ghost of the old ones.

**Catch 2: security.** Letting any PCIe device write to *any* physical address would be a gift to attackers: a malicious or buggy device could overwrite the kernel. So modern systems add an **IOMMU** (Input-Output Memory Management Unit), basically an MMU for devices. The OS tells the IOMMU exactly which memory ranges each device may touch, and anything else gets blocked.

A related detail: devices generally don’t understand your process’s virtual addresses. They use **physical** (or IOMMU-translated “I/O virtual”) addresses, so the kernel must **pin** DMA buffers in memory so they can’t be moved or swapped out mid-transfer.

### 8.6 Try it: look at the rings

On Linux with a wired NIC (replace `eth0` with your interface name, found via `ip link`):

```
$ ethtool -g eth0
```

This shows the **RX and TX ring sizes** (how many descriptors the NIC’s rings hold) and their maximums. You’re looking at the exact shock absorber from Chapter 3. And:

```
$ ethtool -c eth0
$ ls /sys/class/iommu/
```

The first shows interrupt coalescing settings (Chapter 7.5); the second lists active IOMMUs, if your system has one enabled. (Some virtual machines or drivers won’t support these commands. That’s fine.)

---

## Chapter 9. Polling vs Interrupts vs DMA: A Cage Match

### 9.1 They aren’t really competitors

Quick clarification, because this trips everyone up: **DMA and interrupts solve different problems.**

- **DMA** answers: *who moves the bytes?* (The device, not the CPU.)
- **Interrupts vs polling** answer: *how does the CPU learn the move is done?*

So they combine. A typical read from an SSD is **DMA for the data** plus **an interrupt for the notification**.

| Strategy | Who moves the data | How CPU finds out | Good at | Bad at |
| --- | --- | --- | --- | --- |
| PIO + polling | CPU | CPU keeps asking | Tiny, simple devices | Burns CPU; slow |
| PIO + interrupts | CPU | Device taps shoulder | Low-rate devices (keyboard) | CPU still copies every byte |
| DMA + interrupts | Device | Device taps shoulder | Most workloads | Interrupt overhead at extreme rates |
| DMA + polling | Device | CPU checks a flag/ring | Ultra-high rates, ultra-low latency | Burns a core even when idle |

### 9.2 The hybrid: interrupt first, then poll under load

At very high packet rates, interrupts become the problem. At low rates, polling is wasteful. Linux’s networking stack uses a hybrid called **NAPI**:

1. Packets are idle. The NIC uses **interrupts**.
2. A packet arrives. The first interrupt fires; the driver **disables further interrupts** from that queue and schedules a poll.
3. The kernel **polls** the RX ring in batches, pulling many packets per pass, until the ring is empty.
4. Ring empty? **Re-enable interrupts** and go back to sleep.

So: calm traffic costs almost nothing, and floods are handled efficiently in batches. Some extreme systems (like DPDK-based packet processors or high-performance NVMe paths) skip interrupts entirely and **busy-poll** on a dedicated core. They pay with a core pinned at 100% to win the lowest possible latency.

### 9.3 The full journey of a file read

Let’s stitch Chapters 3 through 8 together. Your program calls `read()` on a file whose data isn’t in memory yet:

```
  1. Your code:   read(fd, buf, 4096)               [USER SPACE]
  2. CPU:         `syscall` instruction -> Ring 0   [crosses the wall]
  3. Kernel:      page cache miss. Driver writes a command + descriptor
                  telling the SSD: "read block N into RAM address X"
  4. Kernel:      no data yet, so put your thread to SLEEP, run someone else
  5. SSD:         reads flash, DMAs 4 KB into the page cache (no CPU copying)
  6. SSD:         raises an INTERRUPT: "done"
  7. Kernel:      handler runs, marks your thread RUNNABLE
  8. Scheduler:   your thread gets a core
  9. Kernel:      copies bytes from page cache into your buf, returns   [copy #2]
 10. CPU:         `sysret` -> Ring 3; read() returns 4096
 11. Your code:   consumes bytes from buf one at a time, no syscalls
```

Every idea in the book so far appears in those eleven lines: buffers, the wall, syscalls, blocking, DMA, interrupts, the user-space buffer. If you can narrate that list from memory, you understand low-level I/O better than most working programmers.

### 9.4 Try it: feel polling vs blocking

This program reads stdin either **blocking** (default) or **busy-polling** (if you pass any argument):

```c
/* poll_vs_block.c */
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char **argv) {
    int busy = argc > 1;
    if (busy)   /* make read() return immediately with EAGAIN if no data */
        fcntl(STDIN_FILENO, F_SETFL, fcntl(STDIN_FILENO, F_GETFL) | O_NONBLOCK);

    char c;
    unsigned long spins = 0;
    for (;;) {
        ssize_t n = read(STDIN_FILENO, &c, 1);
        if (n == 1) {
            if (c == '\n') continue;
            printf("got '%c' after %lu empty polls\n", c, spins);
            spins = 0;
            if (c == 'q') break;
        } else if (n < 0 && errno == EAGAIN) {
            spins++;                        /* "anything yet? anything yet?" */
        } else {
            break;                          /* EOF or real error */
        }
    }
    return 0;
}
```

Run it twice (type a letter and press Enter; `q` quits), while watching CPU usage in another terminal with `top`:

```
$ ./poll_vs_block         # blocking: process sleeps, ~0% CPU
$ ./poll_vs_block busy    # polling: one core pegged near 100%
```

In busy mode you’ll also see the “empty polls” counter reach into the millions between keystrokes. That’s the cost of asking instead of being told.

### 9.5 Recap

- **Interrupts** let the CPU ignore a device until it has news. A tiny top half records the event; a bottom half does the heavy lifting.
- **DMA** lets devices copy data into RAM themselves, using descriptor rings (which are ring buffers).
- DMA decides **who moves** the bytes; interrupts vs polling decide **how you find out**.
- Real systems mix them: **DMA + interrupts** normally, switching to **polling** (NAPI, busy-polling) under heavy load.

---

*End of Part III. Next up: Part IV, “The Three Streams,” where we apply all of this to the three cases you originally asked about: input streams (keyboards and files), output streams (why your `printf` shows up late), and network streams (the biggest illusion of all).*

---

*Draft 4 (Part IV: Chapters 10–12)*

# Part IV: The Three Streams

Now we cash in. You asked about three streams: **input**, **output**, and **internet**. We have all the parts: buffers, the wall, syscalls, interrupts, DMA. Let’s follow a byte through each stream from start to finish.

---

## Chapter 10. Input Streams: Keyboards and Files

### 10.1 The keyboard: a stream that’s mostly silence

What happens between your finger hitting a key and `getchar()` returning? On a typical Linux desktop or terminal:

```
  1. Key switch closes; keyboard firmware notices
  2. USB host controller polls the keyboard (every few ms), gets a small report
  3. Host controller DMAs the report into RAM and raises an INTERRUPT
  4. Kernel USB/HID driver decodes it into a key event
  5. Input subsystem delivers the event upward (to your windowing system,
     which feeds a terminal emulator, which writes to a pseudo-terminal)
  6. TTY LINE DISCIPLINE (kernel) collects characters into a line buffer,
     handles echo and backspace
  7. On Enter, the whole line becomes readable
  8. Your blocked read() wakes up, the kernel copies the line to your buffer
  9. stdio hands you one char at a time from ITS buffer
```

(On a physical console or serial line, steps 5–6 are a bit simpler. In a modern terminal emulator there’s an extra middleman, the *pseudo-terminal*, but the kernel’s line discipline still sits in the path.)

That’s **at least three buffers** between your finger and your variable: the device’s report, the kernel’s line buffer, and stdio’s buffer. It’s an input “stream” that’s 99.9% silence, with a few bytes at a time.

### 10.2 Canonical mode: why Enter matters

By default the terminal is in **canonical (cooked) mode**: the kernel’s line discipline buffers a whole line so you can edit it with backspace, and releases it to your program only on Enter. That’s why Chapter 1’s `getchar()` loop sat silent until you pressed Enter.

Games, editors, and tools like `vim` and `less` switch the terminal into **raw (non-canonical) mode**, where each key is delivered immediately:

```c
#include <stdio.h>
#include <termios.h>
#include <unistd.h>

int main(void) {
    struct termios orig, raw;
    tcgetattr(STDIN_FILENO, &orig);          /* remember the old settings */
    raw = orig;
    raw.c_lflag &= ~(ICANON | ECHO);         /* no line buffering, no echo */
    raw.c_cc[VMIN]  = 1;                     /* read() returns after 1 byte */
    raw.c_cc[VTIME] = 0;
    tcsetattr(STDIN_FILENO, TCSANOW, &raw);

    printf("press keys (q quits)\n");
    char c;
    while (read(STDIN_FILENO, &c, 1) == 1 && c != 'q')
        printf("byte %d\n", c);

    tcsetattr(STDIN_FILENO, TCSANOW, &orig); /* ALWAYS restore the terminal */
    return 0;
}
```

Run it. Every keypress shows up instantly, no Enter needed. Now press an **arrow key**: you’ll see three bytes arrive (typically `27 91 65` for Up, i.e. ESC `[` `A`). The “key” was never a single thing: it’s a little byte sequence in the stream. If the program crashes without restoring the terminal, type `reset` to get it back.

### 10.3 Files: the page cache and read-ahead

A file is an input stream too, but the producer (the disk) is far more cooperative than a keyboard: it holds all the data and has no opinion about timing. The interesting part is the kernel’s clever buffering, the **page cache**.

- Whenever the kernel reads file data from disk, it keeps the pages in RAM in the **page cache** (in 4 KB pages). A later read of the same data is served straight from RAM, with no disk access at all.
- The kernel also guesses what you’ll read next. When it sees you reading a file sequentially, it performs **read-ahead**: it asks the disk for the *following* chunks before you request them, so by the time your code gets there, the data is already waiting in RAM.

```
  you read bytes 0..4095     --> kernel reads page 0, notices a sequential pattern
                                 ...and quietly starts pages 1, 2, 3, 4 via DMA
  you read bytes 4096..8191  --> already in page cache. No waiting!
```

That’s a producer that’s been *taught* to stay ahead of the consumer, keeping the buffer non-empty so you never block.

The page cache is also why the second run of anything is faster than the first, and why a Linux machine seems to “use all its RAM”: free memory is mostly cache, and the kernel gives it back instantly when programs need it. (`free -h` shows it in the `buff/cache` column.)

### 10.4 Try it: cold vs warm

```
$ head -c 500000000 /dev/urandom > big500.bin      # ~500 MB file
$ time cat big500.bin > /dev/null                  # likely warm already
$ sync; echo 3 | sudo tee /proc/sys/vm/drop_caches # empty the page cache (needs root)
$ time cat big500.bin > /dev/null                  # COLD: must read from disk
$ time cat big500.bin > /dev/null                  # WARM: served from RAM
```

The cold run is bounded by your disk; the warm runs by memory and syscall overhead. (Dropping caches is a benchmarking trick, not something to do on a production server.)

---

## Chapter 11. Output Streams: `printf`, `flush`, and Why Your Output Shows Up Late

### 11.1 Output is input in reverse, with a twist

For input streams, hardware fills the buffer and *you* drain it. For output streams, **you fill the buffer and something else drains it**: the kernel, then the device. Every rule from earlier applies in mirror image:

- **Full buffer:** you (the producer) block until space frees up.
- **Empty buffer:** the device has nothing to do, and goes idle.
- **Batching:** many small writes get combined into fewer big ones.

The twist is a question input never raises: **“When does my data count as *sent*?”**

### 11.2 The layers of an output stream

```
  YOUR CODE        printf("hi\n")
      |
      v
  USER BUFFER      stdio / cout / BufferedWriter holds bytes in your own RAM
      |            (nothing has crossed the wall yet)
      |   fflush() or buffer full or newline (on a terminal)
      v   ---------------- write() syscall ----------------  privilege wall
  KERNEL BUFFER    page cache (files), tty buffer (terminals),
      |            socket send buffer (network)
      |   kernel decides when; fsync() forces it for files
      v
  DEVICE           SSD / screen / NIC (which may itself cache!)
```

Each arrow is a place where data can sit around waiting. Each layer has its own “flush” action:

| Call | Moves data from | To |
| --- | --- | --- |
| `fflush(f)` | user buffer | kernel (via `write()`) |
| `fsync(fd)` | kernel page cache | the storage device (waits until it reports done) |
| (the device’s own policy) | device write cache | permanent media |

### 11.3 The big surprise: `write()` returning does **not** mean “on disk”

When you `write()` to a regular file, the kernel copies your bytes into the **page cache**, marks those pages **dirty**, and returns immediately. A background mechanism (writeback) flushes dirty pages to disk later, typically within seconds. So if the power fails right after `write()` returns, you can lose data you thought you saved.

If you need durability (databases, editors saving your work), you call `fsync()` (or open the file with flags like `O_SYNC`), and you accept the cost of waiting for the slow hardware. Real storage devices can add yet another layer of volatile caching inside the drive, which is why serious databases care a great deal about the exact guarantees their OS and hardware provide.

The pattern: **speed comes from buffering; safety comes from flushing.** Every buffer is a tradeoff between the two.

### 11.4 Why output shows up late (or out of order)

Three classic “mysteries,” all explained by buffer modes (Chapter 6.5):

1. **A prompt without a newline doesn’t appear.** `printf("Enter name: ");` sits in a line-buffered stdout waiting for `\n`. Add `fflush(stdout)` (C++ `std::flush`, Python `print(..., flush=True)`).
2. **Output vanishes when the program crashes.** If your program dies from a segfault or `abort()`, the stdio buffer in your process memory is gone: nothing flushes it. The last lines you printed are the ones you most wanted to see. (stderr is unbuffered precisely so error messages survive.)
3. **stdout and stderr appear in the wrong order.** On a terminal both are effectively immediate (stdout is line-buffered). Redirect to a file or pipe and stdout becomes **fully buffered**, while stderr stays unbuffered.

### 11.5 Try it: reorder your own output

```c
#include <stdio.h>

int main(void) {
    printf("one\n");
    fprintf(stderr, "two\n");
    printf("three\n");
    fprintf(stderr, "four\n");
    return 0;
}
```

```
$ ./a.out                  # terminal: one two three four
$ ./a.out > out.txt 2>&1   # redirected: cat out.txt shows two four one three
```

Same program, same code, different order: the only thing that changed is *what stdout is connected to*, which changed its buffering mode. Now try the crash version:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    printf("important last words");   /* no newline, no flush */
    abort();                         /* dies without flushing stdio */
}
```

Run it and the text is lost. Add `fflush(stdout);` before `abort();` and it survives.

---

## Chapter 12. Network Streams: Packets, TCP, and the Biggest Illusion of All

### 12.1 The Internet does not do streams

Here’s the plot twist of the whole book. The network has **no concept** of a stream. It moves discrete, independent **packets** that can be **lost, duplicated, delayed, or arrive out of order**. Each packet travels on its own and is only a few hundred to \~1,500 bytes of payload.

So how can a program call `recv()` and get a smooth, reliable, in-order tape of bytes? Because the **kernel’s TCP implementation** builds that illusion, using buffers on both ends.

### 12.2 The receive path

```
  REMOTE SERVER                                   YOUR MACHINE

  [packet 1] ---\                      NIC --DMA--> RX ring --> interrupt/NAPI
  [packet 3] ----+--> the Internet -->                              |
  [packet 2] ---/    (reordered!)                                   v
                                           Kernel TCP/IP stack:
                                            - check checksums, drop duplicates
                                            - use SEQUENCE NUMBERS to put
                                              segments back in order
                                            - strip IP/TCP headers
                                            - ACK what arrived; wait for gaps
                                                                    |
                                                                    v
                                           SOCKET RECEIVE BUFFER (kernel RAM)
                                           [ ...ordered, gap-free bytes... ]
                                                                    |
                                                      recv() syscall copies out
                                                                    v
                                                          YOUR USER BUFFER
```

Every TCP byte has a **sequence number**. If packet 3 arrives before packet 2, TCP holds it aside until 2 shows up, and only then makes the bytes available in order. If a packet is lost, the sender notices the missing acknowledgment and **retransmits**. Your program sees none of this; at worst it sees `recv()` taking a bit longer. The in-order, complete bytes appear in the socket receive buffer, a ring-like queue that your `recv()` drains.

### 12.3 The send path

Sending is the same mirror image as Chapter 11:

- `send()` copies your bytes into the **socket send buffer** in the kernel and (usually) returns right away. It does **not** mean the data has reached the other machine, or even left your NIC.
- The kernel splits the data into segments, sends them, and **keeps a copy** in the send buffer until the peer acknowledges them (in case retransmission is needed).
- If the send buffer is full (the peer is slow, or the network is congested), `send()` **blocks**. Backpressure, again.

### 12.4 Streams have no message boundaries

This is the consequence everyone trips over. TCP gives you a **byte stream**, not messages. If you call `send("hello")` then `send("world")`, the receiver might get `helloworld` in one `recv()`, or `hel` and then `loworld`, or any other split. TCP promises *order and completeness*, not *boundaries*.

See it with a tiny Python experiment on your own machine (loopback needs no real network):

```python
import socket, threading, time

srv = socket.socket()
srv.bind(("127.0.0.1", 0))
srv.listen(1)
port = srv.getsockname()[1]

def server():
    conn, _ = srv.accept()
    time.sleep(0.5)               # let both sends arrive first
    print("recv:", conn.recv(100))
    conn.close()

threading.Thread(target=server).start()

c = socket.create_connection(("127.0.0.1", port))
c.sendall(b"hello")
c.sendall(b"world")
c.close()
```

You’ll almost always see `recv: b'helloworld'`. Two sends became one read, because the bytes just sat in the receive buffer together. The boundary you created on the sending side didn’t survive the trip.

The practical rules:

- **`recv()`/`read()` may return fewer bytes than you asked for.** Always loop until you have what you need.
- **`send()`/`write()` may accept fewer bytes than you offered**, especially on non-blocking sockets. Check the return value and loop.
- If your protocol has messages, **you must add your own framing**: a length prefix (“the next 128 bytes are one message”), a delimiter (like `\r\n` in HTTP headers), or fixed-size records.

A classic helper:

```c
#include <errno.h>
#include <sys/socket.h>
#include <sys/types.h>

/* Read exactly len bytes, or fewer if the peer closes. Returns bytes read, or -1 on error. */
ssize_t recv_all(int fd, void *buf, size_t len) {
    size_t got = 0;
    while (got < len) {
        ssize_t n = recv(fd, (char *)buf + got, len - got, 0);
        if (n == 0) break;                      /* peer closed the connection */
        if (n < 0) {
            if (errno == EINTR) continue;       /* interrupted by a signal: retry */
            return -1;
        }
        got += (size_t)n;
    }
    return (ssize_t)got;
}
```

### 12.4b Peek at your socket buffers

On Linux you can look at the default and maximum buffer sizes the kernel will use for TCP:

```
$ sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

Each prints three numbers: *minimum, default, maximum* bytes. The kernel can grow a socket’s buffer automatically, up to the maximum, on fast or long-distance connections. (Programs can also request sizes with `setsockopt` and `SO_RCVBUF` / `SO_SNDBUF`.) Those sizes directly limit how fast a single TCP connection can go, which we’ll see in Chapter 14.

### 12.5 The three streams side by side

|  | **Input (keyboard/file)** | **Output (screen/file)** | **Network (TCP)** |
| --- | --- | --- | --- |
| Producer | Hardware (key, disk) | Your program | Remote peer’s kernel, via the wire |
| Chunk size at the hardware | 8-byte report / 4 KB page | Varies (tty, 4 KB pages) | \~1,500-byte packets |
| Kernel buffer | tty line buffer / page cache | tty buffer / page cache | Socket receive & send buffers |
| User buffer | stdio / cin / BufferedReader | stdio / cout / BufferedWriter | Whatever you `recv()` into |
| How the kernel is notified | Interrupt (after DMA) | Interrupt on completion | Interrupt/NAPI after DMA |
| Extra illusion | Read-ahead; cooked lines | Delayed write-back | Reordering, retransmit, no boundaries |
| “Pressure” source | Empty buffer = block | Full buffer = block | Receive window = 0 tells sender to stop |

Same skeleton in all three columns: **hardware, DMA, kernel buffer, syscall, user buffer, your code.** Only the producer changes.

### 12.6 Recap

- **Input streams** are mostly waiting. A keystroke passes at least three buffers, and files get help from the page cache and read-ahead.
- **Output streams** are layered, and every layer has its own flush. `write()` returning is *not* “saved”; `fflush` is not `fsync`.
- **Network streams** are TCP’s illusion: the kernel reorders, deduplicates, and retransmits packets, then presents an ordered byte stream, but **without message boundaries**.

---

*End of Part IV. Next up: Part V, “Pressure, Waiting, and Scale,” where we look at blocking vs non-blocking I/O, how backpressure travels all the way across the Internet, and how one server watches 100,000 sockets with `epoll`.*

---

*Draft 5 (Part V: Chapters 13–15)*

# Part V: Pressure, Waiting, and Scale

You now know what a stream *is*. This part is about what it *feels like* when the buffers fill up and empty out: how your program waits, how a slow consumer pushes back on a fast producer, and how one program keeps an eye on a hundred thousand streams at once.

---

## Chapter 13. Blocking vs Non-Blocking I/O

### 13.1 Blocking: the default

By default, I/O calls are **blocking**. If you call `read()` and the buffer is empty, the kernel puts your thread to sleep (Chapter 7.4) and wakes it when data shows up. If you call `write()` or `send()` and the buffer is full, same thing: you sleep until there’s room.

Blocking is wonderful for the programmer: the code reads top to bottom like a story.

```c
int n = recv(fd, buf, sizeof buf, 0);   /* sleeps until data arrives */
handle(buf, n);
```

The catch: **a blocked thread can do nothing else.** If your server has one thread and it’s blocked waiting on Alice’s socket, Bob’s data sits unread in *his* kernel buffer.

### 13.2 The obvious fix, and why it doesn’t scale

One fix: **a thread (or process) per connection.** Each one blocks happily on its own socket. This works great for tens or hundreds of connections. For tens of thousands it hurts:

- Each thread needs a **stack** (often megabytes of *reserved* address space) plus kernel bookkeeping.
- The scheduler has to juggle all of them, and each switch between threads costs time and disturbs CPU caches.
- Most connections are idle most of the time (remember: a stream is mostly silence), so you’re paying for thousands of threads that are just sleeping.

This problem was famous enough in the late 1990s to get its own nickname: **the C10K problem** (“how do you handle ten thousand concurrent connections?”).

### 13.3 Non-blocking mode

The other fix: tell the kernel **“never put me to sleep for this descriptor.”** You set the `O_NONBLOCK` flag. Now:

- `read()` on an empty buffer returns `-1` immediately with `errno == EAGAIN` (or `EWOULDBLOCK`, same thing on Linux): “nothing here right now, try later.”
- `write()` on a full buffer returns `-1` with `EAGAIN`, or accepts only *part* of your data and returns a short count.

```c
#include <fcntl.h>

static int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

But now you have a new problem. If you simply loop around asking every socket “anything yet?”, you’ve reinvented **busy polling** (Chapter 9.4), the thing that pegged your CPU at 100%. What you want is: *sleep until at least one of my many descriptors is ready, and tell me which.* That’s **I/O multiplexing**, the subject of Chapter 15.

### 13.4 Three styles of waiting

| Style | How it works | Strength | Cost |
| --- | --- | --- | --- |
| **Blocking, thread per connection** | Each thread sleeps in its own `read()` | Simple, linear code | Memory and scheduling overhead at scale |
| **Non-blocking + readiness events** | One thread asks “which descriptors are *ready*?” (`epoll`, `kqueue`), then does the read/write | Scales to huge connection counts | You must write state machines instead of linear code |
| **Asynchronous completion** | You submit “read into this buffer” and are told when it’s *finished* (Linux `io_uring`, Windows IOCP) | Fewer syscalls; true async I/O | Newest, most complex APIs |

The distinction between the middle and last rows is subtle but worth remembering: **readiness** (“you can read now without blocking, go ahead”) vs **completion** (“I already did the read; here’s your data”).

### 13.5 A gotcha: regular files are always “ready”

Non-blocking mode and readiness notification are designed for **sockets, pipes, and terminals**, where data arrival is unpredictable. A **regular file** on disk is treated as always ready: the kernel will just block you while it fetches the data, regardless of `O_NONBLOCK`, and `epoll` can’t usefully watch it. That’s one reason asynchronous *file* I/O needed separate solutions (thread pools, `io_uring`) while sockets got `epoll`.

### 13.6 Try it

Go back to `poll_vs_block.c` from Chapter 9.4. The “busy” mode *is* a non-blocking read in a loop. Now you know why servers don’t do that, and what they do instead (Chapter 15).

---

## Chapter 14. Backpressure and Flow Control

### 14.1 What backpressure is

Backpressure is what happens when a **slow consumer** pushes back on a **fast producer**. The word comes from plumbing: if the pipe downstream is clogged, pressure builds up and slows the flow upstream.

It’s the same full-buffer rule from Chapter 2.4, but now you’ll see how far it propagates.

> **A bounded buffer turns “consumer is slow” into “producer waits.”** That’s the entire idea.

The alternative, an **unbounded** buffer, doesn’t remove the problem. It converts a slowdown into running out of memory.

### 14.2 Backpressure in a local pipe

When you type `producer | consumer` in a shell, the kernel creates a **pipe**: a ring buffer in kernel memory (64 KiB by default on Linux). If `producer` writes faster than `consumer` reads, the pipe fills and the kernel blocks `producer`’s `write()` until `consumer` drains some. The producer is automatically slowed to the consumer’s speed. Nobody wrote any code to make that happen.

Let’s see the pressure directly, with a non-blocking write end so we find out about it instead of sleeping:

```c
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    int fds[2];
    if (pipe(fds) < 0) { perror("pipe"); return 1; }
    fcntl(fds[0], F_SETFL, fcntl(fds[0], F_GETFL) | O_NONBLOCK);
    fcntl(fds[1], F_SETFL, fcntl(fds[1], F_GETFL) | O_NONBLOCK);

    char chunk[1024] = {0};
    long total = 0;
    for (;;) {                                   /* nobody is reading yet */
        ssize_t n = write(fds[1], chunk, sizeof chunk);
        if (n < 0) {
            if (errno == EAGAIN) break;          /* the pipe is full: PRESSURE */
            perror("write"); return 1;
        }
        total += n;
    }
    printf("pipe is full after %ld bytes\n", total);

    char buf[4096];
    while (read(fds[0], buf, sizeof buf) > 0) ;  /* consumer drains it */

    ssize_t n = write(fds[1], chunk, sizeof chunk);
    printf("after draining, write accepted %zd bytes\n", n);
    return 0;
}
```

On a typical Linux system the first line prints `65536`, the default pipe capacity. That number is your buffer size, felt rather than read from a manual. Remove `O_NONBLOCK` from the write end (and don’t drain) and the program would simply hang on the full pipe: backpressure with the sleeping turned on.

### 14.3 Backpressure across the Internet

Now the magic part. Two programs on different continents. The sender is fast; the receiver’s application is slow at calling `recv()`. Follow the pressure backward:

```
  SLOW APP stops calling recv()
        |
        v
  Receiver's socket RECEIVE buffer fills up
        |
        v
  TCP advertises a shrinking RECEIVE WINDOW in its ACKs
  ("I only have N bytes of room left")
        |
        v   (travels across the Internet in ACK packets)
  Sender's TCP may only have that much unacknowledged data in flight
        |
        v
  Sender's socket SEND buffer fills up
        |
        v
  Sender's send() blocks (or returns EAGAIN)
        |
        v
  FAST APP is slowed down
```

This is **TCP flow control**. Every TCP acknowledgment carries a **window size**: how many more bytes the receiver is willing to accept right now. When the receive buffer is completely full, the window becomes **0**, and the sender must stop sending data (it periodically sends tiny probes to learn when the window reopens). As soon as your app drains the buffer, the receiver announces a bigger window and data flows again.

So a slow program in one city really does slow down a fast program in another, through the same “full buffer means wait” rule we met in a toy ring of 8 bytes.

### 14.4 Two different pressures: flow control vs congestion control

TCP actually limits the sender with **two** independent windows, and the smaller one wins:

|  | **Flow control** | **Congestion control** |
| --- | --- | --- |
| Protects | The **receiver’s buffer** | The **network** in between |
| Controlled by | Receive window (`rwnd`), advertised by the receiver | Congestion window (`cwnd`), estimated by the sender from loss and delay |
| Signal | “My buffer has this much room.” | Packet loss or rising delay, interpreted as “the network is busy” |

When you notice that a download starts slowly and then speeds up, that’s congestion control probing the network’s capacity.

### 14.5 Why buffer size limits speed: the window and the round trip

A sender can have at most one **window** of unacknowledged data in flight. Acknowledgments take one **round-trip time (RTT)** to come back. So the maximum throughput of one TCP connection is roughly:

> **throughput ≤ window size / RTT**

Example: with a 64 KB window and a 100 ms RTT (say, a trans-oceanic link), you can’t exceed 65,536 bytes / 0.1 s ≈ 655 KB/s, or about 5 megabits per second, no matter how fat the pipe is. The sender sends a window’s worth, then *waits* for acknowledgments. To go faster on long, fast links you need much bigger windows. (TCP’s “window scaling” option and Linux’s auto-tuned buffers, the `tcp_rmem`/`tcp_wmem` limits from Chapter 12.4b, exist for exactly this.) Buffer size isn’t just about absorbing bursts; it directly sets your speed limit over distance.

### 14.6 Backpressure in application code

The kernel handles the pipe and TCP layers for you. Once data is inside **your** program, you’re responsible. The classic bug:

```
  fast reader  --->  [ unbounded in-memory queue ]  --->  slow processor
```

The reader happily keeps reading from the socket (so TCP never pushes back), the queue grows without limit, and eventually the process runs out of memory. The cure is the same as everywhere else in this book: **bound the buffer, and make the producer wait or slow down when it’s full.** Frameworks give this idea different names:

- Bounded queues and bounded channels (a full queue blocks or rejects `put`).
- **Node.js streams:** `write()` returns `false` when the internal buffer exceeds its high-water mark, and you wait for the `drain` event.
- **Reactive Streams** (Java and others): the consumer explicitly requests *n* more items.
- In an `epoll`-style server: when a client can’t keep up, stop reading *from the source*, or stop reading from that client, so the kernel’s buffers (and TCP) do the pushing back for you.

If you remember one sentence from this chapter: **every queue in a streaming system should have a limit, and someone must decide what happens when it’s hit.**

---

## Chapter 15. `select`, `poll`, `epoll`: Watching 100,000 Pipes at Once

### 15.1 The question

You have 100,000 non-blocking sockets. Most are idle. You want to sleep until *any* of them has data, wake instantly, and find out *which ones*. That’s **I/O multiplexing**, and Unix has three generations of tools for it.

### 15.2 Generation 1: `select`

```c
fd_set readable;
FD_ZERO(&readable);
FD_SET(fd_a, &readable);
FD_SET(fd_b, &readable);
select(max_fd + 1, &readable, NULL, NULL, NULL);   /* sleeps until one is ready */
if (FD_ISSET(fd_a, &readable)) { /* read from a */ }
```

It works, and it’s portable, but:

- `fd_set` is a fixed-size bitmask: classically limited to **1024** descriptors (`FD_SETSIZE`).
- You pass the **whole set** into the kernel on every call, and the kernel hands back a modified copy, so you must rebuild it each time.
- Both you and the kernel **scan every descriptor** to find the ready ones: cost grows with the *total* number watched.

### 15.3 Generation 2: `poll`

`poll` takes an array of `struct pollfd { int fd; short events; short revents; }`, so there’s no 1024 limit and you don’t destroy your input. But it’s still **O(n)**: every call, the kernel walks the entire array, and you scan it again to find the flagged entries. With 100,000 mostly idle sockets, you pay to check 100,000 each time even if only 3 have data.

### 15.4 Generation 3: `epoll` (Linux)

`epoll` fixes the real problem, which is that the kernel forgets your interest list between calls. With `epoll`, you **register your interest once**, the kernel remembers it, and `epoll_wait` returns **only the ready descriptors**.

Three calls:

1. `epoll_create1(0)`: create an epoll instance (a kernel object, itself referred to by a file descriptor).
2. `epoll_ctl(ep, EPOLL_CTL_ADD / MOD / DEL, fd, &event)`: add, change, or remove a descriptor you care about.
3. `epoll_wait(ep, events, max, timeout)`: sleep until something is ready; fill `events[]` with just the ready ones.

**How does the kernel do this efficiently?** Remember wait queues from Chapter 7.4? When you add a socket to an epoll instance, the kernel attaches a small **callback** to that socket’s wait queue. When data arrives (interrupt, then bottom half, then the socket receive buffer grows), the callback fires and drops the socket onto the epoll instance’s **ready list**. `epoll_wait` just hands back that list. Work is proportional to the number of **ready events**, not to the number of watched sockets.

```
  select/poll:   every call:  [ check fd 1, fd 2, fd 3, ... fd 100000 ]  -> O(total)

  epoll:         once:        register all 100,000 with the kernel
                 on data:     socket callback --> READY LIST
                 every call:  epoll_wait returns just the ready list      -> O(ready)
```

|  | `select` | `poll` | `epoll` |
| --- | --- | --- | --- |
| Descriptor limit | `FD_SETSIZE` (1024 classically) | None built in | None built in |
| Interest list kept by | You (rebuilt each call) | You (array each call) | The kernel |
| Cost per call | O(total) | O(total) | O(ready) |
| Portability | Everywhere | Nearly everywhere | Linux only |

Other platforms have their own equivalents: **`kqueue`** on BSD and macOS, **IOCP** on Windows (completion-based), and Linux’s newer **`io_uring`** (also completion-based).

### 15.5 Level-triggered vs edge-triggered

`epoll` has two modes:

- **Level-triggered (default):** “this socket **is** readable.” If you don’t drain it, `epoll_wait` reports it again next time. Forgiving.
- **Edge-triggered (`EPOLLET`):** “this socket **just became** readable.” You are told once per new arrival of data, so you **must read until `EAGAIN`** (hence non-blocking sockets), or you may never be woken for the leftover bytes. Fewer wakeups, easier to get wrong.

Start with level-triggered. It’s correct in more situations.

### 15.6 A complete echo server

Here’s a small single-threaded `epoll` server that echoes whatever each client sends. One thread, any number of clients.

```c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <stdio.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#define MAX_EVENTS 64

static void set_nonblock(int fd) {
    fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
}

int main(void) {
    int lsock = socket(AF_INET, SOCK_STREAM, 0);
    int yes = 1;
    setsockopt(lsock, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port = htons(9000),
        .sin_addr.s_addr = htonl(INADDR_ANY),
    };
    if (bind(lsock, (struct sockaddr *)&addr, sizeof addr) < 0) {
        perror("bind"); return 1;
    }
    listen(lsock, 128);
    set_nonblock(lsock);

    int ep = epoll_create1(0);
    struct epoll_event ev = { .events = EPOLLIN, .data.fd = lsock };
    epoll_ctl(ep, EPOLL_CTL_ADD, lsock, &ev);

    struct epoll_event events[MAX_EVENTS];
    for (;;) {
        int n = epoll_wait(ep, events, MAX_EVENTS, -1);   /* sleeps until ready */
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == lsock) {                            /* new connections */
                int c;
                while ((c = accept(lsock, NULL, NULL)) >= 0) {
                    set_nonblock(c);
                    struct epoll_event cev = { .events = EPOLLIN, .data.fd = c };
                    epoll_ctl(ep, EPOLL_CTL_ADD, c, &cev);
                }
            } else {                                      /* data from a client */
                char buf[4096];
                ssize_t got = read(fd, buf, sizeof buf);
                if (got > 0) {
                    write(fd, buf, (size_t)got);          /* simplified: see note below */
                } else if (got == 0 || (errno != EAGAIN && errno != EINTR)) {
                    close(fd);                            /* close also removes it from epoll */
                }
            }
        }
    }
}
```

Compile and run it, then in a few other terminals connect with `nc localhost 9000` and type. Every client is served by the **same single thread**, and while nobody’s talking, the process sleeps inside `epoll_wait` using essentially no CPU.

**What this simplified server skips** (and a real one can’t):

- `write()` may accept only *part* of the data when the client is slow. A real server keeps a **per-connection output buffer**, registers for `EPOLLOUT` to learn when the socket can take more, and stops reading from a client whose output buffer is too full. That’s Chapter 14’s backpressure, implemented by hand.
- Error handling, timeouts, and clean shutdown.

### 15.7 The whole picture, once more

Look at what a single `epoll_wait` call quietly depends on: NIC DMA into a ring (Chapter 8), an interrupt and NAPI poll (Chapters 7 and 9), TCP reordering into the socket receive buffer (Chapter 12), a wait-queue callback that marks the socket ready (this chapter), and a syscall across the privilege wall (Chapter 5). Every chapter so far shows up in one function call.

### 15.8 Recap

- **Blocking** I/O sleeps the thread; **non-blocking** I/O returns `EAGAIN` and leaves you to decide what to do.
- Threads-per-connection stops scaling; **multiplexing** lets one thread watch many descriptors.
- **Backpressure** is just “full bounded buffer means producer waits,” and it propagates from your app, through TCP’s receive window, to the remote sender.
- **`select`/`poll`** scan everything every call; **`epoll`** keeps the interest list in the kernel and returns only what’s ready.

---

*End of Part V. Next up: Part VI, “Going Fast,” where we break the double-buffering rule from Chapter 6 with `mmap` and zero-copy `sendfile`, then close the book with the IOMMU, cache coherency, and the other gremlins that live at the hardware boundary.*

---

*Draft 6 (Part VI: Chapters 16–18)*

# Part VI: Going Fast

In Chapter 6 we admitted that double buffering has a price: data gets **copied** from the kernel buffer into your buffer, and moving bytes costs CPU time and memory bandwidth. This part is about how the fast systems cheat: by mapping memory instead of copying it, by never letting the data enter user space at all, and by dealing with the odd gremlins that live where software meets hardware.

---

## Chapter 16. Memory-Mapped Files (`mmap`)

### 16.1 A quick refresher on virtual memory

Your program doesn’t see physical RAM. It sees **virtual addresses**, and the CPU’s memory management unit translates them to physical addresses using **page tables** maintained by the kernel. Memory is divided into **pages** (4 KB on most systems). A virtual page may map to a physical page, or to nothing yet.

When your code touches a virtual page that isn’t mapped, the CPU raises a **page fault** and the kernel steps in: maybe it allocates a page, maybe it loads data from a file, maybe it kills you with a segmentation fault. Page faults are the hook that makes `mmap` work.

### 16.2 The idea

With `read()`, the kernel *copies* bytes from the page cache into your buffer. With `mmap()`, you say: **“make this file appear in my address space.”** The kernel doesn’t copy anything up front. It just sets up the mapping. When you touch a page of it:

1. Page fault (the page isn’t mapped yet).
2. The kernel finds the file’s page in the **page cache**, reading it from disk (via DMA, Chapters 8 and 10.3) if it isn’t there yet.
3. The kernel maps **that very page-cache page** into your address space.
4. Your instruction resumes, and from now on that memory access is a plain load, with no syscall and no copy.

```
  read():   [ SSD ] --DMA--> [ page cache ] --CPU copy--> [ your buffer ]

  mmap():   [ SSD ] --DMA--> [ page cache ] <---- your page table points HERE
                                   ^
                                   |  you read it directly, no copy
                              your code: p[i]
```

In terms of Chapter 6, `mmap` **removes the second buffer** (and the second copy). Your “user buffer” *is* the kernel’s page cache, shared with you.

### 16.3 Count lines with `mmap`

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main(int argc, char **argv) {
    if (argc < 2) { fprintf(stderr, "usage: %s file\n", argv[0]); return 1; }

    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    struct stat st;
    if (fstat(fd, &st) < 0) { perror("fstat"); return 1; }
    if (st.st_size == 0) { printf("0 lines\n"); return 0; }   /* can't map 0 bytes */

    char *p = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (p == MAP_FAILED) { perror("mmap"); return 1; }

    long lines = 0;
    for (off_t i = 0; i < st.st_size; i++)    /* just array indexing! */
        if (p[i] == '\n') lines++;

    printf("%ld lines\n", lines);
    munmap(p, st.st_size);
    close(fd);
    return 0;
}
```

There’s no `read()` anywhere. The file is just an array named `p`. Run it with `strace -c` and you’ll see a single `mmap` call and no bulk data `read` calls at all. To see the page faults that did the real work:

```
$ /usr/bin/time -v ./mmapcount big.bin 2>&1 | grep -i "page faults"
```

You’ll see **minor** faults (the page was already in the page cache, so the kernel only had to map it) and, on a cold file, **major** faults (the kernel had to go to disk). Faults aren’t free, so the kernel often maps several pages per fault, but they are far cheaper than a syscall per chunk.

### 16.4 Writing through a mapping

With `MAP_SHARED` and `PROT_WRITE`, writing to the array modifies the page-cache page directly, and the kernel eventually writes it back to the file. Exactly like Chapter 11.3, “it’s in memory” does **not** mean “it’s on disk.” The equivalent of `fsync` for a mapping is `msync()`.

### 16.5 When `mmap` is great, and when it isn’t

**Great for:**

- Large files accessed **randomly** (databases, indexes): treat the file like a giant array.
- Files **shared between processes** (each process maps the same page-cache pages: one copy in RAM, many users).
- Avoiding a copy when you only need to look at part of a big file.

**Not great for:**

- **Sockets and pipes.** There’s nothing to map: a stream has no fixed contents, only a window of arriving bytes.
- **Small, strictly sequential reads.** A well-buffered `read()` with kernel read-ahead can be just as fast, and the page-fault machinery can even be slower.
- **Error handling.** If the file is truncated by another process while you’re reading it, touching the vanished page raises `SIGBUS`, not a tidy `-1` from `read()`. I/O errors from the disk also arrive as signals.
- **Predictable latency.** Any memory access can secretly turn into a disk read (a major fault), and you can’t see it in your code.

The lesson isn’t “always `mmap`.” It’s that **a copy is a cost you can sometimes avoid, at the price of different failure modes.**

---

## Chapter 17. Zero-Copy and `sendfile`

### 17.1 The classic web-server problem

A web server’s most common job: take a **file on disk** and send it out a **socket**. The obvious code:

```c
while ((n = read(file_fd, buf, sizeof buf)) > 0)
    write(sock_fd, buf, n);
```

Let’s count what actually happens for each chunk:

```
  1. read():  SSD --DMA-->  page cache (kernel)
  2. read():  page cache --CPU copy-->  your buffer          (kernel -> user)
  3. write(): your buffer --CPU copy-->  socket send buffer   (user -> kernel)
  4.          socket send buffer --DMA-->  NIC

  Two DMA transfers (fine: the CPU isn't involved)
  Two CPU copies, and two syscalls (four mode switches)
```

The data goes up into user space and straight back down again, and your program **never even looks at it**. It’s a courier carrying a sealed package across a bridge and back.

### 17.2 `sendfile`: skip the trip

Linux’s `sendfile()` (and equivalents on other OSes) says to the kernel: **“send this file to that socket, and don’t bother me with the bytes.”**

```c
#include <sys/sendfile.h>

ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
```

One syscall, and the data never crosses into user space:

```
  1. SSD --DMA--> page cache
  2. page cache --> socket send buffer     (a kernel-internal copy, or just
                                            descriptors, see below)
  3. socket send buffer --DMA--> NIC
```

And on network cards that support **scatter-gather DMA**, step 2 can shrink even further: instead of copying the bytes, the kernel just puts **descriptors pointing at the page-cache pages** into the NIC’s ring (Chapter 8.4), and the NIC gathers the data straight from the page cache. Then the CPU copies **no payload bytes at all**, and that’s what “zero-copy” literally means.

A minimal send loop:

```c
#include <errno.h>
#include <stdio.h>
#include <sys/sendfile.h>
#include <sys/stat.h>

/* Send an entire open file to a connected socket. Returns 0 on success. */
int send_whole_file(int sock_fd, int file_fd) {
    struct stat st;
    if (fstat(file_fd, &st) < 0) return -1;

    off_t offset = 0;
    while (offset < st.st_size) {
        ssize_t sent = sendfile(sock_fd, file_fd, &offset, st.st_size - offset);
        if (sent < 0) {
            if (errno == EINTR) continue;     /* signal: retry */
            perror("sendfile");
            return -1;
        }
        if (sent == 0) break;                 /* file shrank under us */
    }
    return 0;
}
```

Notice `sendfile` can send *less* than requested (it follows the same “short write” rules as `write`), and it advances `offset` for you, so a loop is mandatory. Backpressure still applies: on a non-blocking socket you’d get `EAGAIN` when the send buffer is full (Chapter 14).

### 17.3 Scorecard

| Method | Syscalls per chunk | CPU copies of the payload |
| --- | --- | --- |
| `read()` + `write()` | 2 | 2 |
| `mmap()` + `write()` | about 1 (after setup) | 1 |
| `sendfile()` | 1 | 1 (kernel-internal), or **0** with scatter-gather DMA |

(Exact numbers depend on the kernel version, the file system, and the NIC. Treat the table as the *shape* of the improvement.)

### 17.4 Where zero-copy is used, and where it can’t be

Systems that serve large amounts of unmodified data love it: static-file web servers, and **Kafka**, which serves stored log segments to consumers using `sendfile`. Other relatives of the same idea:

- **`splice()`** moves data between a pipe and another descriptor inside the kernel.
- **`MSG_ZEROCOPY`** lets `send()` use your pages directly instead of copying them (with caveats: you must not reuse the buffer until completion is signaled).
- **`O_DIRECT`** bypasses the page cache altogether so data DMAs directly between the device and *your* buffer. Databases that run their own caching use it.
- **Kernel-bypass networking** (DPDK, RDMA) lets user programs talk to the NIC’s rings directly.

**The limit:** zero-copy only works when you don’t need to *look at* or *change* the data. If you must compress, parse, or encrypt it in user space (for example, classic TLS, where your library encrypts bytes before sending), the data has to come up into user space anyway. (Kernel TLS exists precisely to push that work back into the kernel so `sendfile` can work again.) Every optimization in this chapter is the same bargain: **fewer copies, in exchange for less flexibility.**

---

## Chapter 18. IOMMU, Cache Coherency, and Other Gremlins

This last chapter is a field guide to the oddities that live at the border between software and hardware. You can write plenty of correct programs without knowing them, but they explain a lot of “wait, why does it do *that*?” moments.

### 18.1 The IOMMU, again

In Chapter 8.5 we met the **IOMMU**: an MMU for devices. It does three jobs:

- **Protection:** a device can only DMA into memory ranges the OS explicitly mapped for it. A buggy or malicious device can’t scribble over the kernel.
- **Translation:** the device sees “I/O virtual addresses,” which the IOMMU maps to physical RAM. Scattered physical pages can look contiguous to the device.
- **Virtualization:** a virtual machine can be given direct access to a physical device, because the IOMMU confines that device’s DMA to the VM’s memory.

A related annoyance is the **bounce buffer**: if a device can only address part of memory (say, an old device limited to 32-bit addresses) and the buffer lives somewhere it can’t reach, the kernel quietly DMAs into a reachable “bounce” buffer and then copies the data. It works, but it costs you a copy you didn’t ask for.

### 18.2 Cache coherency and “who wrote this last?”

Remember that DMA writes into RAM behind the CPU’s back. Two details make that safe:

- **Cache coherency.** On x86, DMA traffic is kept coherent with CPU caches by hardware, so the CPU sees fresh data. On many other architectures (including numerous ARM systems), DMA can be *non-coherent*, and the driver must explicitly **invalidate or flush caches** around each transfer. Linux wraps this in a “DMA mapping” API so drivers say “I’m handing this buffer to the device” and “I’m taking it back,” and the kernel does whatever the hardware needs.
- **Memory ordering.** Descriptor rings (Chapter 8.4) are shared between the CPU and a device that reads them independently. A driver must fill in *all* the fields of a descriptor, issue a **memory barrier**, and only *then* flip the “device owns this” flag or ring the device’s doorbell register. Otherwise the device might see a half-written descriptor. These are the ring-buffer cursors from Chapter 2, now with a physical-hardware reason to care about the order of writes.

### 18.3 Many queues, many cores, many memory banks

A single RX ring served by a single CPU core becomes a bottleneck on fast links. Modern NICs therefore have **multiple RX/TX queues**, and use a hash of each packet’s connection (addresses and ports) to steer a given connection’s packets to the same queue. This is called **RSS** (Receive Side Scaling). Each queue has its own interrupt, which the OS can pin to a particular core (**IRQ affinity**). Benefits: parallelism, and packets of one connection are processed on one core, keeping that core’s caches warm.

On multi-socket servers there’s one more wrinkle, **NUMA** (non-uniform memory access): RAM is physically attached to specific CPU sockets, and accessing “far” memory is slower. High-performance setups try to keep a NIC’s buffers, its interrupts, and the processing thread all on the same NUMA node.

### 18.4 Bufferbloat: when buffers are *too* big

Throughout this book, buffers have been the heroes. Here’s the plot twist: **an oversized buffer can make things worse.**

Imagine a router with a huge buffer on a slow link. Senders keep pushing data. The buffer fills, so every packet waits behind a long queue (seconds, in bad cases), and the sender doesn’t learn the link is saturated, because packets aren’t being *dropped*. TCP’s congestion control (Chapter 14.4) uses loss and delay as its signals, and a giant buffer hides the loss while exploding the delay. Result: your video call lags whenever someone uploads a file. This is called **bufferbloat**.

The cure isn’t “no buffers.” It’s **right-sized, actively managed** queues (modern queue disciplines such as fq\_codel), which drop or mark packets early to signal “slow down” while the queue is still short. A buffer absorbs *bursts*; it should never be used to hide a *sustained* overload.

### 18.5 Application-level gremlins

A short list of surprises that bite programmers using the APIs from this book:

- **Short reads and writes.** `read`, `recv`, `write`, `send`, and `sendfile` may transfer *fewer* bytes than you asked for. Always loop. (Chapter 12.4.)
- **`EINTR`.** A signal can interrupt a blocking syscall, which then fails with `EINTR`. Usually you just retry.
- **`EAGAIN` vs end-of-file.** On a non-blocking descriptor, `-1` with `EAGAIN` means “no data *yet*.” A return of `0` from `read`/`recv` means the peer **closed**. Don’t confuse them.
- **`SIGPIPE`.** Writing to a socket or pipe whose other end has closed raises `SIGPIPE`, which kills your process by default. Servers either ignore the signal or pass `MSG_NOSIGNAL` to `send()`, then handle the `EPIPE` error.
- **Small writes and Nagle’s algorithm.** TCP may delay tiny segments to combine them (Nagle’s algorithm), which interacts badly with the receiver’s delayed ACKs and can add tens of milliseconds of latency to chatty request/response protocols. Latency-sensitive programs set the `TCP_NODELAY` option. It’s the network cousin of “flush your stdout.”
- **Forgetting to flush.** Everything in Chapter 11.

### 18.6 Try it: look at your own machine

On Linux, a few commands that reveal the gremlins above (replace `eth0` with your interface; some may not work on virtual machines or without privileges):

```
$ ethtool -l eth0                 # how many RX/TX queues (channels) the NIC has
$ lscpu | grep -i numa            # NUMA nodes on this machine
$ cat /proc/interrupts | grep -i eth0    # per-queue interrupt counters (Chapter 7.6)
$ sysctl net.ipv4.tcp_congestion_control # which congestion algorithm TCP is using
$ tc qdisc show                   # queueing disciplines (is fq_codel active?)
```

---

## Epilogue: The One-Page Mental Model

If you’ve made it this far, here’s the whole book on one page. Tape it above your monitor.

```
  HARDWARE            KERNEL SPACE                       USER SPACE
  (chunky, bursty,                                       (wants smooth bytes)
   slow)

  [ SSD / NIC /  ]    [ ring of descriptors ]
  [ keyboard     ] -> [ kernel buffer:      ] ==wall==>  [ user buffer ] --> your code
    DMA moves chunks    page cache / socket        syscall copy   (stdio, cin,
    into RAM, then      buffer / tty buffer        (read, recv)    BufferedReader)
    raises an INTERRUPT
                        wait queues wake your thread; epoll watches many at once

  Output is the same picture in reverse: you fill, the kernel drains, the device sends.
```

And the ten rules to carry out the door:

1. **A stream is a buffer with two cursors.** The “flow” is an illusion.
2. **Hardware is chunky and slow; the CPU is fast.** Buffers decouple them in time.
3. **Empty buffer means the consumer waits; full buffer means the producer waits.** That’s all of blocking and backpressure.
4. **The CPU enforces a wall** between user space and kernel space. Syscalls are the doors, and crossing is expensive.
5. **Make fewer, bigger syscalls.** That’s why user-space buffers exist.
6. **DMA moves the bytes; interrupts (or polling) announce them.** Under heavy load, systems switch from interrupts to polling.
7. **`write()` returning is not “delivered,” and `fflush` is not `fsync`.** Every layer buffers.
8. **TCP is a byte stream with no message boundaries.** Frame your messages and loop on partial reads and writes.
9. **To serve many streams, don’t use many threads: use readiness events (`epoll`).** And bound every queue.
10. **Copies are a cost.** `mmap`, `sendfile`, and `O_DIRECT` remove them, in exchange for less flexibility.

---

*End of the main text (Parts I–VI). Still to come in the next drafts: the appendices (cheat sheets, a glossary, `strace` recipes, further reading), plus a consistency pass over the whole book.*

---

*Draft 7 (Appendices)*

# Appendices

## Appendix A. Syscall and Function Cheat Sheet

### A.1 The core I/O syscalls (Linux)

| Call | What it does | Remember |
| --- | --- | --- |
| `open(path, flags)` | Get a file descriptor for a file or device | Returns the lowest free fd; `-1` on error |
| `read(fd, buf, n)` | Copy up to `n` bytes from the kernel into `buf` | May return fewer than `n`; `0` means EOF/peer closed |
| `write(fd, buf, n)` | Copy up to `n` bytes from `buf` into the kernel | May write fewer; success is **not** “on disk” |
| `close(fd)` | Release the descriptor | Does not imply `fsync` |
| `fsync(fd)` | Push dirty pages to the storage device | The slow, safe one |
| `fcntl(fd, F_SETFL, O_NONBLOCK)` | Switch a descriptor to non-blocking | Then watch for `EAGAIN` |
| `pipe(fds)` | Create a kernel ring buffer with two ends | 64 KiB default capacity on Linux |
| `mmap(...)` / `munmap(...)` / `msync(...)` | Map a file into your address space / unmap / flush | Page faults do the loading |
| `sendfile(out, in, &off, n)` | Kernel-side file-to-socket copy | Loop on short counts |

### A.2 Sockets

| Call | What it does |
| --- | --- |
| `socket(domain, type, 0)` | Create an endpoint (`SOCK_STREAM` = TCP) |
| `bind`, `listen`, `accept` | Server side: claim a port, mark it as listening, take a new connection |
| `connect` | Client side: start the TCP handshake |
| `send` / `recv` | Like `write` / `read` with socket flags (`MSG_NOSIGNAL`, `MSG_PEEK`, and so on) |
| `setsockopt` | Tune a socket: `SO_RCVBUF`, `SO_SNDBUF`, `TCP_NODELAY`, `SO_REUSEADDR` |
| `shutdown`, `close` | Half-close or fully close |

### A.3 Waiting for many descriptors

| Call | What it does |
| --- | --- |
| `select(nfds, ...)` | Old, limited to `FD_SETSIZE` descriptors, O(total) |
| `poll(fds, n, timeout)` | No size limit, still O(total) |
| `epoll_create1`, `epoll_ctl`, `epoll_wait` | Kernel-held interest list, O(ready) |
| `io_uring_*` | Submission/completion rings shared with the kernel |

### A.4 Library level vs syscall level

| Library (user buffer) | Underlying syscall | Flush relationship |
| --- | --- | --- |
| `fopen`, `fread`, `fgets`, `fgetc` | `open`, `read` | Library refills its buffer with one big `read` |
| `fwrite`, `fprintf`, `printf`, `puts` | `write` | `fflush` triggers the `write` |
| `fflush(f)` | `write` | User buffer to kernel |
| `fsync(fileno(f))` | `fsync` | Kernel to device (call `fflush` first!) |
| `setvbuf` | none | Chooses full, line, or no buffering |

---

## Appendix B. Where the Buffers Live

| Buffer | Lives in | Filled by | Drained by | Typical size / note |
| --- | --- | --- | --- | --- |
| Device on-board FIFO | Hardware | The wire / media | The device’s DMA engine | Small (KB to a few MB), overruns lose data |
| NIC RX/TX descriptor ring | Kernel RAM | Driver and NIC | NIC and driver | Hundreds to thousands of entries (`ethtool -g`) |
| Page cache | Kernel RAM | Disk DMA, `write()` | `read()`, writeback to disk | Uses most “free” RAM, in 4 KB pages |
| Socket receive buffer | Kernel RAM | TCP stack | `recv()` | Auto-tuned (`tcp_rmem`) |
| Socket send buffer | Kernel RAM | `send()` | TCP stack, then NIC | Auto-tuned (`tcp_wmem`) |
| Pipe buffer | Kernel RAM | `write()` | `read()` | 64 KiB default |
| TTY line buffer | Kernel RAM | Keyboard driver | `read()` on Enter (canonical mode) | One line, editable |
| stdio / iostream / BufferedReader buffer | User RAM | `read()` | Your code | About 4–8 KB by default (varies) |
| Your own application queues | User RAM | Your code | Your code | **Bound them!** |

---

## Appendix C. `strace` and Friends: A Recipe Book

The best way to *believe* this book is to watch your own programs. These tools let you eavesdrop on the walls and buffers.

### C.1 `strace` (see every syscall)

```
$ strace ./prog                       # every syscall, with arguments and results
$ strace -c ./prog                    # summary: how many of each syscall, and time spent
$ strace -e trace=read,write ./prog   # only the calls you care about
$ strace -e trace=network ./prog      # only socket-related calls
$ strace -T ./prog                    # show time spent INSIDE each syscall
$ strace -ttt ./prog                  # absolute timestamps on each line
$ strace -f ./prog                    # follow child processes and threads
$ strace -p 1234                      # attach to an already-running process
$ strace -o trace.txt ./prog          # write the trace to a file
```

What to look for:

- **Thousands of tiny `read(…, 1)` or `write(…, 1)` calls?** Missing user-space buffer (Chapters 5–6).
- **`write` calls at odd times?** Flush behavior (Chapter 11).
- **`EAGAIN` in a loop?** A busy-poll (Chapter 9, Chapter 13).
- **`epoll_wait` sleeping most of the time?** Healthy event loop (Chapter 15).
- **`sendfile` or `mmap` instead of read/write pairs?** Zero-copy at work (Chapters 16–17).

Note that `strace` slows the traced program, so use it to see *what* happens, not to measure speed.

### C.2 Looking at sockets and descriptors

```
$ ss -tmi                    # TCP sockets with memory (buffer) and congestion info
$ ss -ltn                    # listening TCP sockets
$ ls -l /proc/<pid>/fd       # every open descriptor of a process
$ cat /proc/<pid>/io         # bytes read/written, syscall counts
$ lsof -p <pid>              # open files and sockets of a process
$ sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem
```

`ss -tmi` is particularly good for this book: it shows each connection’s send and receive queue sizes, so you can *see* backpressure building (a growing receive queue means your application isn’t calling `recv()` fast enough).

### C.3 System-wide views

```
$ cat /proc/interrupts       # interrupt counters per CPU (Chapter 7)
$ cat /proc/softirqs         # bottom-half work, such as NET_RX
$ vmstat 1                   # memory, interrupts (in) and context switches (cs) per second
$ iostat -x 1                # disk throughput and queue depths (from the sysstat package)
$ free -h                    # page cache is in the buff/cache column
$ ethtool -g eth0            # NIC ring sizes (Chapter 8)
$ ethtool -c eth0            # interrupt coalescing (Chapter 7)
$ ethtool -l eth0            # NIC queue count (Chapter 18)
```

---

## Appendix D. Glossary

**Backpressure.** A slow consumer slowing a fast producer, usually because a bounded buffer is full.

**Blocking I/O.** An I/O call that puts your thread to sleep until it can complete.

**Bounce buffer.** An intermediate buffer the kernel uses when a device cannot reach the real buffer’s address; costs an extra copy.

**Buffer.** A region of memory that holds data temporarily while it moves between a producer and a consumer.

**Bufferbloat.** Excess latency caused by oversized, unmanaged queues.

**Canonical (cooked) mode.** Terminal mode where the kernel buffers a whole line and releases it on Enter.

**Congestion window (cwnd).** The sender-side limit on unacknowledged TCP data, estimated from network conditions.

**Descriptor.** A small structure telling a device where a buffer is and how big it is; usually stored in a ring.

**DMA (Direct Memory Access).** A device reading or writing RAM on its own, without the CPU copying each byte.

**EAGAIN / EWOULDBLOCK.** Error from a non-blocking call meaning “not ready right now, try later.”

**epoll.** Linux’s scalable readiness-notification mechanism for many descriptors.

**File descriptor (fd).** A small integer your process uses to refer to an open file, pipe, socket, or device.

**Flow control.** Limiting a sender so it doesn’t overwhelm the receiver’s buffer (TCP’s receive window).

**Interrupt (IRQ).** A signal from hardware making the CPU run a handler.

**IOMMU.** A memory-management unit for devices: restricts and translates the addresses they can DMA to.

**Kernel space.** Privileged memory and execution mode of the operating system.

**mmap.** Mapping a file into the address space so it can be accessed like memory.

**NAPI.** Linux’s hybrid networking scheme: interrupt first, then poll in batches under load.

**NIC.** Network interface card.

**NUMA.** Non-uniform memory access: memory is closer to some CPUs than to others.

**Page.** Fixed-size unit of memory management, usually 4 KB.

**Page cache.** Kernel RAM holding recently used file data in pages.

**Page fault.** CPU exception when a program touches a page that isn’t currently mapped; the kernel fixes it (or kills the process).

**PIO (Programmed I/O).** The CPU copying data to or from a device itself, byte by byte or word by word.

**Polling.** Repeatedly checking whether something is ready.

**Privilege wall.** The CPU-enforced boundary between user mode and kernel mode.

**Receive window (rwnd).** The amount of buffer space a TCP receiver says it has left.

**Ring buffer.** A fixed-size circular queue with a read cursor and a write cursor.

**RSS (Receive Side Scaling).** Spreading a NIC’s incoming connections across multiple queues and cores.

**Scatter-gather DMA.** A device collecting data from several separate memory regions in one transfer.

**Stream.** An ordered sequence of bytes consumed or produced piece by piece, backed by a buffer.

**Syscall.** A controlled request from user mode into the kernel.

**TCP.** The Internet’s reliable, ordered byte-stream protocol, built on unreliable packets.

**User space.** The unprivileged memory and execution mode where applications run.

**Zero-copy.** Moving data without the CPU copying the payload between buffers.

---

## Appendix E. Exercises and Mini-Projects

The best way to make this stick is to build things. In rough order of difficulty:

1. **Threaded ring buffer.** Extend the ring buffer from Chapter 2 into a producer thread and a consumer thread. Use a mutex and condition variables (or semaphores) to make the empty case block and the full case block. Watch the throughput change as you vary `CAP`.
2. **Your own `getchar`.** Write `my_getchar(fd)` that keeps a 4 KB buffer and refills it with one `read()` when empty. Compare its syscall count to a raw one-byte `read` using `strace -c`.
3. **Mini `cat` with `mmap`, then with `sendfile`.** Copy a big file to a socket or another file three ways: `read`/`write` loop, `mmap`/`write`, and `sendfile`/`copy_file_range`. Time them cold and warm (Chapter 10.4).
4. **Message framing.** Take the `recv_all` helper from Chapter 12 and build a tiny protocol: a 4-byte length prefix, then that many bytes. Write both a client and a server, and test them with deliberately short writes.
5. **Backpressure by hand.** Modify the Chapter 15 echo server to keep a per-client output buffer, handle partial `write`s, and register for `EPOLLOUT` only while that buffer is non-empty. Test with a client that sends data but never reads.
6. **Watch the window close.** Run a server that accepts a connection and then sleeps without calling `recv()`. Send data from a client and watch `ss -tmi` (and the client’s `send()` eventually blocking). Then let the server drain and watch it recover.
7. **Benchmark the models.** Write a thread-per-connection echo server and compare it with the `epoll` one under a few thousand idle connections (memory use, CPU use).

---

## Appendix F. Where to Go Next

**Guides in the same spirit**

- *Beej’s Guide to Network Programming* (the classic, friendly introduction to sockets) and *Beej’s Guide to C Programming*, both free online.

**The manual pages are your friend.** On Linux, try `man 2 read`, `man 2 write`, `man 2 mmap`, `man 2 sendfile`, `man 7 pipe`, `man 7 socket`, `man 7 tcp`, `man 7 epoll`, and `man 2 io_uring_setup`.

**Books**

- *The Linux Programming Interface* by Michael Kerrisk: the definitive reference for everything in Parts II–V.
- *Computer Systems: A Programmer’s Perspective* by Bryant and O’Hallaron: how hardware, memory, and I/O fit together.
- *Operating Systems: Three Easy Pieces* by Arpaci-Dusseau (free online): clear chapters on scheduling, virtual memory, and I/O devices.
- *UNIX Network Programming* by W. Richard Stevens, and *TCP/IP Illustrated*: the deep dive into sockets and TCP.
- *Linux Device Drivers* (free online; the editions are older but the concepts of interrupts and DMA remain valid) for the hardware side.
- *Systems Performance* by Brendan Gregg, for measuring all of this on real machines.

**Papers and articles**

- Ulrich Drepper, “What Every Programmer Should Know About Memory,” for caches and memory hardware.
- The Linux kernel documentation (kernel.org) on the DMA API and on NAPI.

(Titles and availability change over time, so check each for the current edition and location.)

---

*End of the book. Go break something with strace, and happy hacking.*
