---
layout: post
title: Green file descriptors
date: 2016-02-11
---

In this post, I describe how I placed a class reading from a tap interface in a
greenthread. This was not completely trivial, since greenthreads are expected
to be cooperatively release the CPU. A method that blocks on a read system call
hogs the CPU all to itself.

## Introduction

As part of my work on DragonFlow[1], I wrote a testing framework in python,
which needed to read and write packets to tap devices. Since there were several
tap devices, each tap device is handled by a thread. Since I was assimilating
into an existing system already using greenthreads, that's what I had to use.

Greenthreads[2] are 'cooperative' threads, AKA non-preemptive multitasking.
This means that the programme only seems multi-threaded. It is not. Each thread
has to release the CPU to allow other threads to run.

This is not a problem in this case, as most threads will be stuck waiting for
packets, and will not need the CPU most of the time. However, if a function
blocks on a read call, it does not release the CPU.

## Preliminaries

The class handling the tap devices is TunTapDevice from python-pytun[2]. It
comes with its own read method. The threads are created using eventlet's
*eventlet.GreenPool().spawn()*[3]. This creates our greenthreads.

In general, when using eventlet, one calls *eventlet.monkey_patch()* [4]. This
practically replaces the modules and functions to 'play nicely', e.g.
cooperatively release the CPU.

I went ahead and tried this anyway. The python script blocked on the read
operation. The threads were not playing nice.

TunTapDevice's read method is not patched by eventlet, my guess is since
eventlet doesn't know about it. So when calling TunTapDevice's *read* function,
we get the original read function, which looks like this (in version 2.2.1):

~~~ c
static PyObject* pytun_tuntap_read(PyObject* self, PyObject* args) {
    pytun_tuntap_t* tuntap = (pytun_tuntap_t*)self;
    unsigned int rdlen;
    ssize_t outlen;
    PyObject *buf;

    if (!PyArg_ParseTuple(args, "I:read", &rdlen))
    {
        return NULL;
    }

    /* Allocate a new string */
#if PY_MAJOR_VERSION >= 3
    buf = PyBytes_FromStringAndSize(NULL, rdlen);
#else
    buf = PyString_FromStringAndSize(NULL, rdlen);
#endif
    if (buf == NULL)
    {
        return NULL;
    }

    /* Read data */
    Py_BEGIN_ALLOW_THREADS
#if PY_MAJOR_VERSION >= 3
    outlen = read(tuntap->fd, PyBytes_AS_STRING(buf), rdlen);
#else
    outlen = read(tuntap->fd, PyString_AS_STRING(buf), rdlen);
#endif
    Py_END_ALLOW_THREADS
    if (outlen < 0)
    {
        /* An error occurred, release the string and return an error */
        raise_error_from_errno();
        Py_DECREF(buf);
        return NULL;
    }
    if (outlen < rdlen)
    {
        /* We did not read as many bytes as we anticipated, resize the
           string if possible and be successful. */
#if PY_MAJOR_VERSION >= 3
        if (_PyBytes_Resize(&buf, outlen) < 0)
#else
        if (_PyString_Resize(&buf, outlen) < 0)
#endif
        {
            return NULL;
        }
    }

    return buf;
}
~~~

In essence, the code reads the size of the read buffer, allocates it, and then
call the read system call. If I had to wager a guess, I'd say that's exactly
how *os.read* is implemented[5].

~~~ c
static PyObject *
posix_read(PyObject *self, PyObject *args)
{
    int fd, size, n;
    PyObject *buffer;
    if (!PyArg_ParseTuple(args, "ii:read", &fd, &size))
        return NULL;
    if (size < 0) {
        errno = EINVAL;
        return posix_error();
    }
    buffer = PyString_FromStringAndSize((char *)NULL, size);
    if (buffer == NULL)
        return NULL;
    if (!_PyVerify_fd(fd)) {
        Py_DECREF(buffer);
        return posix_error();
    }
    Py_BEGIN_ALLOW_THREADS
    n = read(fd, PyString_AsString(buffer), size);
    Py_END_ALLOW_THREADS
    if (n < 0) {
        Py_DECREF(buffer);
        return posix_error();
    }
    if (n != size)
        _PyString_Resize(&buffer, n);
    return buffer;
}
~~~

This looks almost identical, give or take the python 3 support. And the
file-descriptor (fd) argument, which the TunTapDevice instance holds. We can
also try to understand what's so special about eventlet's read
implementation[6].

~~~ python
def read(fd, n):
    """read(fd, buffersize) -> string
    Read a file descriptor."""
    while True:
        try:
            return __original_read__(fd, n)
        except (OSError, IOError) as e:
            if get_errno(e) != errno.EAGAIN:
                raise
        except socket.error as e:
            if get_errno(e) == errno.EPIPE:
                return ''
            raise
        try:
            hubs.trampoline(fd, read=True)
        except hubs.IOClosed:
            return ''
~~~

The *hubs.trampoline* method allows the greenthread to be scheduled out[6].

## Solution

My solution, in the end, was not to call TunTapDevice's read method, but use
the patched *os.read* method.

TunTapDevice exposes the file descriptor it holds, with the *fileno()* method.
I have set the file descriptor to be non-blocking. This way we get EAGAIN
errors rather than have \_\_original_read\_\_ block.

~~~ python
def set_blocking(tap, is_blocking):
    fd = tap.fileno()
    flags = fcntl.fcntl(fd, fcntl.F_GETFL)
    if is_blocking:
        flags |= os.O_NONBLOCK
    else:
        flags &= ~os.O_NONBLOCK
    fcntl.fcntl(fd, fcntl.F_SETFL, flags)
~~~

Now we can use *os.read* to read from the file descriptor manually.

For completeness, it may have been better to patch TunTapDevice. All we need to
do is set the file descriptor to non-blocking upon creation, and replace the
read command. The patching code can be done as follows (this wasn't actually
tested, but it should work):

~~~ python

from pytun import TunTapDevice

def monkey_patch_TunTapDevice():
    __original___init__ = TunTapDevice.__init__
    __original_read = TunTapDevice.read
    def new__init__(self):
        __original___init__(self)
        set_blocking(self, False)
    def new_read(self, size):
        return os.read(self.fileno(), size)
    TunTapDevice.__init__ = new__init__
    TunTapDevice.read = new_read
~~~

And now TunTapDevice should be usable normally. As I said, this last bit wasn't
actually tested. It would also only work for instances of TunTapDevice created
after its patching, and not retroactively.

## References

[1] https://wiki.openstack.org/wiki/Dragonflow

[2] https://pypi.python.org/pypi/python-pytun

[3] http://eventlet.net/doc/modules/greenpool.html

[4] http://eventlet.net/doc/patching.html

[5] https://hg.python.org/cpython/file/v2.7.3/Modules/posixmodule.c

[6] http://eventlet.net/doc/hubs.html

