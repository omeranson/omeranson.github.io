---
layout: post
title: "X forwarding from where, now? How?"
date: 2020-10-07
---

In this post I'll discuss how to use `socat` to magically bring X-forwarding
to network namespaces (and, I guess, containers), with no direct networking.

# Introduction

So I was working on this project. I set up a network namespace
with no network, and fired up a graphical monitoring system inside it. It
failed to start. At the time, I was connected to the host running the network
namespace via SSH.

I behaved. I enabled X forwarding in SSH. But it still didn't work.

```
[root@netnshost user]# xclock
Error: Can't open display: localhost:10.0
```

Huh. I guess X forwarding works over the network. And this network
namespace is disconnected from the host network. Who knew?

# The Solution

Luckily, an X server can be connected over a named UNIX socket, and not just
over the network. That's great, but how do we connect a UNIX socket to the
TCP socket?

We'll also need to know where to connect.

## socat

Enter `socat` ([This is their
website](http://www.dest-unreach.org/socat/)). `socat` allows you to
create a bidirectional stream between two sockets. The sockets can be
anything (well, within reason). This means TCP, UDP, SSL, HTTP proxy,
UNIX sockets, socks, TUN devices, and so on and so forth.

I won't presume to go over all the options. Instead, I'll just show-case what
it can do with this simple example.

## Basic `socat` Usage

After I SSHed into the host, I checked the DISPLAY environment variable. This
environment variable tells X programmes where to find the X server.
For more information on the DISPLAY environment variable, this askubuntu
response is great: [https://askubuntu.com/a/432257].

```
[user@client ~]$ ssh netnshost -X
Last login: Wed Jul 11 17:05:10 2018 from 10.0.0.3
[user@netnshost ~]$ echo $DISPLAY
localhost:10.0
```

So this means we now have an X server on `netnshost`, listening on the
local address, on port `6000+10=6010`. `netstat` verifies this:

```
[user@netnshost ~]$ sudo netstat -lntp
[sudo] password for user:
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
...
tcp        0      0 127.0.0.1:6010          0.0.0.0:*               LISTEN      31880/sshd: user@
...
```

After a little bit of digging, I found out that when the X server listens on
a (named) UNIX socket, the file is located in `/tmp/.X11-unix/`. The filename
is of the form `X????`, where `????` is replaced by the display number. e.g.,
for `DISPLAY=:1234`, the UNIX socket is `/tmp/.X11-unix/X1234`.

So basically, we just need to create a unix socket in `/tmp/.X11-unix/`, and
connect it to `localhost:6010`. Easy peezy.

```
[user@netnshost ~]$ socat tcp:localhost:6010 unix-listen:/tmp/.X11-unix/X1234
```

Now we can send it to the background, e.g. with `ctrl+z ; bg`, or running it
in the background in the first place.

## Testing on the Host

We set everything up! Now we can change the DISPLAY environment variable,
and test our solution.  First, we'll test it directly on the host:
```
[user@netnshost ~]$ export DISPLAY=:1234
[user@netnshost ~]$ xclock
X11 connection rejected because of wrong authentication.
```

Ah. Well, it turns out not just anyone can connect to an X server. You need
a cookie. You can see the cookies you have with `xauth list`.

```
[user@netnshost ~]$ xauth list
netnshost/unix:10  MIT-MAGIC-COOKIE-1  d200ce35de774169df2fd17c774ee046
```

So I have a cookie for display *:10*, which makes sense. I'm guessing SSH
took care of that for me. But we changed the name of the display. It's now
*:1234*. So let's add the cookie to that display as well.

```
[user@netnshost ~]$ xauth add :1234 MIT-MAGIC-COOKIE-1  d200ce35de774169df2fd17c774ee046
```

Note that we are using the same cookie on purpose. It's the same X session.
We just changed the name on our end.

And now:
```
[user@netnshost ~]$ socat tcp:localhost:6010 unix-listen:/tmp/.X11-unix/X1234 &
[user@netnshost ~]$ xclock
```

It works! For me, at least.

You may notice something annoying by now - you need to restart `socat`
for every call. We'll fix that. But first, let's get it working in a
network namespace.

## In a Namespace

Let's create our namespace.

```
[user@netnshost ~]$ sudo ip netns add Xtest
```

And open a shell inside:

```
[user@netnshost ~]$ sudo ip netns exec Xtest bash
```

Note that we are now root. If you want to become `user` again, you can run
```
[root@netnshost ~]$ sudo -s -u user
```

Re-add the xauth cookie. You can run `xauth list` to see if it's already there.
If `xauth` complains that your `.Xauthority` file doesn't exist, go ahead and
create it.

```
[root@netnshost ~]$ export DISPLAY=:1234
[root@netnshost ~]$ touch /root/.Xauthority
[root@netnshost ~]$ xauth add :1234 MIT-MAGIC-COOKIE-1  d200ce35de774169df2fd17c774ee046
[root@netnshost ~]$ xclock
```

Works for me!

## `socat` keeps exiting

Now, you remember the annoying problem from before? Where `socat` exits after
every session? So the solution is to use the `fork` option on the UNIX socket:

```
[user@netnshost ~]$ socat unix-listen:/tmp/.X11-unix/X1234,fork tcp:localhost:6010
```

The `fork` option tells `socat` to fork a child once a connection is
established, allowing the parent process to continue listening. `socat`
exits once it detects the connection closes. I guess in this context, it
means the child exits.

It appears that the order in which the sockets are specified is important. If
the TCP socket is first, then `socat` first connects, and then listens on the
UNIX socket. If the UNIX socket is first, then `socat` waits for a connection
on the UNIX socket.

# What About Docker?

It works! As long as `selinux` let's you in.

Basically, you have to take the same steps. You create the named socket
with `socat`. You make sure your container is allowed to access the file.
That's what the call to `chcon` does.

```
chcon -t svirt_sandbox_file_t /tmp/.X11-unix/X1234
docker run -ti --rm --name xeyes -e DISPLAY=:1234 -v /tmp/.X11-unix/X1234:/tmp/.X11-unix/X1234 --entrypoint bash gns3/xeyes
```

You also have to make sure the process in the container is allowed to connect
to the socket file. The best I could find is this unix.stackexchange post
([https://unix.stackexchange.com/q/386767]). There are several solutions
there, but I agree with the author that none of it is strict enough (in the
spirit of SELinux). Perhaps an SELinux primer is in order.

Once you're in the container, you have to make sure the xauth information
is passed as well. Copying over the `.Xauthority` file is not enough, as
the hostname may have changed.

Lastly, run your process:

```
root@1cfd10e9ba20:/# xeyes
```

And voilà! I bet this would work with any container technology, e.g., LXC.

# Conclusion

We have seen a mechanism that allows forwarding the X server
all the way to an isolated network environment within the destination SSH
server. This means that a network namespace, or a container, can now be set-up
to reach the X server, locally or remotely.

In conclusion:

1. `socat` is cool!

2. `socat` is cool!

3. SELinux is hard

# Acknowledgement

Thanks to Itamar Ofek and Shachar Snapiri for helping with the writing and
testing of this.
