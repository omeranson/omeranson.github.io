---
layout: post
title: "Callback Function with Opaque Pattern"
date: 2020-11-06
---

Many times, a library or framework allows you to register a callback function.
Usually, you are also allowed to register a pointer (an opaque) that will be
passed to your callback function. In this post, I explain this pattern.

# Introduction

Many times, a library or framework allows you to register a callback function.
Usually, you are also allowed to register a pointer (an opaque) that will be
passed to your callback function.

I thought this was a well known pattern, but a colleague informed me that no,
it isn't. I couldn't find a good explanation of this pattern after a (very)
quick search.  So I am writing one myself.

I think the best way to show the pattern is by an example. Suppose you have a
JSON library, with a for-each function. The function looks something like this:

```C
typedef void array_for_each_callback_t(void*, int, json_object_t*);
void json_array_for_each(json_object_t* array,
		array_for_each_callback_t* callback, void * opaque) {
	// Verify that the parameter `array` is indeed an array
	if (!json_is_array(array)) {
		return;
	}
	// Get the length of the array
	int length = json_array_length(array);
	for (int i = 0; i < length; i++) {
		// Get the ith element in the array
		json_object_t * element = json_array_at(array, i);
		callback(opaque, i, element);
	}
}
```

Here, `json_object_t` represents a json object. It can be anything, such as
a string, number, array, or dictionary. That's why we test it at the beginning
of the function.

`array_for_each_callback_t` is the function type. It receives:
1. a `void *`, the opaque
2. an int, the element index
3. A json object, the element itself

Now the callback function can use that opaque for context, an
accumulator, or even ignore it completely.

For instance, the following sums the array.

```C
// Forward declaration
void __sum_json_array_add(void* opaque, int /* unused */ index, json_object_t * object);

int sum_json_array(json_object_t* array) {
	int sum = 0;
	json_array_for_each(array, __sum_json_array_add, &sum);
	return sum;
}

void __sum_json_array_add(void* opaque, int /* unused */ index, json_object_t * object) {
	// Downcast the opaque
	int* sum = (int*)opaque;
	// Get the integer value of the json object (We assume it's an integer)
	int value = json_integer_to_int(object);
	(*sum) += value;
}
```

Another example could be to create a new array, where each value is incremented.

```C
// Forward declaration
void __increment_and_push_back(void* opaque, int /* unused */ index, json_object_t * object);

int increment_json_array(json_object_t* dest, json_object_t* src) {
	json_array_for_each(src, __increment_and_push_back, dest);
	return sum;
}

void __increment_and_push_back(void* opaque, int /* unused */ index, json_object_t * object) {
	// Downcast the opaque
	json_object_t* dest_array = (json_object_t*)opaque;
	// Get the integer value of the json object (We assume it's an integer)
	int value = json_integer_to_int(object) + 1;
	// Create a json integer object from an int
	json_object_t new_int = json_integer_from_int(value);
	// Append an object to the end of the array
	json_array_push_back(dest_array, new_int);
}
```
Now that we've seen a few examples, we can discuss the pattern in a bit
more detail.

Firstly, let's agree on naming. We'll call the function *the function*. In
the example above, *the function* is `json_array_for_each`. The location
calling this function will be called *the caller*. In the examples above,
these are `sum_json_array` and `increment_json_array`. The function passed
to *the function* is called *the callback function*. In the examples above,
these are `__sum_json_array_add` and `__increment_and_push_back`. Lastly,
the variable passed to *the function* and then passed back to *the callback
function* is called *the opaque*.

Suppose you have a function which allows you to run your own code for
specific events.  These events can be for every element in a collection,
for every sub-object when parsing a stream, or whenever an event happens.

'Run your own code' means call the callback function. To allow the callback
function to operate on some context, even variables local to the caller, it
is passed an opaque. The opaque is set by the caller, and passed to the
callback function. The function itself knows nothing about the opaque. It may
not read it, write to it, and definitely may not dereference it.
In the examples above, *the opaque* is the local variables `sum` and  the
parameter `dest`.

The benefits of this pattern are that the function can be written without any
knowledge of how it will be used. The callback function is called with the
opaque, and it's the callback function's responsibility to integrate with the
rest of the system.

This also allows code reuse. The same callback function, with a different
opaque, can provide the same functionality to different locations or contexts.

On the other hand, by using this mechanism, you trust the function to pass on
the opaque variable as is. It may not make any assumptions on it, and must pass
it on as-is. This sounds easy on paper, but nothing really verifies this for
you. Although I suppose a clang verifier could be easily written for the more
trivial cases.

Additionally, you have to manage the opaque's lifetime yourself. The callback
function has no real way of knowing if the opaque is still valid, or has been
accidentally destroyed. This pattern, as presented here, doesn't even support
reference counting.

Of course, today, the problem this pattern comes to solve is no longer an
issue. Almost every language supports binding or scoping functions. This
includes C++ (C++11 and up), python, go, Lua, and the list goes on.  Not C
though. At least, not trivially. A quick search finds some projects about
this, but I wouldn't recommend writing something like this myself.

In my case,  this came up, because I didn't know what the integration team
will be using.  My code was supposed to send out events, and I didn't want
to limit myself to a language or environment.

# Integration

## Integration with C++

In the above, we've seen how to integrate with C. We can now see how to
integrate this pattern with C++.

 A trivial solution is to provide a function object as the opaque, and have the
 callback function apply it. A function object is anything that can be called.
 These can be e.g., functions, or instances implementing `operator()`.

 ```C++
 template <typename F> invoke(void* opaque, ...) {
 	F* f = (F*)opaque;
	(*f)(...);
 }
```

In the above `...` are the parameters of the function. They can either be
hard-coded, or extracted using some template magic. See, for instance,
[here](https://stackoverflow.com/questions/28509273/get-types-of-c-function-parameters).

In fact, in newer C++ versions, `std::invoke` is a standard library function.

The usage can be as follows:

```C++
struct Sum {
	int sum = 0;
	void operator()(json_object_t* object) {
		sum += json_integer_to_int(object);
	}
};

int sum(json_array_t* object) {
	Sum sum;
	json_array_for_each(array, std::invoke<Sum>, &sum);
	return sum.sum;
}
```

## Integration With Python

Python's [ctypes library](https://docs.python.org/3/library/ctypes.html) allows
you to call external C functions, and to define callbacks that can be passed to
C code. However, passing pointers around is not trivial.

Instead, we'll use the [cffi library](https://cffi.readthedocs.io/en/latest/).
Note that this is by no means a comprehensive tutorial. It's a quick and dirty
example to showcase the pattern. The documentation even says you should use
the newer method.

Since this example is not as trivial and immediate as the C++ version, I'll
use a slightly different example. This way, the example can be compiled and
tested.

We'll start with the C code:

```C
typedef void f_cb(void* opaque, int v);
int f(int n, f_cb *cb, void* opaque);

int f(int n, f_cb *cb, void* opaque) {
        for (int i = 0; i<n; i++) {
                cb(opaque, i);
        }
}
```

This simple function calls the callback for every number between 0 and `n-1`.

Suppose the code sits in `t.c`, it can be compiled to a shared object (this
example was tested on Linux) with the following commands:
```
gcc -c -o t.o t.c -fPIC
gcc -shared -o libt.so t.o
```

The shared object file is `libt.so`.

The python code is as follows:
```
import cffi

ffi = cffi.FFI()
ffi.cdef("""
    typedef void f_cb(void* opaque, int v);
    int f(int n, f_cb *cb, void* opaque);
""")
t = ffi.dlopen("./libt.so")

@ffi.callback("void(void*, int)")
def cb(l_p, i):
	l = ffi.from_handle(l_p)
	l.append(i)

l = []
l_p = ffi.new_handle(l)
t.f(10, cb, l_p)
print(l)
```

Running the python code should print the following to the standard output:
```
[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

## Integration with other languages

Many languages can interface with C. To get a feel for it, you can look [at
this Wikipedia page](https://en.wikipedia.org/wiki/Foreign_function_interface).
By allowing the caller, which may be written in a different language, to
provide a callback and opaque, this pattern allows for very flexible code,
without tight coupling. The only assumptions are that the callback can be
called with the correct signature, and that the opaque can fit in a pointer.

# Examples in the wild

## netfilter\_queue

`libnetfilter_queue` allows you to sniff packets. You configure iptables to
send the packet to a particular queue, and a userspace application to listen
for these packets.

Listening on the userspace side is done by calling `nfq_create_queue`.
The documentation can be found [here](https://netfilter.org/projects/libnetfilter_queue/doxygen/html/group__Queue.htm)l

The function accepts a callback and opaque. The callback is called with the
opaque and additional packet-related information for every packet that is sent
to the queue.

## yajl

The [yajl library](https://lloyd.github.io/yajl/) provides an event-driven
mechanism to parse json strings. The event-driven design allows even partial
json strings to be parsed.

Every callback function is given a context parameter. This parameter is the
opaque in this pattern. This allows you to e.g., build an C struct from the
json string you are parsing.

## Linux kernel

In the linux kernel, this pattern repeats itself quite a bit. However, the
opaque is sometimes tacked on to other objects. For instance, `struct file`
represents an open file. One of its members is `private_data`, which is a
`void *` that can be set by the driver.

This member can be seen in a few other structures for a similar purpose,
such as `event_trigger_data`.

# Conclusion

Callbacks allow the caller to tell functions what to do in different
situations. Passing an opaque to the callback gives this behaviour context.
This allows for a very flexible mechanism.

I concentrated on C in this post. That's because it's easiest to interface
with it. The pattern itself, however, is valid for any language. Especially
if you don't cross language boundaries. Not every language supports bound or
scoped functions. This pattern gives a good solution in that case.

When I used this pattern, my thoughts were: 'I don't know anything about
the consumers of this code. So I'll make sure they can use it, no matter what.'
