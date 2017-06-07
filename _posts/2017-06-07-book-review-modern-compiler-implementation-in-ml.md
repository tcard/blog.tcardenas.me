---
title: "Book review: Modern Compiler Implementation in ML"
date: 2017-06-07 02:33:23
---

As a starting point for my masterplan of late (writting a language), I've
recently finished reading [this book](https://www.cs.princeton.edu/~appel/modern/ml/).
So let's share some thoughts about it.

To be honest, "read" only applies to about 50 % of it. The rest of it I've only
"skimmed through", either because it's too basic (the lexing and parsing chapters)
or too advanced+concrete+cumbersome+terse+boring+I-know-LLVM-will-handle-this-for-me
(most of the polymorphism, OOP and dataflow chapters, implementation details
of optimizations, etc.).

### Target audience

First, I'll talk about my background, so that I put my views into context: I'm
kind of into languages. I'm at a very
very "beginner" stage, though. I've been following language people on Twitter,
I did the usual Dragon-Book-based compilers course at college, I've been
paying attention to the development of a few languages, specially Go and Rust,
and I've read a paper or two. But that's it, I've never really done any real
compiler or language design work.

From that, the book felt a little _light_ in that I already had at least
some notion about most of its concepts. I was expecting to get the missing
pieces in my knowledge, but I didn't really get them. Instead, those concepts got
consolidated and I'd be more confortable designing a compiler now. But there
were few "aha" moments for me. This is hardly the books' fault, though.

I think this book would work pretty well as an introduction to all this.
Definitely better than the [Dragon Book](https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools)!

<center>
<p><img src="images/appel-ml-1.png" alt="Tiling a IR tree diagram"></p>
<p><small>From IR trees to instructions</small></p>
</center>

### Contents

The first half of the book is about building a compiler for a toy language;
you're expected to follow along actually writing it on top of the downloadable
base ML code (hint: I didn't, but [some people did](https://github.com/prikhi/modern-compiler-implementation-ml/tree/master/tiger)).
The second half proposes several ways the compiler and/or the language could be
extended. Thus, the former reads more like a tutorial than the latter.

The explanations are mostly good, and the prose is well-crafted and easy to
digest. While some effort was put to explain the rationale behind everything, in the end
many things feel too _prescriptive_, too "just do this" specially on the first
part. Concretely, little to no effort
is put on discussing the architecture of the compiler code itself, how pieces
fit and interact together, and the source code comes with no comments at all.
(Special mention goes to [the procEntryExit[123] functions](https://github.com/prikhi/modern-compiler-implementation-ml/blob/master/tiger/frame.sig#L33-L35).
God, I sometimes wonder if many university professors have ever _read_ a line of code!)
Figures, graphs, example snippets and such are often used to great effect, moreso
than plain algorithms (or, God forbid, operational semantics or formal proofs).

The book is like 20 years old now, so some areas I'd be interested about, moreso
than the intricacies of optimizations, register allocation, etc., are not even
mentioned. The runtime system gets very little love. I continue to know nothing
about ELF and DWARF. Nothing at all about concurrency and green threads. Nothing
about JIT, or interpreters. And
little about interfacing with the OS in general. In any case, the book is already
thick enough, but I'd have been nice to have those explained at depth within
the coherent framework set up by the book, instead of having to read about it
from papers and different projects or books with their own idiosyncrasies that I have
to forcibly fit into my head.

Right now, I'm more interested in language _design_ than _implementation_.
But, being very conscious about how deeply they influence each other, it's great
that I know how things like instruction selection, call stack management and
register allocation actually work so that I can keep that in mind while designing
features. This book discusses more design
aspects that I was hoping for, anyway, which is great! It has different chapters
for implementing generics, functional stuff (closures, etc.) and OOP stuff
(inheritance, dynamic dispatch), so you can pick and choose and get a feeling of how those
are typically implemented.

<center>
<p><img src="images/appel-ml-2.png" alt="Register coloring!"></p>
<p><small>Register coloring, step-by-step!</small></p>
</center>

### Conclusion

I'm glad I got to read this book. It felt kind of like a rite of passage. Now,
I'm _initiated_ and I can go and write a compiler which probably would even
ressemble a little what real compilers look like, so I won't be laughed out of
the big boys rooms that hard maybe. I'll probably be able to go deep into most
compiler's code bases and not get too lost about what's going on, and how, and
why —which I expect I'll need to do often when building my own.

I definitely can recommend it. Go for it.


