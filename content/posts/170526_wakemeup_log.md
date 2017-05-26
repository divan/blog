+++
Description = ""
date = "2017-05-26T12:55:00+01:00"
title = "WakeMeUpInTheMiddleOfTheNight log level"
slug = "wakemeupinthemiddleofthenight"
Tags = []

+++

On the last [Golang Barcelona meetup](https://www.meetup.com/Golang-Barcelona/) we decided to try new format of conversation and after the talk started open discussion on logging, tracing and metrics. The idea was to encourage people to read a very nice blog post by Peter Bourgon on this subject - ["Metrics, Tracing and Logging"](https://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html) and discuss it in a free format.

It went surprisingly well, and while most of the people were conscious about the topic, one thing seemed to be still confusing - what to log and when. I think we're all more or less agree that we should log only when program requires human to take a look. It wasn't a case, let's say, 10 years ago - there were no ELK, solid solutions for metrics or tracing - we were using logs for everything, then were building "log analyzers" to deal with a vast amount of information. But now, when we have all these best practices, powerful time-series databases, visualization platforms, tracing tools - what should we print to the log?

One of the coolest approaches to clarify this question I've found in the brilliant article by Daniel Lebrero ["Logging levels: the wrong abstractions"](https://labs.ig.com/logging-level-wrong-abstraction) (thanks [@katzien](https://twitter.com/kasiazien) for sharing this article). The idea is to rename the log levels, at least temporarily, and see how your perception of it changes.

So, let's say you're using any of the popular logging packages for Go, and you probably have these calls all over your code:

```go
log.Error("some error")
log.Warning("some warning")
log.Info("some info")
log.Debug("some debug information")
```
Looks familiar, right?

Now, let's rename those calls to:

```go
log.WakeMeUpInTheMiddleOfTheNight("some error")
log.ToInvestigateTomorrow("some warning")
log.InProdEnv("some info")
log.InTestEnv("some debug information")
```

How does it look like? :)

Yeah, renaming `Error()` method of your logger into `WakeMeUpInTheMiddleOfTheNight()` not only makes it harder to type and forces you think twice, but also gives you a very straightforward answer to the question "Should I print this to the log?". Do you want to be woken up in the middle of the night when this error occurs?

You may quickly create an wrapper for your logger (even if you use `log.Println("[ERROR] error message")` approach) and have something like this:

```go
type LoggerWrapper struct {
    *Logger
}

func (l *LoggerWrapper) WakeMeUpInTheMiddleOfTheNight(fmt string, args ...interface{}) {
    l.Error(fmt, args...)
}
...
logger.WakeMeUpInTheMiddleOfTheNight("something failed")
```

Just try it.
You may want to use `gofmt` tool for that:

```
gofmt -w -l -r "log.Error(x) -> log.WakeMeUpInTheMiddleOfTheNight(x)" .
```

I tried to do this with some of my projects and it was pretty interesting.

For example, when you see:

```go
log.WakeMeUpInTheMiddleOfTheNight("Migrations failed")
```
It makes sense. If running migrations process has failed, it's a sign that something went terribly wrong and I want to be woken up in the middle of the night to investigate this. But when you see this:

```go
log.WakeMeUpInTheMiddleOfTheNight("Page not found")
w.WriteHeader(htttp.StatusNotFound)
```
it immediately raises your eyebrow, Really, do you want to be woken up because of someone requested wrong URL?

# Conclusion
I personally find this idea brilliant, even in a form of mental excercise. It also makes you realize that you don't really need that "WARN" error level - what does it even mean? If it can wait till tomorrow - make it just normal log message, so human will read it and react. "DEBUG" is literally a normal log messages for the program running in a test or development mode. And, as you may guess, "INFO" is really redundant.

So I would argue, what you need in most cases is just two logging levels: normal message and error message (probably decorated with filename:line or stacktrace in some cases).

And, of course, use error level only when the program really needs you in the middle of the night.

Again, thanks for this brilliant idea to Daniel Lebrero and here again the link to the original article: https://labs.ig.com/logging-level-wrong-abstraction
