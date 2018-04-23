---
layout: post
title: "Duck-typing in C++"
date: 2018-04-16
---

<abstract>

Dynamically typed languages, such as Python, or Lua, have duck-typing. What
about C++? In this post, I will try to bring duck-typing into C++.

# Introduction

Duck typing comes from the statement, 'If it walks like a duck, if it quacks
like a duck, it's a duck!' Meaning to say: we don't really care what the
underlying object is, as long as it has these specific functions.

The general idea behind duck-typing is getting polymorphism-like behaviour
without having to define a class-hierarchy. If we take, for instance, this
code block from python:

```Python
def circumference(poly):
	sides = poly.sides()
	sum = 0
	for side in sides:
		sum += side.length()
	return sum
```

In this code, we don't actually care about the type of `poly`. All we care
about is that it has a method called `sides` that accepts no arguments.
The return value of `sides` is also a duck: It only has to be iterable. It
doesn't matter if it's a list, tuple, generator, or a pygmy marmoset handing
out objects.

Even every object in the iterator is a duck. All we know about it is that
it has a method called `length` that accepts no arguments. Since the summation
expects numbers, `length` must return a number.

Now how would we do that in C++?

Suppose we have the following function:

```C++
int circumference(Poly & p) {
	std::list<Side> sides = p.sides();
	int sum = 0;
	for (Side & side : sides) {
		sum += side.length();
	}
	return sum;
}
```

This is all very nice, but what if I have a polygon that does not inherit from
`Poly`? It has the method `sides` that behaves correctly, so where's the
problem?

# (Not so) Trivial Solution

Many of you have probably scoffed and said 'Well, obviously, use templates!'
Well shame on you. Templates are never obvious. And this is no exception!

But that's probably the best way to go about it, so let's try.

For those of you lucky enough to have avoided templates, here is a
[reference](http://en.cppreference.com/w/cpp/language/templates). I'm willing
to write a primer if anyone asks, but it'll take to long to get into now.

```C++
using std::begin;

template <typename T>
decltype(std::declval<typename std::remove_const<typename std::remove_reference<decltype(*begin(std::declval<T>().sides())) >::type >::type >().length())
circumference(T & p) {
	auto sides = p.sides();
	decltype(std::declval<typename std::remove_const<typename std::remove_reference<decltype(*begin(std::declval<T>().sides())) >::type >::type >().length()) sum = 0;
	for (auto & side : sides) {
		sum += side.length();
	}
	return sum;
}
```

I want to take a second to talk about some of the magic in this code. It is
all C++11 compatible. I'll start by listing and explaining the building-blocks,
and then go through how each is used above.

*auto* ([reference](http://en.cppreference.com/w/cpp/language/auto))
should already be familiar to most C++ developers. It is a handy
shortcut. Rather than knowing the type of the return value of some
function, the compiler does it for you. The compiler already knows the
return type of `p.sides()`, so declaring `auto sides = p.sides()` gives
the local variable `sides` the correct type.

*decltype*
([reference](http://en.cppreference.com/w/cpp/language/decltype)) takes
an expression, and infers the type of that expression. It is important
to note that the expression isn't actually evaluated. This is important
since you don't have to worry about side effects (e.g. `begin()` can
only be called once). So `decltype(sides.begin()->length())` means 'the
return value of `length`, called on the dereferenced return value of
`begin`, called on `sides`'.

*std::declval*
([reference](http://en.cppreference.com/w/cpp/utility/declval)) returns a
reference to an instance of the template parameter. So `std::declval<T>()`
returns a reference to an instance of type `T`. syntactically, `T & t =
std::declval<T>()`, but! Calls to `std::declval` cannot be evaluated.
It causes a compiler error (e.g. a `static_assert` on g++ 7.2.1). That's
also why it's important for the `decltype` argument to be unevaluated.

*std::remove_const*
([reference](http://en.cppreference.com/w/cpp/types/remove_cv))
removes the topmost `const` of a *type* (not a value). If the given
type is not a `const`, it is returned unchanged. The static member
`type` returns the new type. So `std::remove_const<const int>::type`
is the same as `int`. Note that it removes the *topmost* `const`. So
`std::remove_const<const int *>::type` is still `const int *`, since the
`const` refers to the dereferenced value, and not the pointer itself. On
the other hand, `std::remove_const<int * const>::type` is `int *`, since
the `const` in `int * const` refers to the pointer itself, and is in
fact the topmost `const`. This is especially relevant for references,
which are always constant. That's why `std::remove_reference` is needed.

*std::remove_reference*
([reference](http://en.cppreference.com/w/cpp/types/remove_reference))
makes the given type a non-reference. This means that given a type `T &`
(or `T &&`), `std::remove_reference` can be used to obtain `T`. Like
`std::remove_const`, the static member `type` returns the new type. If
the given type is not a reference, it is returned unchanged.

*typename* tells the C++ compiler that what follows is a type, and not
to mistake it for anything else. While it is obvious in this case (and
indeed the compiler error said specifically to add `typename`) there
are some cases where `typename` appears to remove ambiguity. I refer you
[here](http://pages.cs.wisc.edu/~driscoll/typename.html) for an example,
and more info.

*begin* returns an iterator pointing to the first element, or,
if the container is empty, just beyond the last element. I want to
take a step back and talk about range-based for loops in C++ first
([reference](http://en.cppreference.com/w/cpp/language/range-for)).

In C++11, the following syntax for range-based for loops was added:

```C++
for (type & element : container) {
	// Do something
}
```

Which is, loosely speaking, shorthand for saying "iterate over all elements in
container." Loosely speaking, since I refuse to go into the quadzillion edge
cases. So comes the question, how do we iterate over all the elements?

So there are three rules:

1. If `container` is an array type, the iteration is over all elements in the
array. The array size must be known.

2. If `container` has the members `begin()` and `end()` (even if they are private
and the types don't match), the iteration starts at `container.begin()`, and increments the
results as long as it's different from `container.end()`. Note that the types
may be different, as long as the inequality operator is defined. So, possibly,
the code becomes equivalent to: (Since C++17, `container.end()` is evaluated once)

```C++
// for (auto & element : container)
for (auto it = container.begin(); it != container.end(); it++) {
	auto & element = *it;
	// Do something
}
```

3. If the functions `begin` and `end` exist and are applicable for the type
of `container`. So, possibly, 
the code becomes equivalent to: (Since C++17, `container.end()` is evaluated once)

```C++
// for (auto & element : container)
for (auto it = begin(container); it != end(container); it++) {
	auto & element = *it;
	// Do something
}
```

`std::begin` (and `std::end`) are defined to return the above rules, i.e. for
arrays, `std::begin` returns a pointer to the beginning of the array. For
containers containing the `begin` and `end` members, these are called.
`begin`  (note the unqualified version) is expected to be overloaded for
custom classes that do not have `begin` and `end` members, but can still
be iterated. In order for `begin` to overload `std::begin`, we use
`using std::begin` to bring `std::begin` to the local scope. In other words,
`using std::begin` means that you can call `begin` directly, without qualifying
it. This way, you will hit one of the overloads (either `begin` if it exists,
and `std::begin` if it doesn't). This raises an interesting question: What if
I define a `begin` function specifically for `list`. Will it override `list.begin()`
in a range-based for loop?

So, back to the point at hand, using the function `begin` gives me a unified,
and hopefully accurate enough, way to get the type of the iterator.

So, the very long line:

```C++
decltype(std::declval<typename std::remove_const<typename std::remove_reference<decltype(*begin(std::declval<T>().sides())) >::type >::type >().length())
```

Is completely unreadable. While testing this, I broke it down intto bite-sized
aliases:

```C++
template <typename T>
using Side = decltype(*begin(std::declval<T>().sides()));

template <typename T>
using UnqualifiedSide = typename std::remove_const<
			typename std::remove_reference<Side<T> >::type >::type ;

template <typename T>
using Length = decltype(std::declval<UnqualifiedSide<T> >().length());
```

An _alias template_
([reference](http://en.cppreference.com/w/cpp/language/type_alias))
allows you to write long template expression as a short
template. In the example above, `Side<T>` is equivalent to
`decltype(*begin(std::declval<T>().sides()))`. `Side<T>` is just a
shorter, more readable alias. I find the example clearer than the
explanation.

First of all, we define `Side<T>`, which represends the type of objects
returned by `sides()`. `sides()` can return a list, or vector, or any
range-iterable object. `Side<T>` it the contained type of that object.
In fact, you could replace the use of `auto` with:

```C++
	for (Side<T> side : sides) {
		sum += side.length();
	}
```

And `Side<T>` already defines a reference, and a `const`. We'll get to that
in a minute. But first, how do we know what type is actually returned?

`T` is the type of the argument passed to the function `circumference`. Since
`T` is a type and not an instance or reference, we call `std::declval` to get
a (fictitious) reference to a `T`. On that reference, we call `sides()`. This
returns a range-iterable object. The function `begin` returns an iterator to
the first element in the object returned from `sides()`. Dereferencing this
iterator `(*...)` returns the first element itself. Lastly, we wrap everything
in `decltype` to get the type, and make sure this code is never executed.

If this code was executed, compilation would fail. You can't execute an
`std::declval`. This code also assumes that the object returned from `sides()`
is not empty. But since we're only doing a syntactic walk through the types,
it's not really a problem.

Now somewhere along the line, `begin` decided to return a `const_iterator`
instead of an iterator. So the type of dereferenced element is `const`
(this is `Size<T>`). If the `length()` method is not defined as `const`
as well (say, the blog post writer was lazy), the compiler is a bit
cross. So we have to use `std::remove_const` to fix that.

Only issue is, that `Side<T>` is also a reference. Calling `std::remove_const`
on a reference is a no-op, since references are always constant, and
`std::remove_const` removes only the top-level `const`. So we have to call
`std::remove_reference` as well.

This is what `UnqualifiedSide<T>` does. It takes `Side<T>`, and removes its
referenceness (makes it a non-reference, i.e. converts `U&` to `U`). It then
makes it non-constant (i.e. converts `const U` to `U`). `::type` is the way
to get the type back from `remove_*`. We pepper the template with `typename`s
so that the compiler knows these are types. At last, we have the type of
element we iterate over.

We can now call `length()` on it. Except it's a type, and we need an instance
(or a reference). So `std::declval` again, and then we call `length()`.
Lastly, we call `decltype`, since we want the type of the length, and not
the actual length of this non-existent object.

Tra-daa! `Length<T>`, our return value, and type of summation.

Despite the complexity, this works even better than the Python version,
since it works on anything that implements the add operator, and accepts
0 in the constructor (which we can solve with a parameter with a default
value). I can definitely construct something that is initialised with 0,
but can have strings added to it! (If you don't get it, try running '0 +
"hi"' in python.)

## Example Programme

So let's build a small example programme for this. We'll start by defining
a simple `Side` class.

```C++
template <typename T> struct Side {
	T m_length;
	Side(T length) : m_length(length) {}
	T length() { return m_length; }
};
```

Basically, it is a container that returns it's value via the `length` function.

Now we'll define a polygon class:

```C++
struct Poly1 {
	std::vector<Side<int> > m_sides;
	Poly1(std::initializer_list<Side<int> > l) : m_sides(l) {}
	std::vector<Side<int> > sides() { return m_sides; }
};
```

Again, basically a container that returns the value given to the constructor.

And a simple `main`:

```C++
int main() {
	Poly1 p1({1,2,3});
	std::cout << PolyUtil::circumference(p1) << std::endl;
	return 0;
}
```

You can also go nuts and define a `Poly2` class with a list instead of a vector,
float instead of an int, and see how wonderfully it works.

## Problem

So what's the problem?

The code is a bit clunky. All these *decltype*s and *std::declval*s may be
a bit daunting to developers who aren't familiar with them, but you can send
them this way to help them get settled in.

My big issue is this: what if I want to hide the logic in a .cpp file? This
code works well if everything is in one file. But in a real project, different
classes and utilities are in different files. Sometimes, even in different
libraries. And sometimes, I don't want my code to be recompiled, since perhaps
it's proprietary.

So let's experiment. An .h file (say `polyutil.h`):

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

A .cpp library file (say `polyutil.cpp`):

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

And our main:

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

And flop! We get the following _linker_ errors:

```
main.cpp:66: undefined reference to `decltype ((((declval<std::remove_const<std::remove_reference<decltype (*(begin((((declval<Poly1>)()).sides)())))>::type>::type>)()).length)()) PolyUtil::circumference<Poly1>(Poly1&)'
```

Why does this happen? Templated code in C++ is compiled for every template
instantiation. As opposed to just once, when the code is encountered.
Template instantiation happens only when the code is actually needed,
e.g., when a templated function is called.

When the compiler instantiates `T` to be `Poly1`, as in our example,
it looks for the compiled code. But it was not needed when compiling
`poly.cpp`, so that code doesn't exist.

In fact, since `poly.cpp` only contains the method definition, and nothing
else (no calls), it doesn't contain any `circumference` symbols. (You
can try and compile the code, and run `nm`, and review the output if
you don't believe me. In fact, you shouldn't believe me, and make sure
I'm not fibbing).

# Conclusion

Basically, we saw that C++ allows you to write duck-typed code. It has two
main issues: It is clunky, and it can't be hidden in a .cpp file.

I find the code clarity part a lesser issue. You can break up complex type
inference code into small aliases. Having to put the code in the header file
bothers me a bit more.

The original plan was to solve this problem, and maybe propose a solution for
the clunkiness. However, just the introduction took a
while, so I'm going to have to break here.

