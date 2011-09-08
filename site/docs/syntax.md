layout: default
title: Acute Programming Language
subtitle: Small, focused documentation
---
## Syntax

Acute implements a basic syntax, which can be composed out of three rules:

    expression ::= literal | message
    literal ::= [0-9]+ | "[^"]*"
    message ::= identifier [ '(' [ arg [ ',' arg ]* ]? ')' ]?

All message sends take this form. We do not support any operators at all, including assignment. Everything is done through message sends, including assignment.

### Examples

To add two numbers, one might construct a chain of messages like the following:

    1 +(2)

Note the parenthesis, these are required. Any message that takes arguments requires parenthesis, addition is no different than any other message, it is not an operator.

We can create a new object by passing the `clone` message to some other object.

    Object clone

…and methods can be defined like this:

    method(a, b, a +(b))

Obviously, we'll want to define a name for our method; or rather, associate our method with some slot named something. To do this, we can call `setSlot`.

    setSlot("add", method(a, b, a +(b)))

Now we can call it like this:

    add(1, 2)
