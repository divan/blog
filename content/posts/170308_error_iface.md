+++
Description = ""
date = "2017-03-08T12:07:40+01:00"
title = "Misusing error interface"
slug = "misusing_error_interface"
Tags = []

+++


I used to think that misunderstanding interfaces in Go can lead, at most, to not very readable code and worse maintainability. From my observations misusing interfaces becomes visible usually during refactorings, where you questioning what this type or abstraction actually represents. But, at least, the code tends to work and it's not buggy.

# The bug
But here is the interesting piece of code that actually was buggy:

```go
err := SomeFunc()
if _, ok := err.(MyError); !ok {
    log.Println(err)
}
```
Quite idiomatic, right?  Error type naming is good, following [recommended way to name error types and variables](https://github.com/golang/go/wiki/Errors#naming).  We use here normal [type assertion](https://golang.org/doc/effective_go.html#interface_conversions), "extracting" underlying static type from an interface. If an error doesn't have this type inside, we should log it.

But it didn't work. Variable `ok` was always true.

First, I checked `SomeFunc()`:

```go
func SomeFunc() error {
    // case 1: we want caller to print this error
    err := json.Unmarshal(data, &v)
    if err != nil {
        return err
    }
    ...
    // case 2: we want caller to ignore this error
    err = errors.New("my error")
    return MyError(err)
}
```

Well, it's a bit smelly, but looks kinda ok. In the first case, we definitely do not return `MyError`, so type assertion should reflect this. Why was `ok` true in this case? 

Next jump to the definition of `MyError` gave me an answer:

```go
type MyError error
```

What??

`MyError` was actually an interface type! Even more - it was "synonym" for `error` interface!

Developer who wrote this code was assuming that
```go
type MyError error
```
 is the same as:
```go
type MyError struct {
   error
}
```
Which is more than far from the truth.

# Understanding interfaces
If you do understand interfaces in Go, you will spot immediately that in a first case MyError is an interface type, while in the second case it is a concrete type.

The tricky part here is that type assertion works both with concrete and interface types - it's ok to "extract" one interface from another. For example, you can "reduce" `io.ReadCloser` to `io.Reader` by this:
```go
r, ok := rc.(io.Reader)
```
and it's perfectly valid case. It's [in the spec](https://golang.org/doc/effective_go.html#interface_conversions):

> That type must either be the concrete type held by the interface, or a second interface type that the value can be converted to.

I do believe that to truly understand interfaces you need to learn how they are implemented. I wrote a blog post some time ago with visual explanations of interfaces - ["How to avoid go gotchas"](https://divan.github.io/posts/avoid_gotchas/). Here is a refresher picture what interface is under the hood:
![interfaces](https://divan.github.io/images/iface2.png)
If you haven't read it yet, [please do](https://divan.github.io/posts/avoid_gotchas/) - I'm genuinely interested in a feedback and hope my way of explaining things works for others.

Now, back to our the initial code. What was happening there is following:

 - `SomeFunc()` returns error of interface type `error` with a static type inside different from `MyError`
 - type assertion trying to "extract" interface `MyError` from interface `error`
 - as `MyError` was defined as the same type as `error`, any variable of type `error` automatically satisfies `MyError` interface as well

Hence, type assertion `err.(MyError)` is always positive for any non-nil variable of type `error`.

# Conclusion

Misunderstanding interfaces can lead to some non-obvious bugs, so it's crucial to build a proper understanding of what concrete and interface types are in Go.

If you feel like you don't have a clear picture, try one of my previous articles, where I tried to explain interfaces visually - ["How to avoid Go gotchas"](https://divan.github.io/posts/avoid_gotchas/). And, as always, don't forget the (re)read the ["Effective Go"](https://golang.org/doc/effective_go.html#interfaces_and_types).

Also, proper testing - when you test not only happy path - would have caught this bug before it went to production.

Happy coding!
