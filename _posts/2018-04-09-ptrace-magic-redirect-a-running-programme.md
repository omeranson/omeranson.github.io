---
layout: post
title: "ptrace magic: Redirect a running programme"
date: 2018-04-08
---

In this post I will show how to use ptrace to redirect the standard output of
a running programme.

# Rationale

Many times, I found myself running a programme, and having it spew its output
to the standard output to make sure it started properly. Usually things I
wrote that should run in the backgrounds, but were riddled with bugs.

So now it's running. I can send it to the background with CTRL+z
and `bg`. But it is being noisy, and it's difficult to work on the
terminal. (Yes, I know opening a new terminal is easy. That's not
the point!)

So I wrote this little utility, that hacks into the daemon (or other process),
and overrides the standard output with a given file. Actually, it works for
any file descriptor, but I mostly tested it on the standard output.

It isn't very useful. But I found it slightly cool.

# High Level

The simplest API I could think of was a command line utility getting the PID
to connect to, the file descriptor to overwrite, and the file to overwrite the
file descriptor with:

```
./redirect: Usage: ./redirect <pid> <fd> <new outpath>
```

This API may need to be extended to include the flags of `open(2)`.

So basically, we connect to the target process with ptrace. We make it open
the file by calling `open(2)`. We override the file descriptor by calling
`dup2(2)`. And then we go have a beer.

Wait. Before the beer.

The code to do all of this has to run from within the context of the other
process. We can't code that directly. Additionally, `open(2)` needs a pointer
to the file name. That also has to be somewhere in the target process' memory
space.

So we need some place to put the code, and execute it. And we need some place
to put the file name, and point to it.

My solution was to use the code segment to store everything. `redirect`
(this utility) does the following:

* Make a backup of the current code segment and registers. We're going to write
  both to the memory, and the registers. And we're going to want to revert all
  our changes.

* Copy over the filename. The start address is the instruction pointer. This
  way we already have the filename address, and everything is written to the
  same place, and it's all close together.

* Overwrite the next instruction with the `syscall` instruction. Overwrite the
  instruction after that with the `int3` (debugger breakpoint) instruction.

* Update the registers. These are used to pass parameters to the syscalls.

* Let the target process run for a bit. It will do two instructions: syscall
  and interrupt. The interrupt will return control to the tracing process.

  `ptrace` let's you take control of the target process when calling and returning
  from a syscall. The benefit of that solution is that you don't have to inject
  and catch the `int3` instruction.

* Revert what we did to the process. Restore the code segment and the registers.

This is a very simplistic implementation. It ignores memory permissions. i.e.
what if the code segment is not readable? (Actually, the kernel does the
reading, so it might work...)

It also ignores signal handling. This is not really an issue, since the tracing
process can capture all signals and block them for the target process until the
redirection is complete. But in the current implementation, you probably don't
want a signal being sent while opening the file. It might cause the system call
to return early, and we don't handle that very well.

But this is the gist of it.

I am going to call the process that calls `ptrace` the _tracing_
process. The process to which we attach is called the _target_ process,
or _tracee_, or sometimes the _other_ process. It should be clear from
context, but just in case.

# Code

So now we can work through the code. The code itself is available
[here](https://github.com/omeranson/redirect), commit 34cbd0489b9b4d6b8b72f1ba3902704d46207e1e.  Only
versions for 32-bit and 64-bit x86 exist. And the 32-bit version was
tested five years ago. (And the 64-bit version wasn't heavily tested either.)

The `main` and `redirect_output_by_strings` functions are your run-of-the-mill
bad command line argument handling. I say bad, because usually I'm a firm believer
in `getopt`. Their goal is to parse the PID and file descriptor into integers,
and then call `redirect_output`.


```C
int redirect_output(pid_t pid, int fd, const char * outpath) {
```

The next block attaches to process with PID `pid`. 

```C
	rc = ptrace(PTRACE_ATTACH, pid, NULL, NULL);
	if (rc) {
		die("attach");
	}
	waitpid(pid);
```

The attached process (the tracee) is then sent a `SIGSTOP` signal. Once
the tracee actually stops (I think it may wait till the process is
scheduled to handle the signal, or maybe the process is in kernel space
and can't be stopped), the `waitpid` call will return.

This code predates `PTRACE_SEIZE`, which is preferred due to some edge-cases.
Additionally, the tracee can be stopped without the use of signals by using
`ptrace(PTRACE_INTERRUPT, ...)`.

```C
	struct user_regs_struct regs;
	rc = ptrace(PTRACE_GETREGS, pid, NULL, &regs);
	if (rc) {
		die("getregs");
	}
	memcpy(&oldregs, &regs, sizeof(regs));
```

This backs up the current registers into the local variables `regs` and `oldregs`.
We're going to use `regs` to send new register values to the tracee, so we
need an additional copy.

```C
	size_t size = calculate_size(outpath);
	void * backup = malloc(size);
	addr = (void*)regs.rip;
	getdata(pid, addr, backup, size);
```

This backs up the next `size` bytes in the code segment. `calculate_size` sums the
length of `outpath` (the command line argument), including its terminating NUL
character, and the size of the instructions we're going to inject.

We will see that we can only copy _word_ bytes back and forth to the other process.
So `calculate_size` pads the return value so that it is divisible by _word_
bytes. On my system it's 4 (`getconf WORD_BIT`). But in case that ever changes,
we have a nice typedef for it:

```C
typedef uint32_t word_t;
#define word_size sizeof(word_t)
```

`getdata` copies the `size` bytes of data from the tracee at address
`addr`, to the tracer at address `backup`. We'll see exactly how this is done
in a minute.

So in essence, we're copying some bytes from the code segment, just where the
tracee was about to execute.

I'll now digress and show `getdata` and `putdata`. These are helper functions
to copy from and to the tracee.

```C
void getdata(pid_t child, const word_t * addr, word_t *str, int len) {
	int i;
	int j = ((len + word_size -1) / word_size);
	for (i = 0; i < j; i++) {
		*str++ = ptrace(PTRACE_PEEKDATA, child, addr++, NULL);
	}
}
```

`getdata` reads `len` bytes from the process `child`. The source address is
`addr`, a pointer to the tracee's virtual memory address space, and the
destination is `str`, a pointer to the tracer's virtual memory address space.

`PTRACE_PEEKDATA` reads and returns one word of data from the tracee. To read
more than a single word, `ptrace(PTRACE_PEEKDATA, ...)` has to be called
multiple times.

Since `len` does not have to be divisible by the word size, we set `j` to be
the number of words to read, rounded *up*. We assume `str` is big enough. We
made sure of that in `calculate_size`.

`putdata` follows the same vein.

```C
void putdata(pid_t child, const word_t * addr, const word_t *str, int len) {
	int i;
	int j = ((len + word_size -1) / word_size);
	for (i = 0; i < j; i++) {
		ptrace(PTRACE_POKEDATA, child, addr++, *str++);
	}
}
```

Now that we backed up the data in the tracee's code segment, it is time to
overwrite it.

```C
	void * data = alloca(size);
	memset(data, 0, size);
	memcpy(data, outpath, outpath_len);
	memcpy(data+outpath_len, insert_code, sizeof(insert_code));
	putdata(pid, addr, data, size);
```

Recall that `size` may pad by a few extra bytes to be an integral number of
words. Therefore, some of `data` may be uninitialized but still copied over.
So let's just set everything to 0. Why `alloca` rather than `calloc`? This way
I don't need to free `data` later.

So `data` contains the path to the file, and the new instructions. We write
all this to `addr`, which was the tracee's instruction pointer. So we even
have a pointer to the path in the other process. Woohoo!

The new instructions are:
```C
static char insert_code[] = "\x0f\x05\xcc";
```

`\x0f\x05` is the binary code for `syscall`. `\xcc` is the binary code for
`int3`, which is what debuggers use for breakpoints. This is also the reason
it takes exactly one byte.

Now we are going to tell the tracee to call the following system calls:

* `open(path, O_WRONLY)`

  Open the file in write-only mode, and return the file descriptor.

* `dup2(open_retval, fd)`

  Overwrite the existing file descriptor with the new one. Both file
  descriptors now point to the same file.

* `close(open_retval)`

  Close the new file descriptor. The file will be accessed via the old,
  overriden file descriptor.

For each of these, we overwrite the tracee's registers, and tell it
to continue.  We regain control when the syscall returns, because the
next instruction is `int3`.

```C
	regs.rip = (datatype)(addr+outpath_len);
	regs.rax = 2; /* Open */
	regs.rdi = (datatype)addr;
	regs.rsi = O_WRONLY | O_CREAT;
	regs.rdx = S_IRWXU | S_IRWXG | S_IRWXO;
	rc = ptrace(PTRACE_SETREGS, pid, NULL, &regs);
```

`rip` is the instruction pointer. 64-bit linux follows the _System V
AMD64 ABI_.  The arguments are passed via the following registers,
in order: `rax`, `rdi`, `rsi`, `rdx`, `rcx`, `r8`, and `r9`. `rax`
is the system call number (2 for `open`).  `open` has 3 parameters.

The first argument is passed via `rdi`. It is the path to the file to open.
We put this file in `addr` earlier, so that's what we're passing.

The second argument (in `rsi`) is `flags`. It states that we open the file
in write-only mode, and create it if it doesn't exist.  The third argument
(in `rdx`) is `mode`.  In case the file is created, this argument states
that the permissions will be world readable, writable, and executable.

`ptrace(PTRACE_SETREGS, ...)` updates the tracee's registers with the data
in `regs`.

We now tell the tracee to continue, and wait for it to stop again on the `int3`
instruction we inserted.

```C
	rc = ptrace(PTRACE_CONT, pid, NULL, NULL);
	if (rc) {
		die("cont");
	}
	waitpid(pid, NULL, 0);
	rc = ptrace(PTRACE_GETREGS, pid, NULL, &regs);
```

The last call to `ptrace(PTRACE_GETREGS, ...)` repopulates `regs` with the
registers from the tracee. We need this to read `open`'s return value, in `rax`.

Next, we want to call `dup2(fd, rax)`.

```C
	regs.rip = (datatype)(addr+outpath_len);
	regs.rdi = regs.rax;
	regs.rax = 33; /* dup2 */
	regs.rsi = fd;
	rc = ptrace(PTRACE_SETREGS, pid, NULL, &regs);
```

We return the instruction pointer to our `syscall` and `int3` instructions. We
set the syscall number (in `rax`) to `dup2` (33). The first argument (in `rdi`)
to the newly opened file, available in `rax` (note: Until we overwrite it). The
second argument (in `rsi`) is `fd`, the file descriptor we want to overwrite.

We then update the tracee's with the new register values, using
`ptrace(PTRACE_SETREGS, ...)`.

We again tell the tracee to continue, and wait for it to stop again on
the `int3` instruction we inserted.

```C
	rc = ptrace(PTRACE_CONT, pid, NULL, NULL);
	if (rc) {
		die("cont");
	}
	waitpid(pid, NULL, 0);
	rc = ptrace(PTRACE_GETREGS, pid, NULL, &regs);
```

Lastly, we want to close the new file descriptor. The file will be written to
via the old file descriptor `fd`.

```C
	regs.rip = (datatype)(addr+outpath_len);
	regs.rax = 3; /* close */
	rc = ptrace(PTRACE_SETREGS, pid, NULL, &regs);
	rc = ptrace(PTRACE_CONT, pid, NULL, NULL);
	if (rc) {
		die("cont (4)");
	}
	waitpid(pid, NULL, 0);
```

The system call number for `close` is 3 (in `rax`). The first argument is the
file descriptor to close, in `rdi`, which still holds `open`'s return value
from the preparation of `dup2`.

Lastly, we put everything back as we found it.

```C
	putdata(pid, addr, backup, size);
	rc = ptrace(PTRACE_SETREGS, pid, NULL, &oldregs);
```

Recall that `backup` holds the old data. It was populated with the call to
`getdata` above. `oldregs` contains a copy of the original registers of the
tracee.

```C
	rc = ptrace(PTRACE_DETACH, pid, NULL, NULL);
```

That's it. We're done. `PTRACE_DETACH` stops the tracing, and let's the tracee
continue on its merry way.

# Conclusion

So now we know how to use `ptrace`. There is a lot more info in the man page.
So run `man ptrace` and enjoy the view, feeling slightly proud that it's not
completely in Quenya.

