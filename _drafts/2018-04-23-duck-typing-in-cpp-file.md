---
layout: post
title: "Duck-typing in C++, part 2"
date: 2018-04-23
---

In the previous [part]({reference}), I showed an example of a duck-typed
function in C++. It bothered me that I have to recompile the code for
every duck (read template instantiation), and that I have to put the code
in the header file. I'd prefer most code (except, say, adaptors) to sit
in .cpp files.

# Introduction

Duck typing comes from the statement, 'If it walks like a duck, if it quacks
like a duck, it's a duck!' Meaning to say: we don't really care what the
underlying object is, as long as it has these specific functions.

The general idea behind duck-typing is getting polymorphism-like behaviour
without having to define a class-hierarchy. In the previous [part]({reference}),
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

Whith the header file (polyutil.h):

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

So the intuitive thing to do is to somehow remove templates from the
real code, and then it can be compiled once. Any templated code has to
be compiled in the header file. There is no way around it, since only
then do you know how to instantiate the templates.

So how would our code look without templates? Here's one option:

```C++
PolyUtil::LengthDuck PolyUtil::circumference(TDuck & p) {
	auto sides = p.sides();
	LengthDuck sum = 0;
	for (auto & side : sides) {
		sum += side.length();
	}
	return sum;
}
```

It looks promising, until you realise you have to define `TDuck`, `LengthDuck`,
and everything that is needed to get from `TDuck` to `LengthDuck`.

I foresee that LengthDuck might give us an issue, so let's go ahead and change
the signature so that I can give the function a LengthDuck reference.

```C++
void PolyUtil::circumference(TDuck & p, PolyUtil::LengthDuck & sum) {
	auto sides = p.sides();
	sum = 0;
	for (auto & side : sides) {
		sum += side.length();
	}
	return sum;
}
```

{LengthDuck (composite-) contains Length<T>, which is initialized with `operator=`.

Now we can define our classes, `TDuck` and `LengthDuck`.

```C++
struct TDuck {
	virtual SidesDuck sides() = 0;
};
```

```C++
struct LengthDuck {
	virtual LengthDuck & operator=(int) = 0;
	virtual LengthDuck & operator+=(const LengthDuck&) = 0;
};
```

Oy. Now we need to define SidesDuck.

```C++
struct SidesDuck {
	virtual IterDuck begin() = 0;
	virtual IterDuck end() = 0;
};
```

And now `IterDuck` (It never ends).

```C++
struct IterDuck {
	virtual LengthDuck operator*() = 0;
	virtual bool operator==(const IterDuck &) = 0;
	virtual bool operator!=(const IterDuck &) = 0;
};
```

Now note that all these classes are abstract. We need to inherit them. I'm going
to compromise, and say that the inherited classes' code can be in the header file.
After all, it's adaptor code, and not the actual logic. It will probably look
like this:

```C++
template <typename T>
class TDuckImpl : public TDuck {
	T & m_t;
public:
	TDuckImpl(T & t) : m_t(t) {}

	virtual SidesDuckImpl<T> sides() {
		return T.sides();
	}
};
```

```C++
template <typename T>
class SidesDuckImpl : public SidesDuck {
	typedef typename decltype(std::declval<T>().sides()) SidesType;
	SidesType & m_sides;
public:
	SidesDuckImpl<T>(SidesType & sides) : m_sides(sides) {}
	virtual IterDuckImpl<T> begin() {
		return begin(m_sides);
	}
		
	virtual IterDuck end() {
		return end(m_sides);
	}
};
```

```C++
template <typename T>
class IterDuckImpl : public IterDuck {
	typedef decltype(begin(std::declval<T>().sides())) IterType;
	IterType & m_iter;
public:
	LengthDuckImpl<T>(IterType & iter) : m_iter(iter) {}
	virtual LengthDuckImpl<T> operator*() {
		return *m_iter;
	};
	virtual bool operator==(const IterDuck & iterduck) {
		if (IterDuckImpl<T> & other = dynamic_cast<IterDuckImpl<T> &>(iterduck)) {
			return m_iter == other.m_iter;
		}
		return false;
	}
	virtual bool operator==(const IterDuck & iterduck) {
		if (IterDuckImpl<T> & other = dynamic_cast<IterDuckImpl<T> &>(iterduck)) {
			return m_iter != other.m_iter;
		}
		return false;
	}
};
```

{What if begin and end don't return the same type?}

```C++
template <typename T>
class LengthDuckImpl<T> : public LengthDuck {
	template <typename T>
	using ConstSide = decltype(*begin(std::declval<T>().sides()));

	template <typename T>
	using Side = typename std::remove_const<typename std::remove_reference<ConstSide<T> >::type >::type ;

	template <typename T>
	using Length = decltype(std::declval<Side<T> >().length());

	typedef typename Length<T> LengthType;
	LengthType * m_length;
public:
	LengthDuckImpl<T>() : m_length(nullptr) {}
	~LengthDuckImpl<T>() {
		delete m_length;
		m_length = nullptr;
	}

	LengthDuckImpl<T> & operator=(int i) {
		m_length = new LengthType(i);
	}

	LengthDuckImpl<T> & operator+=(const LengthDuck & other) {
		// ...
			m_length += other.m_length;
		// ...
		// Exception?
	}

}
```

{virtual is important} {note no templates}

# Conclusion

