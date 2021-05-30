---
layout: post
title: "Monte Carlo Method for Integration"
date: 2021-05-30
use_math: true
---

This is a quick introdution to the Monte Carlo method for integration. I
develop it here from scratch relying on known statistical properties. To tie
this in to programming, I also provide a short, simple implementation in Python.

# Introduction

Integration is hard. In some cases, the value of the integral cannot be found
analytically. There are numerical methods for solving integrals. Some are
simple, and some are complicated.

In this post, I will introduce such an method. In case you didn't guess already,
the method relies on the Monte Carlo method. This means we take many random
samples, and try to gather information about the population from which we
sample.

# Problem Description

Suppose we want to find the value for the integral $\int
f\left(x\right)\mathrm{d}x $ in the range $x_{min}$ and $x_{max}$. The only
thing we assume is that $f\left(x\right)$ is defined for every value in this
range. For the purpose of simplicity, we assume the range is inclusive.

Since we are using a numerical method, there is going to be some level of
innaccuracy. So there should be a way to get a measure of the error.

# In Theory

In this section we'll explore the theory behind this method. Monte Carlo
methods are statistics based methods.  So here we will go a bit over the
mathematics of why this works. I'll try to take it slowly.

First, let's write down the value we are trying to calculate:

\\(
  \int_{x_{min}}^{x_{max}}f\left(x\right)\mathrm{d}x
\\)

In this case, we assume $x_{min}$ and $x_{max}$ are known. We also
assume that we have $f\left(x\right)$. We don't need to analyse $f\left(x\right)$,
just call it a bunch of times and get the values. We also assume that
$f\left(x\right)$ has a finite value for any $x$ in the range
$\left[x_{min},x_{max}\right]$.

Now the magic starts. Let's assume that we have a continuous random
variable $X$, and that the probability of this $X$ is uniform over
the range $\left[x_{min},x_{max}\right]$.

\\(
X \sim U\left(x_{min},x_{max}\right) \\\\\\\\
P_{x}\left(X\leq x\right) =\begin{cases}
0 & x\<x_{min}\\\\\\\\
\frac{x-x_{min}}{x_{max}-x_{min}} & x_{min}\leq x\leq x_{max}\\\\\\\\
1 & x_{max}\<x
\end{cases}\\\\\\\\
p_{X}\left(x\right) =\frac{1}{x_{max}-x_{min}}
\\)

Here $p_{X}\left(x\right)$ is the probability density function (PDF)
of $X$. Intuitively, it tells us the probability of $X=x$. Note,
however, that due to the magic of infinitesimal numbers, $P_{x}\left(X=x\right)=0$,
so technically it tells us the probability $P_{x}\left(x\leq X\leq x+\mathrm{d}x\right)$.

We can define a random variable $Y$ such that $Y=f\left(X\right)$.
Let's see if we can find its PDF. There is a method of coordinate change.
The method for moving from one random variable to another, when they
are related by a well behaved function, is:

\\(
p_{Y}\left(y\right)=\sum p_{X}\left(f^{-1}\left(Y\right)\right)\left|\frac{d}{dy}f^{-1}\left(Y\right)\right|
\\)

Only we don't have $f^{-1}\left(y\right)$. Also we have this weird summation
term because no one promises us $f^{-1}\left(x\right)$ is a function, and a
single value of $y$ may lead back to many values of $x$. In other words, many
values of $x$ may return the same value in $f\left(x\right)$.

So let's try the other direction.

\\(
p_{X}\left(x\right)=p_{Y}\left(f\left(x\right)\right)\left|\frac{d}{dx}f\left(x\right)\right|
\\)

This looks promising. That differential is a bit worrying, since we
don't have it. But let's keep going and see what happens. Let's try
and find the expected value of $Y$. The expected value is a form
of statistical mean:
\\(
EY=\int_{-\infty}^{\infty}y\cdot p_{Y}\left(y\right)\mathrm{d}y
\\)

I don't know how to solve this directly, but we can transform back
to $x$:
\\(
EY=\int_{-\infty}^{\infty}f\left(x\right)p_{Y}\left(f\left(x\right)\right)\left|\frac{\mathrm{d}}{\mathrm{d}x}f\left(x\right)\right|\mathrm{d}x
\\)

We may have cheated a bit with the range, but under the assumption
that $f\left(x\right)$ behaves reasonably well, we can do that. Note
also that $p_{Y}\left(y\right)=0$ for values of $y$ that cannot
be reached via $f\left(x\right)$.

Now, that term in the middle looks familiar. Let's replace it:

\\(
EY=\int_{-\infty}^{\infty}f\left(x\right)p_{X}\left(x\right)\mathrm{d}x
\\)

Substituting for $p_{X}\left(x\right)$ from above, we get:
\\(
EY=\frac{1}{x_{max}-x_{min}}\int_{x_{min}}^{x_{max}}f\left(x\right)\mathrm{d}x
\\)

This looks promising. How does it help?

In statistics, we have the Central Limit Theorem. This means, that if we
take enough samples of $Y$, and take their mean, eventually we will get
close to $EY$. How close? That depends on the distribution.  However, if we
have $n$ samples, chances are we won't be much more than $\sigma/\sqrt{n}$
away from it. $\sigma$ is the standard deviation of $Y$, which intuitively
tells us how wide the range of $Y$ is, and what are the chances we'll see
values that are very different from each other.

Suppose we have $n$ samples of $Y$. We'll call them $y_{i}$. We'll
define $\bar{y}$ as the mean, and the our approximation will give
us:
\\(
\bar{y}=\frac{1}{n}\sum_{i=1}^{n}y_{i}\approx EY=\frac{1}{x_{max}-x_{min}}\int_{x_{min}}^{x_{max}}f\left(x\right)\mathrm{d}x
\\)

We will get to how close in a moment. First let's talk about how to
get these samples? We know how to get samples of $X$, since $X$
is uniform. So, given $n$ samples of $X$ (which we'll call $x_{i}$),
we can say that $y_{i}=f\left(x_{i}\right)$. This is why we only
need to assume that we can call $f\left(x\right)$ a bunch of times.
That's all we ever use it for.

So now, let's discuss the error. As we said, our standard deviation
is $\sigma/\sqrt{n}$. But we don't know $\sigma$, which is a property
of $p_{Y}\left(y\right)$, and therefore of $f\left(x\right)$. The
Central Limit Theorem gives us that

\\(
S^{2} =\frac{1}{n-1}\sum_{i=1}^{n}\left(y_{i}-\bar{y}\right)^{2}\approx\sigma^{2}
\\)

Note that this is a bit of a simplification. But it's good enough
for our purposes. So we know that our standard deviation from $EY$
is $S/\sqrt{n}$. So we can say:

\\(
\int_{x_{min}}^{x_{max}}f\left(x\right)\mathrm{d}x \approx\left(x_{max}-x_{min}\right)\cdot\bar{y} \\\\\\\\
\varepsilon \approx \frac{S\cdot\left(x_{max}-x_{min}\right)}{\sqrt{n}}
\\)

We will take $\varepsilon$ as a measure for our error. Note that this isn't
the maximum possible error. Rather, it says that we have a relatively good
chance (a bit less than 70%) to be closer to the real value than the standard
deviation. We have a very good chance (more than 95%) to be closer to the
real value than $2\varepsilon$. All this provided that $n$ is big enough (say,
$n\geq1000$).

# In Practice

Now that we went through the theory, we realise that all we need it to take
some samples, and calculate the mean.

We can use these samples to get the standard deviation too.

The following Python code covers it. You need `numpy` to get it working.

```py
import numpy as np
import numpy.random

# Initialisation code for the random number generator.
rng = np.random.default_rng()


def monte_carlo_definite_integral(f, low, high, n):
    """
                                   high
    Calculate an approximation to ∫    f(x)dx.
                                   low

    Returns the approximated value, and its standard deviation.
    """
    xs = rng.uniform(low, high, n)
    f_v = np.vectorize(f)
    ys = f_v(xs)
    mean = np.mean(ys)
    std = np.std(ys, ddof=1)
    range_size = high - low
    return range_size*mean, range_size*std/np.sqrt(n)
```

Note that `np.mean(ys)` is exactly $\bar{y}$, and `np.std(ys, ddof=1)` is
exactly $S$. It's almost as if someone else knew about this.

We can even see that it's working:

```
>>> monte_carlo_definite_integral(lambda x: x, 0, 2, 1000)
(1.9757557552049256, 0.03601030396241144)  # Expected value is 2

>>> monte_carlo_definite_integral(lambda x: x*x, 0, 3, 10000)
(9.019680243940124, 0.08055872243658405)  # Expected value is 9

>>> import math
>>> monte_carlo_definite_integral(math.exp, 0, 1, 10000)
(1.7156674208045377, 0.0048931664797902605)  # Expected value: 2.7182-1=1.7182
```

We can see that sometimes the value is off by a bit more than $\varepsilon$,
the second return value.  As we said, $\varepsilon$ shows that there's a
good chance you won't be off by more than that. But it's never 100%, since
this is a statistical method.

# Conclusion

We have seen how to use the Monte Carlo method to estimate single variate
integrals. We have also seen how to estimate how bad is our estimation.
The extension to multi-variate, while not trivial, can be done.

I find it really cool that a complex mathematical process leads to a practical
solution that is less than ten lines long. Basically, after two pages of maths
and the Central Limit Theorem (which is a feat in itself), the result is seven
lines of code, and some boilerplate.
