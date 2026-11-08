---
title: Handling Java InterruptedException
date: 2021-07-26 12:00:00 +00:00
tags: [Java, Exception, InterruptedException]
description:
---

## The bad practice

From my own experience as a developer, Java's `InterruptedException` is one of those checked exceptions that you come across from time to time (much more often if you work in a concurrent programming context) but never deal with correctly. Generally, you (or I—or, hopefully, the old me) either end up swallowing the exception or, at best, logging it.

Before we go into how we should deal with this typed exception, we need to understand what this exception is for in the first place.

## What is the InterruptedException for?

In Java (or any other language), when we call a method, the method may take a long time to finish, either because the method itself does a lot of work or because the method blocks on some external event (I/O operations, locks…). However, in order to have responsive applications, sometimes you want to be able to cancel an operation (for example, you launch a compilation and then try to cancel it gracefully), or, in other words, you want to be able to tell a thread to stop doing what it’s doing.

## The good practice

In Java, the Thread class provides a method whose job is to interrupt a thread:

```
public void interrupt()
```

What this method does is simply set a status flag (that every thread has), the “interrupt status”. When this flag is set, it means someone is asking the thread to stop gracefully (meaning, cleaning everything it needs to clean before stopping). When a thread notices that the flag is set, for example, by using the following method:

```
public boolean isInterrupted()
```

It should clear it and then throw an `InterruptedException` to signal to the calling method that it was interrupted. I am using the word should here because the thread could just ignore the flag and keep doing whatever it’s doing. So, interruption relies on good practices from the implementers of the code running in the thread.

So, when you see a method that throws an `InterruptedException`, it is basically saying that it is a blocking method, that it provides the possibility to interrupt it gracefully, and that, if it is ever interrupted, it will throw an `InterruptedException`.

Now, the mistake that developers generally make when they call such blocking methods is that they don’t handle this exception correctly (for example, by just catching the exception and doing nothing in the empty catch block)… What the caller of such blocking methods should do is either pass the exception up the stack to let the caller treat it or, if the exception cannot be propagated (maybe because you implement an interface that you can’t change), catch the exception, do whatever cleaning you need to do at your level, and reset the interrupt status. This way, you don’t lose the information that the thread was interrupted, and you give calling methods the possibility to interrupt themselves too, in case they are relying on this flag.

## Conclusion

To sum things up:

This is bad

```
try {
// Some blocking code that throws InterruptedException
} catch(InterruptedException ex){
// Nothing
}
```

This is better

```
try {
// Some blocking code that throws InterruptedException
} catch(InterruptedException ex){
//... some cleaning if needed ...
Thread.currentThread().interrupt();
}
```

## References

- [https://docs.oracle.com/javase/tutorial/essential/concurrency/interrupt.html](https://docs.oracle.com/javase/tutorial/essential/concurrency/interrupt.html)
- [https://docs.oracle.com/javase/8/docs/api/?java/lang/InterruptedException.html](https://docs.oracle.com/javase/8/docs/api/?java/lang/InterruptedException.html)
- [https://docs.oracle.com/javase/8/docs/api/?java/lang/InterruptedException.html](https://docs.oracle.com/javase/8/docs/api/?java/lang/InterruptedException.html)
