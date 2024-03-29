+++
date = "2015-12-14T08:36:54-03:00"
draft = false
title = "How to complain about Go"
slug = "go_complain_howto"
tags = ["golang"]

+++
Over the years of existence of Go programming language, the articles with its critique was always popular, bringing a lot of discussion from both sides. Recently, [Maksim Kochkin](https://github.com/ksimka) even created GitHub repo with [curated list](https://github.com/ksimka/go-is-not-good) of articles complaining about golang's imperfection.

So, is it true that ranting about Go flaws is a trend nowadays? With carefully gathered links in the repository above, we can check this! :) Unfortunately, there are only 17 articles in the list, which is a bit disappointing because it's not enough for fine statistical analysis, but we can use this anyway.

Here is a trending line for the [number of Go complaints written per year](https://docs.google.com/spreadsheets/d/1qKFykmm0yapLq1FKuouvqVZkWo-HHCfJnrTikqRoAfs/edit#gid=1956718702):

![Trending](/images/go_rants_trend.png)

The trending is obvious - from one in 2009 to five in 2015. Hopefully, the next 2016 year will continue this trend line.

But what I liked most about this list - is a brief resume on the major authors' points. We can measure the popularity of users' complaints by counting aggregated points. Again, let's do it in our Google Sheet document, which is written in Google, and probably involves a lot of production Go code to let us enjoy this:

![Trending](/images/go_rants_top.png)

Let me write again the top-5 winners:

 - error handling / no exceptions
 - no generics
 - stuck in 70's
 - no OOP
 - too opinionated

Again, it's a bit disappointing that absolute winner (*"error handling"*) is mentioned only 7 times. Many thousands of people are using Go every day and only 7 put their major complaint into words in the form of a blog post? This flank definitely needs more support.

If you don't like Go, you may contribute here. Especially if you tried Go before you started complaining about its design. But how to start the  article to make a proper effect? Take a look at these 5 top complaints and choose on which one you want elaborate.

Keep in mind, *"error handling"* and *"no generics"* are absolute winners - you have absolutely no excuse not to add them to your list. Next, be careful with *"no OOP"* because it's easy to attack by quoting Alan Kay. *"Too opinionated"* is a good choice, but should be carefully argued so you don't look like a grouchy geek rather than a thoughtful hacker. *"Stuck in 70's"* works best if you want not only complain about Go but popularize another modern and objectively good programming language.

But what if it's not enough? There are 41 more other complaints scored 1 or 2 in the list, not including ones you can make up on your own! So, to help you make a choice, here are some rules and guidelines on how to properly complain about Go, depending on you previous background and experience. 

For example, if you come from...

## ...Python
We all love Python, so the attack vector should be on the maturity and abundance of libraries. NumPy is a must have point - Go still don't have a scientific library of that quality. Not sure how SpaceX is able to use Go without one, but whatever, NumPy sells well.

Try not to mention mature libraries like Twisted, Requests and various solutions for solving the 10K problem. Just don't mention it. Instead, try to disprove relatively high memory usage in Python. For example, you can say that Python memory usage after goroutines leakage is competitive with Go. Someone from Mozilla [said this](https://docs.google.com/presentation/d/1LO_WI3N-3p2Wp9PDWyv5B6EGFZ8XTOTNJ7Hd40WOUHo/mobilepresent?pli=1&slide=id.g70b0035b2_1_119), so can you.

As a bonus, mention that Go slice indexing doesn't have convenient -1 index. *"Is compiled"* flaw also works here, without any explanation. Just because.

## ...C++
If you're C++ developer, you are in the most vulnerable position, because Go was created as an answer to the C++ problems. Which easily can be muted or presented as powerful features.

Start by demonizing GC. Everyone knows that Garbage Collection is an evil and every CPU tact matters. Even if you write simple REST-backend in C++ and it takes 6 months of your life to accomplish it - you still have speed and no-GC blissfulness. Stick with that.

As a logical continuation - tell that Go is not good for embedded. Don't mention that authors explicitly said that it was never meant as a language for embedded. It's a great point anyway. And if your commenters will send you links to [Embd](http://embd.kidoman.io), [Gobot](http://gobot.io) or [gomobile](https://github.com/golang/mobile), just say that it's not True Embedded™ or just disable comments.

And, of course, all rants about how Go restricting you from shooting yourself in the foot are also great here. Go doesn't make you feel clever than your coworker that bangs his head against the wall trying to understand your code. Why do you need such a language after all?

## ...Rust
Despite the fact that there aren't many people in the wild who can claim "background in Rust", there are still a bunch of people from C++ background who are totally in love with Rust. They didn't try it in production or even for pet projects, but that's not important. Pure love doesn't need logic. So, many points valid for C++ developers will work for you also. Don't forget *"zero-cost abstraction"* phrase after mentioning how evil GC is.

Cargo is considered to be a good solution for dependency management, so attack this side of Go aggressively. No chance to lose here. Also, your key points should be Go's simple type system and lack of pattern matching. Basically, everything that differs in Go from Rust will work here. And that's a lot, so you can write a solid longread. Or two.

## ...Java
After mentioning generics and exceptions, punch them in the guts and smash them by comparing Java's IDE with Go's IDE. Of course, there is no Go IDE, because Go is too simple to require one, but it's bad also. Double kick. It's "advanced" vs "primitive" and you're a winner here.

Next, write about the lack of good debugger (integrated into IDE, of course). How can someone write even simple code without a debugger? Don't ask if you really need debugger in Go that much as you need it in Java and why. Stay away from this subject, it's slippery.

And the whole range of 'features' that you can use to make people understand why Go is bad - from lack of JVM to *"lack of basic data structures"*. Take your time and improvise while your IDE starts.

## ...Ruby
If you come from Ruby and don't really like Go, I probably won't help you. Go is quite popular in Ruby community and many Ruby developers, being non-arrogant and pragmatic, fall into Go pretty easily, so you're in trouble. Even Basecamp, the guys that made Ruby on Rails, [love Go](https://signalvnoise.com/posts/3897-go-at-basecamp) and use it inside. Sorry.

## ...D
Without any doubts, your main argument (after generics and exceptions, of course) should be the name of Google. It's pretty much obvious that popularity of Go is simply a result of a huge money support by Google. Of course, Google pays people a lot to write articles about Go and to organize conferences and meetups. They are so rich, that next year Go community will have [6 international conferences](https://github.com/golang/go/wiki/Conferences) including one in [Dubai](http://www.gophercon.ae/)! When D have a company that can do the same for D, the world will understand that Go's popularity is a fake.

Also, Go is not a real system programming language. You can't write your own memory allocator with Go. So switch to D. Please.

## ...Perl
Emm.. 

## ...C\#
Probably the best strategy here is to attack the simplicity of Go. Simplicity equals primitivity, everyone knows in MS world. Also Go is made by Google, not by MS, so it's doomed. There are so many good solutions for modern programming language theory you learn in the college course of C#! But poor people behind Go just are not aware of them. Don't hesitate to teach them. You can't do pretty much anything with a language that primitive as Go.

Also, mention the most relevant things - no Visual Studio support ([Visual Studio Code's Go plugin](https://github.com/microsoft/vscode-go) doesn't count). No debugger in IDE, of course. And no DirectX support, that's important.

## ...Haskell
If you come from Haskell, I shouldn't give you any advice. You already must be a professional in mocking Go. It's in Haskell 101 course. New Haskell books contain special chapter "How to laugh on Go", after all.

Even if you [intuitively understand](https://honza.ca/2015/11/language-choice) that Go is way more practical than Haskell and entry barrier really matters - keep insisting that it has "objectively poor design". Because everyone knows which language has objectively good design.

Bonus points can be earned by [using](https://twitter.com/bhurt42/status/940629768126521344) thought-terminating cliché that Go authors think developers are not smart enough to understand brilliant language, which there only one and only. Benefits of using this soundbite "[were demonstrated over and over](https://twitter.com/puffnfresh/status/940829295290830849)".

# Conclusion
I hope this article will find its readers. In 2016, we need more articles with rants on Go - at least 6 to keep the trend. Some of the articles in the list above written by students and schoolboys, so if you just started CS class and don't have any real-life experience - don't hesitate to tell the world how bad Go is.

After all, the viral effect of the articles with criticism is well known - colleagues and managers send you the link as a prove that you shouldn't use Go in production, without even reading or analyzing its content. The title is usually enough, so don't be afraid.

Or.. you can just write some good code in the language that works best for your case.





