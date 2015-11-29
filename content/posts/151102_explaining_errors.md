+++
date = "2015-11-02T08:36:54-07:00"
draft = false
title = "Explaining Go error handling"
slug = "go_errors"
tags = ["golang"]

+++

I recently translated great article — [Errors are values](https://blog.golang.org/errors-are-values) by Rob Pike — and we discussed it in our [podcast Golangshow](https://golangshow.com/) (in russian). One thing I was surprised about is that even experienced Go developers sometimes do not understand the core idea of that article.

Looking back, I remember my first impressions when I read it for the first time. It was similar to *“It looks like Pike just adds some complexity to what could’ve been solved gracefully with exceptions”*. I have never been fond of exceptions, but that’s the first thought I remember. The example in the article was clearly asking for comparison with exceptions’ way to deal with errors and it didn’t look like a winner here.

Still I knew, there must be something more profound in these words — *“errors are values”*. After all, I was always comfortable with Go errors handling, so I gave some time to myself to absorb the article.

And then I got it.

Go doesn’t want us to treat errors as something different from our main code. Erroneous situation is a first-class citizen in program flow design.

<blockquote>Errors shouldn’t be hidden or ignored in the same way as you don’t hide or ignore any other code. They are part of your logic and code.</blockquote>

Just try to imagine your way of thinking when you deal with usual concepts — values, conditions, loops etc., and apply it to the errors. Errors are the same level entities as the rest of your code. You don’t ignore return values of other types for no reason, right? You don’t ask language to bring special way to handle boolean variables, because “if” is boring. You don’t ask yourself “What should I do, if I don’t know what to do with this slice on this abstraction level?”. You just program the logic you need.

Again, errors are values, and errors’ handling is a normal programming.

Of course, there are always some patterns to deal with errors (like with any other programming conception), but they emerge naturally and fit perfectly in the existing language capabilities.

<hr />

Let’s try to illustrate it with an example, close enough to that one in the original article. Say, you have a task — “make repetitive writes with io.Writer and calculate number of bytes written, and stop after 1024-th byte”. You start with straightforward approach:

<pre><code class="go">var count, n int
n = write(“one”)
count += n
if count >= 1024 {
    return
}

n = write(“two”)
count += n
if count >= 1024 {
    return
}
// etc</code></pre>
http://play.golang.org/p/8033Wp9xly

Of course, you instantly see what’s wrong with this code and, following DRY principle, you decide to deduplicate code, moving repeating parts to separate function or closure:

<pre><code class="go">var count int
cntWrite := func(s string) {
  n := write(s)
  count += n
  if count >= 1024 {
    os.Exit(0)
  }
}

cntWrite(“one”)
cntWrite(“two”)
cntWrite(“three”)</code></pre>
http://play.golang.org/p/Hd12rk6wNk

Now it’s better, but still not perfect. You still need a closure, which depends on external variable. It also uses os.Exit(), which makes it hardly reusable after first refactoring. We can do better. Let’s see how our thoughts flow — we have a write function, which does something else, except just writing bytes and we need it to be reusable and isolated entity. Let’s refactor our code:

<pre><code class="go">type cntWriter struct {
    count int
    w io.Writer
}

func (cw *cntWriter) write(s string) {
    if cw.count >= 1024 {
        return 
    }
    n := write(s)
    cw.count += n
}

func (cw *cntWriter) written() int { return cw.count }

func main() {
    cw := &cntWriter{}
    cw.write(“one”)
    cw.write(“two”)
    cw.write(“three”)
    fmt.Printf(“Written %d bytes\n”, cw.written())
}</code></pre>
http://play.golang.org/p/66Xd1fD8II

Now it looks much better, we can reuse this custom writer in other functions, it’s isolated and easy to test.

Now, just replace ‘counter’ with ‘error value’ and you’ll get almost the same example as in original article about error handling. But take a note how easy and logical was your flow of thoughts towards this code. You wasn’t distracted by looking for the special counting/passing features of the language. You simply was implementing the logic you need in the best possible way.

<hr />

This idea is profound enough and could be hard to grasp, especially with the mindset focused on The Only Right Way To Handle Errors™. It definitely takes some time to absorb.

Of course, it’s debatable, and I can find both a lot of pros and cons for this approach, as well as for others. We’re not in the black&white world, but Go approach to errors is kind of mature and fresh at the same time, it extremely simple and hard to understand at the same time, it requires some rethinking and effort to get the idea. But, what is more important, it works great in practice.

And once you get it, you stop fight the language. You stop looking for special ways to handle or hide errors. Go makes you respect errors as any other part of your program. You just handle them, without expecting language do the magic for you. In the long run, your code becomes better, even if do not realize it yet.

Now, come and read this article again —  [Errors are values](https://blog.golang.org/errors-are-values) — and try to get the gist of it with this perspective.
