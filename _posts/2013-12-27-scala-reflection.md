---
layout: post
title: Learning Scala Macros
published: true
tags:
    - Scala
    - Macros
    - Tech
---

In the [first part of this series](http://imranrashid.com/posts/learning-scala-macros/), I learned how to use Macro Annotations to add a few simple methods to a class, to fill
in the definition of a trait.  My eventual goal is to be able to take a trait which defines a few basic `Int` and `Float` fields,
and automatically generate a class that would read those fields from a `ByteBuffer`.  Eg., given this:

<script src="https://gist.github.com/squito/7094987.js?file=trait.scala"></script>

My macro should generate the equivalent of

<script src="https://gist.github.com/squito/7094987.js?file=goalClass.scala"></script>

However, I was only able to add the methods for one fixed trait -- not very useful.  If I was
going to be able to fill in the definitions for any trait, I'd need to use Scala Reflection to figure out which methods I needed
to add.

Once again, I'd like to warn to the reader that is not so much a tutorial, as a journal of my path to discovery, with some mistakes
and detours along the way.

## Reflection Basics

My first step was to learn the basics of [Scala Reflection](http://docs.scala-lang.org/overviews/reflection/overview.html).  Its pretty easy to play
around with Reflection in the repl.  After starting with our same set of imports from last time:

import language.experimental.macros
import reflect.macros.Context
import scala.annotation.StaticAnnotation
import scala.reflect.runtime.{universe => ru}

the first key step is get a handle on the `Type` of something, using the `typeOf` method from our universe.  Once we have a type, we can ask for lots
of information, like all methods and variables.  (If you're familiar with Reflection in Java, this should all be similar in spirit.)

<script src="https://gist.github.com/squito/7847796.js?file=simple_reflection_repl.scala"></script>

Note that `typ.members` returned a `MemberScope`, but its a `Traversable` which I can `map`, `filter`, etc.  Here I've filted down to just the methods, and
also "cast" them to methods with `asMethod`.

I only needed to do some small filtering on top of this.  I wanted to filter down to only those methods that took no arguments, returned either an `Int` or a
`Float`, and that were undefined.  I was able to write some fairly straight-forward utility methods to handle the first two:

<script src="https://gist.github.com/squito/7847796.js?file=reflection_method_filters.scala"></script>

I could use these the type of my example trait, and filter out most of the methods I should ignore:

<script src="https://gist.github.com/squito/7847796.js?file=repl_method_filter_1.scala"></script>

I only had one problem left.  My trait had supplied a definition for `y`, but I wanted my macro to only supply a definition for *undefined* methods.

#### Detour: Using Reflection on Methods

There is actually a very simple solution to finding defined and undefined methods.  But first, I got the crazy idea of _using reflection to discover
how to use reflection_.  This is section is totally unnecessary for my end goal, but hopefully you'll find it interesting nonetheless.

I needed to find some way to differentiate method `x` from method `y`.  My first instinct was to just look at what was available to me with tab-completion,
thinking would look like "isDefined".

method_repl_explore.scala

Hmm, scanning the list, i didn't see anything that jumped out at me.  The list of methods was too long to go through them all, and see which one would be different.
But then I realized -- I could use reflection to call of those methods on both, and see which ones yielded different results.  That is, I would call all of the
methods available on the `MethodSymbol`s.  I could use reflection to call those methods -- that is, I'd be getting `MethodSymbol`s that were members of `MethodSymbol`.   (Yes, there was probably an easier way, but now this just sounded fun.)  To call methods using reflection, you first need to get a runtime mirror, from that get an instance mirror, and then finally from there you can call the methods.  Putting it all together:

<script src="https://gist.github.com/squito/7847796.js?file=diff_method_methods.scala"></script>

Now I could run this on `x` and `y` to figure out how to find an abstract method:

<script src="https://gist.github.com/squito/7847796.js?file=diff_method_results.scala"></script>

Hmm, that was a little disappointing.  The only methods with any differences were from the name or the method type.  I was expecting a method that returned `true` for `x` and `false` for `y` (or vice versa).  What gives?

So I turned to stackoverflow.  Turns out [it was an oversight](http://stackoverflow.com/questions/16792824/test-whether-a-method-is-defined), but there is a workaround.

<script src="https://gist.github.com/squito/7847796.js?file=is_method_defined.scala"></script>

Nonetheless, that was a fun little experiment in using reflection. (I warned you it was a detour)


## Runtime and Compile Time Universes

So far I've been using runtime reflection as an easy way to learn.  But to use in my macros, I'd need to switch to compile time reflection.  For the most part, reflection is the same, but you use a different `Universe` in compile time reflection.  Since all of the types are dependent on the universe, these also means all the types of my methods change.

I really wanted to have common methods which worked for both runtime & compile time reflection, and which I could unit test.  Turns out its pretty simple.  I put all of [my methods into a helper class](https://github.com/squito/learn_macros/blob/master/macros/src/main/scala/com/imranrashid/oleander/macros/BasicReflection.scala) which takes a `scala.reflect.api.Universe` (the parent to runtime & compile time universes).  Then I can instantiate it with either `scala.reflect.runtime.universe` or `context.universe` in a macro.

The full code is checked into my learn_macros project, even with some [simple unit tests](https://github.com/squito/learn_macros/blob/master/macrotests/src/test/scala/com/imranrashid/oleander/macros/BasicReflectionTest.scala).

## Parameterizing Annotations

I've got the basics of reflection -- now I just needed to get a handle on a `Type` of my target trait inside my macro annotation.  This one was surprisingly difficult to figure out, I had to turn to stackoverflow in the end.  As always, [Eugene Burmako was quick to give an answer](http://stackoverflow.com/questions/19791686/type-parameters-on-scala-macro-annotations).

When using my macro annotation, I make the target trait a type parameter on the annotation class, and then pull it out the type with a call to `typeCheck`:

<script src="https://gist.github.com/squito/7847796.js?file=annotation_type_parameters.scala"></script>

Then the annotation can be used like so:

<script src="https://gist.github.com/squito/7847796.js?file=using_annotation_type_parameters.scala"></script>

Again, I put together some [unit tests to verify the behavior](https://github.com/squito/learn_macros/blob/master/macrotests/src/test/scala/com/imranrashid/oleander/macros/MacrosWithReflectionTest.scala)

## Putting It All Together

We've got all the key pieces now -- its just a matter of putting it all together.  Since what's left is mostly normal scala coding, I won't go through it in
detail here.  But you can take a look at the [full implementations](https://github.com/squito/learn_macros/blob/master/macros/src/main/scala/com/imranrashid/oleander/macros/ByteBufferBacked.scala).  There are two versions of my annotation, `@ByteBufferBacked` with just getters, and `@MutableByteBufferBacked` which also adds in setters.  You can also take a look at the [unit tests](https://github.com/squito/learn_macros/blob/master/macrotests/src/test/scala/com/imranrashid/oleander/macros/ByteBufferBackedTest.scala) or just check out the full project and run the tests yourself.

## Conclusion

In these two blog posts, we've learned how to use scala macro annotations to automatically expand a class definition so that it implements a trait.  Alogn the way we've learned the basics of working with ASTs, how to simplify our lives with quasiquotes, and how to use reflection to explore types.  Our macro isn't the most robust yet -- it could certainly use a lot more error handling -- but we've certainly got a taste for how everything works.

I won't promise more in this series, but there are few more ideas I'd like to explore.  First, Eugene Burmako suggested that I could achieve the functionality I want in my library using normal macros, instead of macro annotations.  While it would change the user api somewhat, this is particularly appealing because
macro annotations won't make it into scala until 2.12 at the earliest.

Second, now that I've got a proof of concept, I'd really like to explore the idea of having classes store their data in `ByteBuffer`s.  Probably most readers are only interested in the discussion of using macros and reflection, but aren't sure what the point of my macro is.  I hope that by expanding these ideas somewhat, I can make it easy to store general purpose data structures directly in byte buffers.  That can have all sorts of potential benefits: save memory, avoid serialiation, store data off-heap, and lead to more cache-aware data structures (particularly important for numerical computing).  But, its still just an idea, I need to prove those claims.

I hope you've found this helpful in your exploration of macros.  Please let me know if you found this useful, if parts are unclear, or if you found any errors.

