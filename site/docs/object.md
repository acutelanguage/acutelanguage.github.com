layout: default
title: Acute Programming Language
subtitle: Small, focused documentation
---
## Object Model

As in Io, Acute has a small object model. It needs to be lightweight for a variety of reasons. For instance, we create tens of thousands of objects over a typical small program. Io's objects rack in at a hefty **60 bytes** on a 64-bit platform. Acute on the other hand, weighs in at a whomping **16 bytes** assuming no packing occurs. Realistically, it only needs **65 bits**, or if we encode the lookup bit into the MSB of the slot table, we need just **8 bytes**. These are discussions for another time though. Suffice to say, objects in Acute are very lightweight.

Everything in Acute is represented by an object. The global namespace, locals of a method, even the AST (messages). Our model is not so different to that found in the Io programming language, with a couple of notable exceptions. First, and easiest to explain, is the fact that they are more efficient in terms of space; lastly, we grant you more hooks into the system allowing you to redefine language semantics at runtime. This may sound a bit scary, and in fact, it can be, but that's a philosophical discussion for another day.

Let's start with an example, how does the model function.

### AST Creation

As discussed in the [Syntax](/docs/syntax.html) section previously, we work by passing messages to other objects. We walk the tree of messages to evaluate them; but how is this tree created? Simply, we split the source file up into tokens, match those tokens against expected patterns, and create message objects. Then we pass this into the objects evaluator, and let it take it from there.

### Evaluator

The basic evaluator operates in a two phase process. First we must look up the name of the thing we're calling, find out which object it lives on, and return its value (or in the case of the Acute bootstrap, we do this with an additional layer of indirection I'll talk about later). This value gets passed to the evaluator which checks it for things like an activatable bit, which if present, will perform another lookup for an `activate` method, invoke it and return its result. Otherwise, it will just return the result directly. This result is then used as the next receiver in the tree walker; and so the cycle of life spins.

### Slots

Each object maintains a list of what are known as *slots*. These hold data. In Io, the data they hold are the objects themselves. In Acute, they're an intermediate object known as a *Slot*. This Slot object has slots itself, for instance, its `data` slot will hold the object directly, without an intermediate Slot object. This avoids a recursive dependency problem. We provide this intermediate layer, for no other reason than flexibility. It allows you to build up some features easily that are more difficult to do in Io directly. One example of this is slot access control.

Consider for a moment, that you want only the instance of an object to call certain methods on itself. What might that look like?

    setSlot("Foo", Object clone do(
      setSlot("number", 1)
      setSlot("privateMethod", method(number *(2)))
    ))

Now, we may only want to allow access to `privateMethod` from within a particular copy of `Foo`. This is a non-trivial problem in Io. You'd have to create a proxy object which maintains the bookkeeping information on this access control. The caller would have to interface with it, or you'd have to hack it into place. Since Io doesn't allow you to override slot lookup in a meaningful way, this becomes harder to do. I'm not going to go into details of how to do it in Io, I just don't have the time or energy. Suffice to say, it's easier in Acute. Consider:

Lets assume that we already have our `Foo` definition as above. We just need to change a few things.

    Foo setSlot("lookup", method(str,
      setSlot("slot", self _lookup(str))
      slot ifTrue(
        slot isPrivate and(call sender ==(self)) ifFalse(Exception raise("You tried to call a private method"))
      )
      slot
    ))

That's pretty much it. Here we override slot lookup, extending it in such a way to ensure our caller has appropriate permission to call the slot.

### Recap

To recap, our algorithm is like this:

1. Create AST nodes from input stream as `Message` objects.
2. Pass nodes to evaluator
3. Check message for a cached result, if present, return it immediately
4. Look up the name of the message in the receiver of the message (recurse through parent tree)
5. If we fail to find an object through lookup, throw an exception
6. Check the activatable bit on the object we found
7. If object is activatable, look up its activate method (recurse through parent tree)
8. Throw an error if we have no activate method
9. Call the activate function and return its result
10. If object wasn't activatable, return the result
11. Tree walker takes the last result from the evaluator and uses it as the receiver for its next message.
