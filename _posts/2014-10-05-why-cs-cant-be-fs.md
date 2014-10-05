---
layout: post
title: "Why can't C# become F#?"
description: ""
category: 
tags: ['fsharp']
---
{% include JB/setup %}

Recently on twitter, I asked a [question](https://twitter.com/adamchester/status/517432980323590144):

> I wonder which will happen first: C# gets more awesome F# features, or everyone realises that F# already has the awesome features

A more knowledgable person than I, [Vasily Kirichenko](https://twitter.com/kot_2010) eventually [replied](https://twitter.com/kot_2010/status/517570346506280960):

> c# cannot get some of the fundamental f# features. "Everything is expression" for instance.

This got me thinking, what specific features of F# are impossible, or even unlikely to be the default, in C# in the near future?

Here's a list I have started:

 * __Everthing is an expression__ - expressions are more composable, safer, and compact.
 * __Immutable by default__ - C# has a long history with default mutable, it seems changing this would be near impossible.
 * __Default non-nullablity__ - C# has a long history with default nulls, as above.
 * __Type inference__ - while C# has the `var` keyword, and may get improved type inference capabilities in the future, it seems highly unlikely it could approach the capabilities of F# without a fundamental redesign.
 * __Conciseness__ - The F# language is basically designed from the ground-up to be short and concise. From not requiring semicolons and curly brackets, to significant whitespace for code blocks. Common patterns like structrual equality, and type inference contribute to this feature.

More generally, it seems unlikey that C# would do this in the near future:

 * __Partial function application__ - Unsure - could this happen?
 * __Functions that don't require a class__ - C# is primarily OO - and the class is the basic building block. It seems like it would require a huge shift in thinking for this to happen.
 * __Computation Expressions__ - allowing features such as the very flexibile `async` workflows.


Have I missed anything here? Have I got something wrong? Let me know.
