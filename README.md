# Neve

Neve is a hybrid-paradigm interpreted programming language favoring functional programming.  It is designed
to be not only easy, but *fun* to use, be predictable, expressive, and *never crash*.

I’m writing this as an informal specification for the language because I really wanted to get my thoughts out.  I’ve 
been spending a couple years coming up with a language that would be fun to use for my non-performance critical 
side projects, and that could also hopefully be fun to use for you too!  And I’d really like to know what your thoughts
are.  If I’ve missed anything or something seems confusing, please kindly let me know!

And, just in case you’re curious–development started a couple months ago, but it’s been going pretty slowly.  
Neve barely supports anything, and I’m working on rewriting the whole bytecode compiler from Python to Kotlin 
because Python’s been causing a mess, so I wouldn’t recommend trying out any programs yourself just yet.  If
you still want to check it out, you’re welcome to visit [nevec](https://github.com/neve-lang/nevec) and 
[neve](https://github.com/neve-lang/neve).

# Table of Contents 

* [Core Philosophy](#core-philosophy)
* [Quick Syntax Overview](#quick-syntax-overview)
* [Essential Semantics](#essential-semantics)

## Core Philosophy

With every new language comes a same question: **why use it over another language**?  This essentially comes down 
to what the language itself brings, why it was made in the first place, and what makes it unique.  This section 
will mainly involve words of mouth.

Neve’s core principles are the following: **expressiveness**, **safety and predictability**, and **awesome DEVX**.  

### Expressiveness

Neve tries to achieve expressiveness by **favoring functional programming**.  It still supports `while` and `for` loops 
due to its hybrid nature, but it makes it clear that functional constructs are preferred wherever possible, by doing 
tiny things like [requiring the `for` loop variable to be mutable].

Neve tries to achieve expressiveness by using what I like to call its **soothing syntax**, but it’s difficult to 
hit the mark just right with syntax–it’s not objective, and it spends a lot of the language’s strangeness budget.

### Safety and Predictability

Neve values safety and predictability *a lot*, so much so that it intends to be *a language that never crashes*.  
Now, of course, Neve *will* crash if you run into fatal issues like a heap corruption error, or if you manually write 
a valid Neve bytecode file that does unsafe things.  But in a regular setting, your code should *never crash* because of
a runtime error–those don’t exist in Neve, thanks to its [refinement types] and deep value analysis.

Now, I understand that those are bold claims to make, and it’s easy to ask yourself, “if Neve (plans to)
do it, why haven’t other languages already done it?”  But my answer to this question is, I think it’s expected to have 
new ideas emerge over time.  For example, Java is a widely used language, and yet, it didn’t support null safety until Java 
1.8, introducing its `Optional` class.  Other languages just don’t have the goal to be a language that never crashes.

And, if you’re still skeptical, I completely understand.  This `README` aims to answer every question you might be 
asking yourself right now, and if you see an issue I hadn’t caught, please let me know!

### Awesome DEVX

The last of those core points can’t really be proven by language features, and with a compiler and VM still under heavy 
development, the best I can show you are Neve’s compiler error messages.  So here’s one:

![image](assets/error.png)


## Quick Syntax Overview

In this section, I’ll try to give you a quick taste for Neve’s syntax, without dumping too many language 
features at once and without talking about syntax forever.  This means that I’ll have to keep the example simple, 
but that’s okay.  With that said, here’s a simple Neve program:

```rb
fun main
  let numbers = 0..10

  # Initial numbers: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  puts "Initial numbers: #{numbers}"

  let evens = numbers.filter with Int.is_even
  let doubled = evens.map |x| x * 2

  puts "Then: " if doubled.is_empty = "No doubled evens!" else doubled.show
end
```

It’s a silly program, but hopefully, it gives you a clear vision of what Neve is.

A couple things you might’ve noticed:

* Newlines are significant in Neve.  [How are they treated at the parser level?](#newlines-in-neve)
* Parentheses to call a function are optional in Neve.  [How does the parser handle that?](#function-calls-with-optional-parentheses-in-neve)

These questions dive into how the parser and lexer themselves work, so I’ll leave them at the bottom of this 
`README`, just in case you’re interested!

## Essential Semantics

In Neve, variables are declared with a `let` keyword or a `var` keyword.  `let` is used to specif
immutable, whereas the latter says that  

### Newlines in Neve

As mentioned just above, newlines in Neve are significant whitespace.  But there seems to be a lot of discussion 
regarding the best way to implement a parser that supports them.  I think Neve’s approach to this problem is interesting, 
and felt like sharing it just in case it inspires someone’s approach to the same problem!  You’re welcome to skip this section 
if you’re not interested.

Neve’s approach is simple–it ignores any type of whitespace *unless it’s required*.  Now, before you roll your eyes: I 
promise it’s not just like JavaScript does.  Let me explain in detail.

During the parsing phase, the Neve Lexer skips all kind of whitespace *except* for newlines.  For these kinds of tokens, 
it emits a **newline token** that is passed over to the parser, just like a normal token.

However, this is where the interesting part happens–because Neve uses a single-pass parser, the parser keeps two tokens 
in memory at every state: a `previous` token and a `current` token.  They’re pretty straightforward–when the `current` token
changes, the `previous` token is set to `current`’s previous value, and then we continue.  However, the parser treats 
**newline tokens** a bit differently.  Whenever it calls `advance()` or a function that moves the parser’s position forward, 
it *skips every newline token*, and once they’re all skipped, the last newline token stays in `previous`, because `current` 
just skipped it.  This lets the parser skip every newline token it doesn’t need, and it just has to check if the `previous` 
token is a newline token to check if there was a newline.  This can easily be abstracted away in a tiny `hadNewline()` 
function.

### Function Calls with Optional Parentheses in Neve

Optional parentheses are another can of worms, and I thought clarifying how Neve handles that could also be valuable.  Same
as before–you’re free to skip this section if you want to.

Neve considers something a function call if it sees an identifier, and one of the following:
* A single **newline token**.
* An **expression starter token** *with no newline token before it*.
* A parenthesis *with no newline token before it*.

This means that the following will be considered function calls in Neve:

```rb
let a = call
let b = call x, y, z
let c = call(x, y, z)
let d = call(
  x,
  y,
  z
)
let d = some.call.this.and_that(x, y, z)
        #    ---- ---- -------- will be called if they are associated functions
        #                       (this is done by the semantic resolver, which
        #                        examines the AST and outputs a corrected version.)
```

But the following *won’t*:

```
let a = call
("Hello")
```

Neve considers the following tokens **expression starters**:
* Left bracket tokens: `(`, `[`, `|`
* Identifiers
* Strings, integers and floats
* Expression or expression starter keywords: `true`, `false`, `nil`, `not`, `self`, `with`
