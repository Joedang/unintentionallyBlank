---
title: "Dirac Delta Expansion"
date: 2024-08-01
layout: post
oneLiner: "Orthogonal functions are pretty neat."
splash: 
    src: /img/beachSplash.jpg
    alt: "an image I took in Seaside, OR back in 2011"
mathjax: true
---

# Intro
A few years ago, I was reading the 7th edition of Arfken\'s Mathematical Methods for Physicists.
I really liked [example 5.1.7](https://archive.org/details/Mathematical_Methods_for_Physicists/page/n274/mode/1up?view=theater),
and wondered about using for *other* sets of orthonormal basis functions.
Instead of expanding a function into a series of sine/cosine waves, as with a Fourier expansion,
you could apply the same formula to *any* set of orthonormal functions.
For example, you could expand a function using [Legendre Polynomials](https://en.wikipedia.org/wiki/Legendre_polynomials).

I wrote a Jupyter notebook (using R instead of Python) to play around with the ideas and put stuff in a more presentable format.
Unfortunately, Python sucks, especially for stuff you want to sit around for years and still work when you come back to it.
I'm pretty sure nobody has ever looked at that Jupyter notebook except me.
So, I'm gonna try and put it in a more easily accessible form here.

## Background
Okay, I think I need to try and explain the basics of function decomposition, because the people who don't already know about it are gonna be confused and the people who do... well you can skip down a few paragraphs.
Either way, you're gonna need a rough understanding of calculus.

### Function Decomposition
For engineering purposes, sometimes you need to be able to approximate a function.
Maybe it's not practical to compute the function "the right way".
So, you can add together a bunch of simple functions.
The more simple functions you use, the more accurate your approximation.

This concept isn't just practical, it also helps with math.
Sometimes functions are expressed in inconvenient ways, and it would be nice to represent them differently.
If you add together an *infinite* number of simple functions, you no longer have a mere approximation,
you have an *exactly equivalent* representation of your original function.
Now, since you're representing your function with a different expression,
you have different options available for manipulating that expression,
which *can* make the math easier/possible to solve.
(An infinite process might sound silly from an engineering perspective,
but remember that it's just a few extra symbols when you're doing math.)
It's also really important to note that,
depending on the function you're approximating and the type of decomposition you use,
the new representation may only be equivalent to the original *over a particular domain*, but that won't be relevant here.

Two types of function decomposition you're probably already familiar with are 
[Taylor series expansion](https://en.wikipedia.org/wiki/Taylor_series) 
and [Fourier series expansion](https://en.wikipedia.org/wiki/Fourier_series).
Those wiki pages provide good visualizations of how those expansions work, so I'm not going to bother making my own.
You may also be familiar with the term "Fast Fourier Transform" (FFT),
which is just a clever algorithm for computing a Fourier series expansion.
Personally, I thought Taylor and Fourier expansions made a lot of conceptual sense when I first learned about them.
"Taylor just makes all the derivatives match, and Fourier... *\*waves hands\** is just a weird Taylor expansion." so I thought.
However, most of my professors and textbooks neglected to explain why Fourier expansion is so special.

### Orthogonal Functions
A "convolution" is just multiplying two functions together, and integrating:  

$$\int f(x) g(x) dx$$

Usually, the bounds of the integral are important, and there's some coefficient to make the math work out cleaner.
If the functions being convolved are complex, then one of them needs to be conjugated.
All that tends to be boilerplate.
So, to make things more readable and less error-prone, it's common (especially in physics) to use 
[Dirac's "bra-ket" notation](https://en.wikipedia.org/wiki/Bra%E2%80%93ket_notation):  

$$\left< f | g \right> \equiv C \int\limits_{x_0}^{x_f} f^*(x) g(x) dx$$

Let's define a function:

$$ \phi_n(x)=\sqrt{2}\sin(n \pi x) $$

$\sqrt{2}$ and $\pi$ are just there to make the math cleaner.
Basically, they fudge things so that $C=1$, $x_0=0$, and $x_f=1$.
$n$ is just an unsigned integer, which we'll use to create a whole series of functions.

If we convolve two $\phi_n$ functions together, and they have different values of $n$, something interesting happens.

$$
\left< \phi_n | \phi_m \right> = 
\begin{cases}
1,& \text{if } n=m \\
0,& \text{if } n\neq m
\end{cases}
$$

The proof of this isn't particularly enlightening, but it's a decent exercise.
(You get to work out some integrals and grind through a bit of logic.)
The important part is that this is very reminiscent of a "dot product" (aka inner product).
For example, say you have a vector space, with "orthonormal" (orthogonal and normal) basis vectors $\hat x_i$.
So, $\hat x_1$ points in the x direction, $\hat x_2$ points in the y direction, et cetera (they are "orthogonal"),
and they all have length 1 (they are "normal").
Dot products between these $\hat x_i$ basis vectors follow essentially the same relationship as
convolutions between the $\phi_n$ functions:

$$
\hat x_i \cdot \hat x_j = 
\begin{cases}
1,& \text{if } i=j \\
0,& \text{if } i\neq j
\end{cases}
$$

So, that's where the term "orthogonal function" comes from in math.
(Note that it means something completely different in computer science!)

### The Dirac Delta Function
The easy definition of the Dirac delta is:

$$
\int\limits_{x_0}^{x_f} \delta(x) dx = 
\begin{cases}
 1,& \text{if } x_0 < 0 < x_f \\
-1,& \text{if } x_0 > 0 > x_f \\
0,& \text{otherwise} 
\end{cases}
$$

So, if you happen to integrate over $x=0$, the result is $1$.
(The case where it's $-1$ is just a natural consequence of $x_0 > x_f$. Flipping the bounds of an integral flips its sign.)
If you *don't* happen to integrate over $x=0$, the result is $0$.
I like to think of it as the limit of a normal distribution as the standard deviation tends to 0.
So, its width gets squished to zero, and its height gets squished to $\infty$, but the integral remains $=1$.
The [Wikipedia article](https://en.wikipedia.org/wiki/Dirac_delta_function) has a nice animation of this.

This function is really useful for physics.
It will also be really convenient for the examples later on, because of its ability to "pluck out" values from an integral.
What I mean by that is, because of the properties of $\delta(x)$, you can basically get rid of the integral and replace it with the value of whatever is multiplied with $\delta(0)$:

$$
\int\limits_{-\infty}^{\infty} \delta(x)f(x) dx =  f(0)
$$

### Out of Scope
There are a few details I skipped over.
The $\phi_n$ functions can't provide good approximations of functions that *don't* pass through $(0,0)$ or $(1,0)$,
because $\phi_n$ *always* passes through those points.
They also won't be able to approximate most functions if we change the domain.
(This is why Fourier expansions include cosine terms and a constant term.)
I'm crafting my examples to avoid these issues, but they'd matter for solving real problems.

## Explaining Arfken's Example
One way of looking at it is that it's just a partial Fourier expansion.
Another way of looking at it is as an inner product (a dot product).
In this way of looking at things, our "vectors" are functions.
We can take the "dot product" between them with an integral:  
$\left< f | g \right> = \int\limits_0^1 f(x)g(x) dx$.  
Our "basis vectors" are also functions: $\phi_n(x)=\sqrt{2}\sin(n \pi x)$.

Arfken introduces the expansion of the delta function in a weird way. He
states that\
$\delta(x-t) = \sum \phi_n^*(t) \phi_n(x)$\
and then shows that it does, in fact, match the definition of the delta
function.

I really don\'t like this, since it makes the expansion seem magical. I
think it makes much more sense to just *find* the expansion the way you
would with any other function.

So, in the $x$ domain:\
$\delta(x-t) = \delta(x-t)$.

We can find the $\phi$ expansion of $\delta$ just like we would with any
function:\
$\delta(x-t) = \sum_n \left< \phi_n(x) \, | \, \delta(x-t) \right> \phi_n(x)$

We can then expand that bra-ket to figure out the form of $\delta$ in
the $\phi$ space. If we\'re using the usual definition for the inner
product:\
$\left< \phi_n(x) \, | \, \delta(x-t) \right> = \int \phi_n^*(x) \delta(x-t) \, dx$\
\$ = \\phi_n\^\*(t)\$

So, the expansion of $\delta(x-t)$ must be:\
$\delta(x-t) = \sum \phi_n^*(t)\phi_n(x)$

It\'s important to note that this depends on the \"usual\" definition of
the inner product. If we had something *other than*
$\left< f | g \right> = \int f(x) g(x) \, dx$, then this would not be
the correct expansion for $\delta(x-t)$.

### Define the basis functions and the expansion of $\delta$. {#define-the-basis-functions-and-the-expansion-of-delta}

Arfken uses the Fourier expansion of $\delta(x-0.4)$ on the interval
$[0,1]$ as his example. This is that.

\... Well, this is that plus some other stuff too.
#### How it\'s usually shown

Basically, as you use more and more terms to approximate $\delta$, the
approximation looks better and better. You can also take the integral of
the approximation to gauge how close it is.

Warning: These plots may take a long time to render, if you decide to
run this code yourself!

![](diracDeltaExpansion.gif)

#### Adding the basis functions in reverse order

Okay, so that\'s pretty neat. It also looks how you\'d expect if you\'ve
ever been shown a plot of a Fourier approximation. It more or less has
the shape of the function, with some high-frequency junk thrown in, the
sharp edges are extra wiggly, and adding more high-frequency terms get
you closer to the function.

But, what if we don\'t start with the low-frequency terms and then add
the high-frequency terms? What if we start with the higher terms and
then add the low ones?

This code approximates $\delta(x-0.4)$, but adds in the terms in reverse
order, relative to the last plot. So, it starts out with
$\sqrt{2}\sin(200 \pi x)$ and then becomes
$\sqrt{2}\sin(200 \pi x) + \sqrt{2}\sin(199 \pi x)$ and so on.

It\'s kind of neat how the expected integral ($\int\delta(x-t)\,dx=1$)
doesn\'t really approximate the expected value until the last term is
added. It makes sense, since the last term contributes the most to the
integral. It\'s also kind of neat how the terms really only flip the
integral around zero until the last term.
![](diracDeltaReverseExpansion.gif)

#### Add terms randomly

This one just goes crazy and shuffles the order in which the terms 1
through 200 are used.

![](diracDeltaRandomExpansion.gif)

#### Even terms only

It might also be interesting to plot only the even or odd terms, since
only the odd terms contribute to the integral.

![](diracDeltaEvenExpansion.gif)

#### Odd terms only
I suppose it makes sense that there would be an oddly reflected peak on
the other side of $x=0.5$, since all the even terms have odd symmetry
about that point. And of course, the value of the integral just seems to
be rounding error, since the even terms don\'t contribute anything to
the integral.

I\'m not sure what the deal is with the oscillations between the peaks
though.

![](diracDeltaOddExpansion.gif)

The even-symmetrical peaks make sense for similar reasons as the last
plot. All the odd terms have even symmetry about the middle. And the
value of the integral closely approximates 1, since the odd terms are
the only ones that contribute to it.

### Same thing using Legendre polynomials
![](diracDeltaLegendreExpansion.gif)
Well would you look at that, math really does work!

Also, I totally see why we don\'t use Legendre expansions in numerical
stuff. It totally sucks. It takes *way* more time and terms to get the
same quality of approximation. To be fair, the way I wrote it, the
Legendre approximation has two extra loops in it, relative to the
Fourier version (one for `sapply()` and one for `sumover()`). *But*, the
Legendre functions also have *two* combinatorial functions in them\...
So, I\'m pretty sure it\'s those factorials that are taking so much
time.

That\'s pretty cool though, that we\'re completely out of the nice,
familiar realm of Fourier analysis and into some other weirdo basis
functions. It\'s nice to see these Hilbert space shenanigans
demonstrated with something that *isn\'t Fourier*.
