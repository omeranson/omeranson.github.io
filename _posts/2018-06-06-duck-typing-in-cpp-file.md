---
layout: post
title: "Duck-typing in C++, part 2"
date: 2018-06-06
---

In the previous [part]({% post_url 2018-04-29-duck-typing-in-cpp%}), I showed an example of a duck-typed
function in C++. It bothered me that I have to recompile the code for
every duck (due to template instantiation), and that I have to put the code
in the header file. I'd prefer most of the code (except, say, adaptors) to sit
in .cpp files.

# Introduction

Duck typing comes from the statement, 'If it walks like a duck, if it quacks
like a duck, it's a duck!' Meaning to say: we don't really care what the
underlying object is, as long as it has these specific functions.

The general idea behind duck-typing is getting polymorphism-like behaviour
without having to define a class-hierarchy. In the previous [part]({% post_url 2018-04-29-duck-typing-in-cpp%}),
I introduced the following function (now moved and failing in polyutil.cpp):

```C++
template <typename T>
PolyUtil::Length<T> PolyUtil::circumference(T & p) {
	auto sides = p.sides();
	Length<T> sum = 0;
	for (auto & side : sides) {
		sum += side.length();
	}
	return sum;
}
```

With the header file (polyutil.h):

```C++
#ifndef POLY_H
#define POLY_H

#include <utility>  /* For std::declval */
namespace PolyUtil {
	using std::begin;

	template <typename T>
	using Side = decltype(*begin(std::declval<T>().sides()));

	template <typename T>
	using UnqualifiedSide = typename std::remove_const<
				typename std::remove_reference<Side<T> >::type >::type ;

	template <typename T>
	using Length = decltype(std::declval<UnqualifiedSide<T> >().length());

	template <typename T>
	Length<T> circumference(T &);
};

#endif // POLY_H
```

And the main (in main.cpp, and it's important to have them in separate .cpp
files for this to fail):

```C++
template <typename T> struct Side {
	T m_length;
	Side(T length) : m_length(length) {}
	T length() { return m_length; }
};

struct Poly1 {
	std::vector<Side<int> > m_sides;
	Poly1(std::initializer_list<Side<int> > l) : m_sides(l) {}
	std::vector<Side<int> > sides() { return m_sides; }
};

int main() {
	PolyUtil pu;
	Poly1 p1({1,2,3});
	std::cout << pu.circumference(p1) << std::endl;
	return 0;
}
```

We get the following _linker_ errors:

```
main.cpp:66: undefined reference to `decltype ((((declval<std::remove_const<std::remove_reference<decltype (*(begin((((declval<Poly1>)()).sides)())))>::type>::type>)()).length)()) PolyUtil::circumference<Poly1>(Poly1&)'
```

Why does this happen? Templated code in C++ is compiled for every template
instantiation. As opposed to just once, when the code is encountered.
Template instantiation happens only when the code is actually needed,
e.g., when a templated function is called. In our case, when the function
is called, the function code is no longer available to be compiled.

# Second Solution

In this blog post, I'm going to try and find a solution to the compilation
error above.

I should warn you, it ain't pretty. (Unless you like this sort of thing.)

## High Level

The general idea is to remove the templates from the code that will live in
`polyutil.cpp`. This way, `polyutil.cpp` can be compiled once. The down-side
is that we have to know the parameters in advance.

That's not really a problem in C++. We can use abstract classes with virtual
functions. The function works on the abstract classes, and their implementation
has template parameters.

I'm going to compromise, and say that the implementation of these abstract
classes can be in the header file.  After all, it's
adaptor code, and not the actual logic.

That was from 10,000 ft. Let's take a closer look.

Our original function looked like this:

```C++
template <typename T>
PolyUtil::Length<T> PolyUtil::circumference(T & p) {
	auto sides = p.sides();
	Length<T> sum = 0;
	for (auto & side : sides) {
		sum += side.length();
	}
	return sum;
}
```

We need to wrap `T` and `Length<T>` in an abstract class.  Note that `Length<T>`
is constructed in the function body. That won't do. We can't construct
an abstract class - we don't know its implementation.

We'll solve that by passing an instance of an abstract class containing
`Length<T>` as well. The adaptor code will hide this change. That is, the
adaptor code will retain the original signature, and call the new function
with its modified signature.

So `T` is wrapped inside an abstract class called `TDuck`. `Length<T>` in
`LengthDuck`. Since `TDuck` and `LengthDuck` are abstract, we can get away
with not passing the template parameter.

In the end, the function will look like this:

```C++
void PolyUtil::circumference(PolyUtil::TDuck & p, PolyUtil::LengthDuck & sum) {
	auto sides = p.sides();
	sum.__construct__(0);
	for (auto & side : sides) {
		sum += side.length();
	}
}
```

Note that there are a lot of steps to cover between having a `TDuck` (or `T`)
instance, to actually knowing how to sum the lengths. If we walk through the
function we see that:

```C++
	auto sides = p.sides();
```

We need an abstract class to contain the type of `sides()`. We'll call it
`SidesDuck`.

```C++
	for (auto & side : sides) { /* ... */ }
```

`SidesDuck` needs to have `begin()` and `end()` function members, which return
the iterators. It says
[here]({reference}) that in C++11, these iterators must have the same type.
So `begin()` and `end()` both return an `IterDuckRef` (more on that soon).
Recall from the previous part that `begin(c)` and `end(c)`
call `c.begin()` and `c.end()`, respectively, if they exist.

We'll call the iterator (the result of `sides.begin()` and `sides.end()`)
`IterDuck`. It will have to implement the increment and inequality
operations to adhere to the reference. It'll also have to support
the dereference operator so that we can get a single _side_. The type
of that will be a `SideDuck`. i.e. the iterator returns an instance of
`SideDuck` when dereferenced.

Lastly,
```C++
	sum += side.length();
```

From the loop body, we know that `LengthDuck`'s `+=` operator must accept
the return value of `side.length()`.

I sneakily skipped the following line:

```C++
	sum.__construct__(0);
```

The assignment operator was changed to a specialised construction member function.
This is so that we can easily tell the difference between two cases: 1) The
assignment operator is forwarded to the contained object. 2) The assignment
operator should *create* the contained object. Instead of thinking up heuristics
of what should be done when, it's easier to just be explicit.

There are a few more requirements. For instance, we need to be very careful
with the operations on the underlying objects (`T`, `Length<T>`, etc.). We
don't know what side-effects any operation may have. In C++. This even includes
assignment.

A complete solution would also have correct memory management, i.e. the memory
state will have to be the same as if the original templated function was
called directly.

Lastly, we don't always know if we're going to get copies or references. It's
up to whoever implemented `T` (the template parameter) and its related objects.
C++ gives us the tools to tell these cases apart (e.g., overloading, or
[`is_reference`](http://en.cppreference.com/w/cpp/types/is_reference)), but
we're going to take the easy way out. We'll assume whatever's more comfortable
for us.

I think this covers the high- and mid-level stuff.

## Low Level

So, as we said, we need to define a bunch of abstract classes. Then we need
to provide concrete implementations. The abstract classes can't be templated,
but the concrete implementations will have to be. Therefore, all these
definitions and implementations have to sit in `polyutil.h`.

Let's start with the abstract classes. The first one is `TDuck`.

```C++
struct TDuck {
	virtual ~TDuck() {}
	virtual SidesDuck & sides() = 0;
};
```

As I mentioned above, we have to make sure the memory management is done
right. We can put that logic in the virtual destructor. We need it anyway
since we're using a class hierarchy.

Note that `TDuck` returns a reference to a `SidesDuck` on a call to `sides()`.
`SidesDuck` is an abstract class. We can't move copies of it around, since
the compiler doesn't know how to initialise copies of abstract classes.

This also makes it easier for us to be conservative with operations on the
contained objects. As I said above, we have to be very careful with what we
do to objects given to us from outside.

```C++
struct SidesDuck {
	virtual ~SidesDuck() {}
	virtual IterDuckRef begin() = 0;
	virtual IterDuckRef end() = 0;
};
```

As promised, `SidesDuck` has two member functions: `begin()` and `end()`,
which return an `IterDuckRef`.

Those of you who were paying attention will note that `SidesDuck` returns a
copy of `IterDuckRef`, and not a reference. You can also already guess that
`IterDuckRef` will hold a reference to `IterDuck`. So what's going on? What's
the point?

Basically, the magic around range-based for-loops chooses
`const_iterators` by default. This means that using references will turn
all our iterator references to reference constant objects. That's not
good for us. We want to hide away our magic inside these objects,
whose const-ness has nothing to do with the original objects they are
hiding inside.

This also gives us the small benefit that different calls to `begin`
won't interfere with one another.

Since we return a copy, and not a reference, to `IterDuckRef`, it must
be a concrete class (not abstract).

```C++
struct IterDuckRef {
	IterDuck * m_iterduck;
	IterDuckRef(IterDuck * iterduck) : m_iterduck(iterduck) {}
	~IterDuckRef() { delete m_iterduck; }
	virtual SideDuck & operator*() { return **m_iterduck; }
	virtual bool operator!=(const IterDuckRef & other) {
		return (*m_iterduck) != (*other.m_iterduck);
	}
	virtual IterDuckRef & operator++() {
		++(*m_iterduck);
		return *this;
	}
};
```

As its name suggests, `IterDuckRef` holds a reference to an `IterDuck` (which
holds the actual iterator), and forwards all calls to that reference.

As a final note on this, in C++17, `begin()` and `end()` are
allowed to return different types. To make this very long example shorter,
and allow it to be C++11-compatible, we'll ignore that feature. We can solve
this (allow the C++17 feature and be C++11 compatible) by holding another
pointer in `IterDuckRef`.

This could have also been done by adding another layer of indirection in
`IterDuck`. i.e. have `IterDuck` contain a `BeginIterDuck` and an
`EndIterDuck`, each one holding the iterator returned from `begin` and `end`,
respectively. There's nothing that can't be solved with another layer
of indirection.

```C++
struct IterDuck {
	virtual ~IterDuck() {}
	virtual SideDuck & operator*() = 0;
	virtual bool operator!=(const IterDuck &) = 0;
	virtual IterDuck & operator++() = 0;
};
```

`IterDuck` (and `IterDuckRef`) should behave like iterators. The
increment operator (`operator++`) allows us to move to the next element
in the loop. The dereference operator (`operator*`) allows us to get that
element, and the inequality operator (`operator!=`) allows us to test when
the loop ended. In our case, we know `operator!=` it will only be called
with the result of `sides().end()`, but I tried to keep it general.

```C++
struct SideDuck {
	virtual ~SideDuck() {}
	virtual LengthDuck & length() = 0;
};
```

We only use `SideDuck` to get the length. So `length()` is the only member
function `SideDuck` has.

```C++
struct LengthDuck {
	virtual ~LengthDuck() {}
	virtual void __construct__(int) = 0;
	virtual LengthDuck & operator+=(const LengthDuck&) = 0;
};
```

`LengthDuck` is slightly trickier. If you recall, we changed the signature
to get a reference to an instance of `LengthDuck` from the caller. Calling
`__construct__` should create a new instance, which will be the return value.
Anything else should be forwarded to the contained object.

I think that covers the abstract classes. Now it's time for the implementation.

Recall `TDuck`.

```C++
struct TDuck {
	virtual ~TDuck() {}
	virtual SidesDuck & sides() = 0;
};
```

The implementation is:
```C++
template <typename T>
class TDuckImpl : public TDuck {
	T & m_t;
	SidesDuckImpl<T> * m_sides;
	void clear() {
		delete m_sides; m_sides = nullptr;
	}
public:
	TDuckImpl(T & t) : m_t(t), m_sides(nullptr) {}
	virtual ~TDuckImpl() { clear(); }

	virtual SidesDuckImpl<T> & sides() {
		clear();
		m_sides = new SidesDuckImpl<T>(m_t.sides());
		return *m_sides;
	}
};
```

`TDuckImpl` is a concrete implementation of `TDuck`, with the template
parameter `T`. We allow template parameters, since this code sits in `polyutil.h`,
and is compiled every time it's needed. This is our compromise.

`TDuckImpl` stores the reference to the instance of `T`, the argument to
`circumference` (That's our original function, a long time ago). It also
implements `sides()`, which creates and returns an implementation of
`SidesDuck`. Note that the only operation on the argument is calling
`m_t.sides()`, and only when someone invokes the wrapping `sides()`
method.

We know `sides()` will be called exactly once, and we take advantage of that.
Otherwise, we would have had to handle the memory-management with some more
magic, like [`std::shared_ptr`](http://en.cppreference.com/w/cpp/memory/shared_ptr), or yet another layer of
indirection.

The next abstract class is `SidesDuck`,
```C++
struct SidesDuck {
	virtual ~SidesDuck() {}
	virtual IterDuckRef begin() = 0;
	virtual IterDuckRef end() = 0;
};
```

And here is the implementation.

```C++
template <typename T>
class SidesDuckImpl : public SidesDuck {
	typedef decltype(std::declval<T>().sides()) SidesType;
	SidesType & m_sides;
public:
	SidesDuckImpl<T>(SidesType & sides) : m_sides(sides) {}
	virtual IterDuckRef begin() {
		return IterDuckRef(new IterDuckImpl<T>(::begin(m_sides)));
	}
		
	virtual IterDuckRef end() {
		return IterDuckRef(new IterDuckImpl<T>(::end(m_sides)));
	}
};
```

Pretty trivial. Contain the return value of `sides()` in `m_sides`. Here we
assume `sides()` returns a reference. This is different from the previous part.

`begin()` and `end()` forward to `::begin(...)` and `::end(...)`. This should
take care of adhering to the rules of which `begin`/`end` functions are
called first. At least in the general case where no-one is being smart and
overrides these functions.

`IterDuckRef` is shown above, and all it does is forward to the `IterDuck`
reference. Recall that `IterDuck` is:

```C++
struct IterDuck {
	virtual ~IterDuck() {}
	virtual SideDuck & operator*() = 0;
	virtual bool operator!=(const IterDuck &) = 0;
	virtual IterDuck & operator++() = 0;
};
```

The implementation is:

```C++
template <typename T>
class IterDuckImpl : public IterDuck {
	typedef decltype(begin(std::declval<T>().sides())) IterType;
	IterType m_iter;
	SideDuckImpl<T> * m_iter_deref;
	void clear() {
		delete m_iter_deref;
		m_iter_deref = nullptr;
	}
public:
	IterDuckImpl<T>(IterType iter) : m_iter(iter), m_iter_deref(nullptr) {}
	virtual ~IterDuckImpl<T>() {
		clear();
	}
	virtual SideDuckImpl<T>& operator*() {
		clear();
		m_iter_deref = new SideDuckImpl<T>(*m_iter);
		return *m_iter_deref;
	};
	virtual bool operator!=(const IterDuck & iterduck) {
		const IterDuckImpl<T> & other = static_cast<const IterDuckImpl<T> &>(iterduck);
		return m_iter != other.m_iter;
	}
	virtual IterDuckImpl<T> & operator++() {
		++m_iter;
		return *this;
	}
};
```

Again, mostly trivial. `operator++` and `operator!=` are forwarded to the
contained iterators (`m_iter`).

There is some magic around the dereference
operator (`operator*`). We must return a reference (since `SideDuck` is abstract).
So we carry the burden of memory management, where `IterDuckImpl` maintains a
pointer to the last dereferenced object, and deletes that pointer upon every
call to `operator*`. The client code maintains a single reference to whatever
the iterator returns, so this is safe. But if the client code were to carry
references across loop iterations, we would be in deep trouble.

`IterDuck` returns a `SideDuck`.

```C++
struct SideDuck {
	virtual ~SideDuck() {}
	virtual LengthDuck & length() = 0;
};
```

It's implementation:

```C++
template <typename T>
class SideDuckImpl : public SideDuck {
	typedef decltype(*begin(std::declval<T>().sides())) SideType;
	SideType & m_side;
	LengthDuckImpl<T> * m_length;
	typedef typename std::remove_reference<decltype(*(std::declval<LengthDuckImpl<T> >().m_length))>::type LengthType;
	void clear_length() {
		delete m_length;
		m_length = nullptr;
	}
public:
	SideDuckImpl<T>(SideType & side) : m_side(side), m_length(nullptr) {}
	virtual ~SideDuckImpl<T>() {
		clear_length();
	}
	virtual LengthDuckImpl<T> & length() {
		clear_length();
		LengthType * l = new LengthType(m_side.length());
		m_length = new LengthDuckImpl<T>(l);
		return *m_length;
	}
};
```

Here we assume that `m_side.length()` returns a non-reference. In the
original code, the addition-assignment would have probably called the
copy constructor.  I say probably, since, if `Length<T>`'s `operator+=`
expected a reference, and `side.length()` returned a reference, the copy
constructor would be avoided.

To mimic this behaviour, we create a new `LengthType` object, and initialise
it with the copy constructor. We don't really have a choice. If `length()`
returns a copy, the compiler won't let us save a reference. That copy has to
be stored somewhere.

As I said above, we could differentiate these cases with e.g.
[`std::is_reference`](http://en.cppreference.com/w/cpp/types/is_reference),
but we're taking the easy way out. This post has grown long enough.

Lastly, we put everything together. An adaptor function, with the original
signature, looks like this:

```C++
template <typename T>
Length<T>
circumference(T & p) {
	LengthDuckImpl<T> result;
	TDuckImpl<T> duck(p);
	circumference(duck, result);
	return *result.m_length;
}
```

All it does is create a duck wrapper for `p` (a `TDuck` containing a `T`),
create a duck wrapper for the result (a `LengthDuck` containing a `Length<T>`),
and calls the non-template implementation.

# Conclusion

Let's recap on what we have done. We've rewritten our duck-typed function
to accept a `TDuck` abstract class, instead of the actual duck. Note that
the entire function is without templates. Otherwise, we'd run into the same
issue.

We have created an abstract class to wrap every type of object we reference
throughout the new function.

We then created implementations for each of these abstract classes. The
implementations have template parameters.

Lastly, we wrote an adaptor function accepting the original signature,
with the original types. The adaptor function wraps the arguments and
return values, and then calls the rewritten function.

All in all, for 6 lines of code, we wrote approximately 200 lines of adaptor
code.

This *still* won't do.

I'm going to stop here. I wanted to have the 200 lines of adaptor code written
automatically, maybe with a [_clang++_](https://clang.llvm.org/) extension.
That includes quite a bit of research, so this will take a while.
