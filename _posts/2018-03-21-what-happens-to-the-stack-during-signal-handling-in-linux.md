---
layout: post
title: "What happens to the stack during signal handling in Linux?"
date: 2018-03-21
---

In this post I explore the torment a stack goes through when a Linux process
handles a signal.

## Rationale

In python, `eventlet` gives you a timeout functionality. It looks like this:

```python
timeout = eventlet.Timeout(some_timeout)
try:
	...  # Stuff that will throw an exception after some_timeout time.
except eventlet.Timeout as t:
	if t is timeout:
		... #  Timeout happened
```

I wanted something similar in C++:

```C++
Timeout timeout(some_timeout);
try {
	TimeoutStarted ts = timeout.start();
	...  # Stuff that will throw an exception after some_timeout time.
} catch (Timeout * t) {
	if (timeout == *t) {
		printf("Timeout!\n");
	} else { throw; }  // Reraise
}
```

Where `timeout.start()`, among other things, sets an alarm (There are many
problems with this design, but it makes for a nice topic):

```C++
class Timeout {
	// ...
	TimeoutStarted start() {
		// ...
		signal(SIGALRM, alarm_handler);
		alarm(m_timeout);
		// ...
	}
	// ...
}
```

And the signal handler just raises the exception (should work, right?):

```C++
Timeout * timeout_instance;  // Set during constructor

void alarm_handler(int signum) {
	throw timeout_instance;
}
```

## The Problem

There are some issues with this design. By using alarm in this way, and having
a single global pointer for the timeout instance, there can be a single timeout
used at any given time.

Additionally, `timeout_instance` is global, and technically accessing it
from a signal handler is naughty. But this is a small issue which can be
easily be worked around, say using `std::atomic` (the only writer is the
`class Timeout`).

Lastly, and the biggest, is that you *should not* raise an exception from
a signal handler. It causes undefined behaviour.

I used [1] as my reference (too lazy to find the standard).

I am not going to solve this problem. I am just going to understand why this
happens.

## Scope

Before we go any further, there are a few assumptions I am taking.

I'm going to completely ignore threads.  I'm looking at a single threaded
programme now. Maybe I'll get to threads in a different post.

There's a nifty feature which makes the signal handler run on a completely
different stack. This is set in the call to `sigaction` and `signalstack`.
I'm not going to go into that in too much detail. It complicates the
story, which is not too simple to begin with.

The subject of this post is mostly the stack. There are other things that
require looking into, e.g., signal information, and flags, but I will randomly
ignore these.

## The High Level

Basically, the story goes like this:

* The kernel is notified that the process received a signal

* Eventually, `handle_signal` is called. The implementation is architecture
  dependant.

* `handle_signal` calls `setup_rt_frame`. This is also
  architecture specific, and I checked only x86 and arm. I assume the
  behaviour is similar in other architectures.

* `setup_rt_frame` adds a stack frame, which includes all the necessary
  information to restore the execution state. This includes e.g., registers,
  return address, and signal information.

* The programme counter (or instruction pointer, or that magic register that
  points to the next instruction to be executed) is updated to point to the
  signal handler.

* The return address is set to be the _restorer_, which can be set from
  user-space. By default, it is `sigreturn(2)`, which resets the programme
  state (registers, stack, signal information, and anything else saved in
  `setup_rt_frame`) to before the signal handling.

* When the signal handler returns, it returns to the _restorer_. The
  _restorer_ restores the programme to its original state, before the
  signal was handled.

## The Low Level

I'm going to concentrate on x86, and version 4.15.11. Most of the code is
taken from signal.c[4]. There's also going to be a lot of hand-waiving.
This is not a code-walk, I just wanted to understand the general idea,
and now I'm dumping it here.

So, in `handle_signal`:
```C
	if (v8086_mode(regs))
		save_v86_state((struct kernel_vm86_regs *) regs, VM86_SIGNAL);
```

We start by checking if we're in v8086 mode, which can happen only if the
kernel is compiled for 32 bits, and the specific flag (`X86_VM_MASK`) is
set. If so, save the specific registers into `current->thread`. `current`
is a global pointer describing the current process[2]. The field `thread`
is described as 'CPU-specific state of this task'.

```C
	/* Are we from a system call? */
	if (syscall_get_nr(current, regs) >= 0) {
		/* If so, check system call restarting.. */
		switch (syscall_get_error(current, regs)) {
		case -ERESTART_RESTARTBLOCK:
		case -ERESTARTNOHAND:
			regs->ax = -EINTR;
			break;

		case -ERESTARTSYS:
			if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
				regs->ax = -EINTR;
				break;
			}
		/* fallthrough */
		case -ERESTARTNOINTR:
			regs->ax = regs->orig_ax;
			regs->ip -= 2;
			break;
		}
	}
```

The general idea here is that if we're in the middle of a syscall
(`syscall_get_nr`), either restart the system call, or have it return stating
it was interrupted (`-EINTR`). There are several types of restarts, and they
are explained rather nicely here[3].

```C
	/*
	 * If TF is set due to a debugger (TIF_FORCED_TF), clear TF now
	 * so that register information in the sigcontext is correct and
	 * then notify the tracer before entering the signal handler.
	 */
	stepping = test_thread_flag(TIF_SINGLESTEP);
	if (stepping)
		user_disable_single_step(current);
```

Basically, disable single stepping. This is a debugging feature.

```C
	failed = (setup_rt_frame(ksig, regs) < 0);
```

This is where the real magic happens.

In `setup_rt_frame`, it chooses which inner function to take according to the
type of architecture. We will stick to the default, 64-bit case, which calls
`__setup_rt_frame`.

Inside `__setup_rt_frame`, we have:
```C
	frame = get_sigframe(&ksig->ka, regs, sizeof(*frame), &fpstate);
```

In essence, `get_sigframe` reads the stack pointer (`sp`), allocates
space by substracting `frame_size` and aligning the stack. In this
context, `frame_size` is `sizeof(*frame)`, where `frame` has type
`struct rt_sigframe`.

Verify that the newly allocated frame is writable.

```C
	if (!access_ok(VERIFY_WRITE, frame, sizeof(*frame)))
		return -EFAULT;
```

If `SA_SIGINFO` was set, the signal handler is also given the signal info,
(a `siginfo` structure, explained in the `sigaction(2)` man page), and a
context information object, explained in the `getcontext(3)` man page.

If `SA_SIGINFO` was set for this signal handler, initialise the signal info
structure.

```C
	if (ksig->ka.sa.sa_flags & SA_SIGINFO) {
		if (copy_siginfo_to_user(&frame->info, &ksig->info))
			return -EFAULT;
	}
```

Initialize the context information.

```C
		put_user_ex(frame_uc_flags(regs), &frame->uc.uc_flags);
		put_user_ex(0, &frame->uc.uc_link);
		save_altstack_ex(&frame->uc.uc_stack, regs->sp);
```

The next bit is interesting. It is from the 32-bit version of this
function.  I wanted to show the vdso option as well. The 64-bit version
enforces the use of `SA_RESTORER`:


```C
		/* Set up to return from userspace.  */
		restorer = current->mm->context.vdso +
			vdso_image_32.sym___kernel_rt_sigreturn;
		if (ksig->ka.sa.sa_flags & SA_RESTORER)
			restorer = ksig->ka.sa.sa_restorer;
		put_user_ex(restorer, &frame->pretcode);
```

This code adds the return address, i.e. the code to execute when the
signal handler returns.

Initially, the `restorer` is `sigreturn`, as it appears in vdso (I'm guessing
it's actually vsyscall, but I didn't check).

If, however, the `SA_RESTORER` flag is set, then the restorer is taken from
the information passed to `sigaction` when setting the signal handler. The
man page is very specific about not using this flag in applications. I'm
guessing it's there for standard libraries. `musl` adds it automatically,
overwriting it if it was passed from the client.

The last line of the above stanza places sets the selected restorer as the
signal handler's restore address.

The next line of code allows GDB to detect the magic stack frame we just added.
It is also from the 32-bit version, but I thought it is very interesting, since
historically there was a marker for these special stack frames:

```C
		put_user_ex(*((u64 *)&rt_retcode), (u64 *)frame->retcode);
```


 Where the value of `rt_retcode` is the assembly for `movl
 $__NR_rt_sigreturn, %ax ; int $0x80`:

```C
static const struct {
	u8  movl;
	u32 val;
	u16 int80;
	u8  pad;
} __attribute__((packed)) rt_retcode = {
	0xb8,		/* movl $..., %eax */
	__NR_rt_sigreturn,
	0x80cd,		/* int $0x80 */
	0
};
```

I'll stress that the above is not used in the general case, or even in 64-bit.

Next, set up the signal context, part of the user context structure
(I'll send you again to `getcontext(3)`). I will mention that this is
where all the registers are saved to the special stack frame.

```C
	err |= setup_sigcontext(&frame->uc.uc_mcontext, fpstate, regs, set->sig[0]);
	err |= __copy_to_user(&frame->uc.uc_sigmask, set, sizeof(*set));
```

And if we didn't fail:

```C
	if (err)
		return -EFAULT;
```

Replace the actual register values for the running process:
```C
	/* Set up registers for signal handler */
	regs->di = sig;
	/* In case the signal handler was declared without prototypes */
	regs->ax = 0;

	/* This also works for non SA_SIGINFO handlers because they expect the
	   next argument after the signal number on the stack. */
	regs->si = (unsigned long)&frame->info;
	regs->dx = (unsigned long)&frame->uc;
	regs->ip = (unsigned long) ksig->ka.sa.sa_handler;

	regs->sp = (unsigned long)frame;
```

Note that `sp` holds the location of the new stack frame, `ip` is the location
of the signal handler. 

```C
		/*
		 * Clear the direction flag as per the ABI for function entry.
		 *
		 * Clear RF when entering the signal handler, because
		 * it might disable possible debug exception from the
		 * signal handler.
		 *
		 * Clear TF for the case when it wasn't set by debugger to
		 * avoid the recursive send_sigtrap() in SIGTRAP handler.
		 */
		regs->flags &= ~(X86_EFLAGS_DF|X86_EFLAGS_RF|X86_EFLAGS_TF);
		/*
		 * Ensure the signal handler starts with the new fpu state.
		 */
		if (fpu->initialized)
			fpu__clear(fpu);
```
Lastly, after clearing the floating-point unit and resetting some flags
(the documentation seems fairly verbose), call `signal_setup_done`. Upon
failure, `signal_setup_done` _forces_ a segmentation fault signal,
meaning the application can't block it.

Upon success, there's some more signal bookkeeping which I will not go into,
and the debugger is notified (if attached).

That's it. Once we switch back to userspace, the process will start executing
in the signal handler. Once it returns, it will land in the restorer, which
will restore the process state to before the signal was sent.

## Conclusion

So basically, back to our original issue: We want to inject an exception. The
exception injection code will have to detect the special stack frame (easily
done with the GDB stub, we can't really count on it), and use the stored
registers to do its magic.

The only problem is that all this is architecture and version specific. Not
so good...

I am not going to solve this problem. That would require understanding how
exception handling happens, and possibly being able to get this information
in a portable way (maybe another vdso function).

## References

[1] http://en.cppreference.com/w/cpp/utility/program/signal

[2] http://www.xml.com/ldd/chapter/book/ch02.html

[3] https://stackoverflow.com/questions/9576604/what-does-erestartsys-used-while-writing-linux-driver

[4] https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/signal.c
