+++
date = "2018-12-01T16:55:00+02:00"
title = "Fountain codes and animated QR"
slug = "fountaincodes"
Description = "Using fountain codes to speed up animated QR data transfer"
Tags = ["golang", "coding theory", "fountain codes"]

+++

![fountain](/images/fountain.jpg) <sup>(source: [Anders Sune Berg](https://olafureliasson.net/archive/artwork/WEK110140/waterfall))</sup>

In the [previous article]({{< ref "/posts/181118_animated_qr" >}}) I've described a weekend project called [txqr](https://github.com/divan/txqr) for unidirectional data transfer using animated sequence of QR codes. Straightforward approach was to repeat the encoded sequence over and over until the receiver gets complete data. This simple repetition code was good enough for starter and trivial to implement, but also introduced long delays in case the receiver missed at least one frame. And in practice it often did, of course.

This sort of problems of sending data over a noisy communication channel is quite well studied and there is a whole theory exists to deal solve it – coding theory.

In the comments to previous article, [Bojtos Kiskutya](https://disqus.com/by/bojtoskiskutya/) mentioned LT codes that can yield much better results for *txqr*. That was exactly the type of comments I expect – not only suggestion for improvement, but also discovering some new and exciting stuff. And I never heard about LT codes, so next couple of days I spend reading everything I could find about them.

It turned out, [LT codes](https://en.wikipedia.org/wiki/Luby_transform_code) (**LT** stands for **L**uby **T**ransform) are just one of the implementations of the broader family of codes called [fountain codes](https://en.wikipedia.org/wiki/Fountain_code). It's a class of [erasure codes](https://en.wikipedia.org/wiki/Erasure_codes) that can produce potentially infinite amount of blocks from the source message blocks (K), and it's enough to receive slightly more than K encoded blocks to successfully decode the message. Receiver can start receiving blocks from any point, receive blocks in any order, with any erasure probability – fountain codes will work as soon as you received K+ different blocks. That's where name "fountain" comes from, actually – fountain's water drops represent encoded blocks, and you just have to fill the bucket with enough drops, without actually caring which drops exactly you get.

That was a perfect match for my case, so I quickly found Go based implementation [google/gofountain](https://github.com/google/gofountain), and replaced my naive repetition code with Luby transform code implementation. Results were quite impressive, and in this post I'll share details about LT algorithm, gotchas in using *gofountain* package and the final results with comparison with original ones.

# Fountain codes are damn cool

If you, like myself, have never heard about fountain codes before, don't worry – they are relatively new and solve very narrow and specific problem. They're also extremely cool. They beautifully harness properties of randomness, mathematical logic and clever probability distribution tuning to achieve their goal.

While I'm going to describe mostly LT codes, there are many other algorithms in this family – [Online codes](https://en.wikipedia.org/wiki/Online_codes), [Tornado codes](https://en.wikipedia.org/wiki/Tornado_codes), [Raptor codes](https://en.wikipedia.org/wiki/Raptor_code#Legal_complexity), etc, with Raptor codes being superior in almost every aspect, excluding a legal one. They seem to be heavily covered by patents and not widely implemented.

So the idea behind LT codes is relatively simple – encoder splits message into {{< col "#00A2FF" "source blocks" >}}, and continuously creates {{< col "green" "encoded blocks" >}} that consist of 1, 2 or more randomly selected {{< col "#00A2FF" "source blocks" >}} XOR-ed with each other. Block IDs used to construct each new {{< col "green" "encoded block" >}} are stored in it in an arbitrary way.

![lt encoder](/images/ltcodes.gif)

Decoder, in it's turn, collects all the {{< col "green" "encoded blocks" >}} (like a drops from a fountain)  – some consisting from just 1 {{< col "#00A2FF" "source block" >}}, some from 2 or more – and tries to restore new blocks by XORing them back with already decoded blocks.

So when decoder sees the {{< col "green" "encoded block" >}} with just 1 {{< col "#00A2FF" "source block" >}} – it does nothing else than adding it to the list of decoded blocks. Once it catches the block with 2 XOR-ed {{< col "#00A2FF" "source blocks" >}}, it checks the IDs, and if one of them is already in the list of decoded blocks – easily restore it due to the property of XOR. The same applies to decoding {{< col "green" "encoded blocks" >}} with more than 2  {{< col "#00A2FF" "source blocks" >}} – once you have all but one blocks decoded – just XOR it.

### Solition distribution

The coolness lays in choosing how to decide how many blocks should be encoded with just 1 {{< col "#00A2FF" "source block" >}}, with 2 or more. Too many single block packets and you loose the redundancy needed. Too many multiple-blocks packets – and it'll take too much time to get single blocks in a very noisy channel. That's what [Michael Luby](https://en.wikipedia.org/wiki/Michael_Luby), after which Luby codes are named, calls the [Solition Distribution](https://en.wikipedia.org/wiki/Soliton_distribution) - almost perfect distribution, which ensures that you have enough single block packets, definitely have **a lot** of 2-blocks packets and also a long tail of many-blocks packets up to N-block ones, where N is the total amount of {{< col "00A2FF" "source blocks" >}}.

![solition distribution](/images/solition.png)

Here is a closer look at the head of the distribution:

![solition distribution zoom](/images/solition_zoom.png)

You can see here some non-zero amount of single block packets, then 2-blocks packets take a fair share of all distribution (it's exactly a half to be precise), and the rest is distributed in descending way between the packets with higher number of XOR-ed blocks.

All that give LT codes a nice property of not depending on how often or in which pattern communication channel loses packets.

For my txqr project it means that fountain code should give on average much lower decoding time with any given encoding/transmission parameter.

# google/gofountain

The google's package gofountain implements several fountain codes in Go, including Luby transform code. It has [very little API](https://godoc.org/github.com/google/gofountain#NewBinaryCodec) (which is a good sign for library) – basically just `Codec` interface with a few implementations and `EncodeLTBlocks()` function, plus a few pseudorandom generators helpers.

However, I stuck trying to understand what does the second parameter of `EncodeLTBlocks()` is supposed to mean:

```go
func EncodeLTBlocks(message []byte, encodedBlockIDs []int64, c Codec) []LTBlock
```

Why would I want to provide block IDs to encoder, I don't even want to worry about blocks' properties, as it's an algorithm implementation and not the library user's concern. So my first guess was just to pass all block IDs – `1..N`.

It was close enough – debug output from tests produced blocks that looked like what I expected, but decoding had been never finishing.

I checked [GoDoc page for gofountain](https://godoc.org/github.com/google/gofountain#NewBinaryCodec) to see what other packages use it, and found only one open-source library for transmitting large files over lossy networks - [pump](https://github.com/sudhirj/pump) by [Sudhir Jonathan](https://github.com/sudhirj), I decided to leverage the power of friendly Gophers community and contacted Sudhir in Gophers slack, asking if he could help me to clarify those IDs usage. 

And that worked, Sudhir was extremely helpful and gave me an elaborate answer, which clarified all my doubts. The right way to use this library was to pass incremental IDs for `1..N*r` blocks, where N is total amount of source blocks, and r is a redundancy factor., which probably should lay in 1.4-2.0 range. 

As LT codes designed to provide limitless source of blocks, the proper usage would be to generate random chunks of IDs, and call `EncodeLTBlocks` in a loop, but as `txqr` API expected returning a concrete number of chunks to be endoded as QR, I decided not to break API for now and went with redundancy factor approach. So for `encodedBlockIDs` parameter I use simple helper function:

```go
// ids create slice with IDs for 0..n values.
func ids(n int) []int64 {
    ids := make([]int64, n)
    for i := int64(0); i < int64(n); i++ {
        ids[i] = i
    }
    return ids
}
```

and call it via:

```go
    codec := fountain.NewLubyCodec(N, rand.New(fountain.NewMersenneTwister(200)), solitonDistribution(N))

    idsToEncode := ids(int(N * e.redundancyFactor))
    lubyBlocks := fountain.EncodeLTBlocks(msg, idsToEncode, codec)
```

That's probably a needlessly boring part for the reader not interested in working with `gofountain`, but I hope it'll be helpful for someone struggling with its API, coming to this post via search results.

# Testing results

As I preserved the API of the initial package, the rest was a breeze. As you might remember from the [previous article]({{< ref "/posts/181118_animated_qr" >}}), I used the txqr Go package in both iOS app and web app called `txqr-tester`, running in the browser. That's where mindblowing crossplatform nature of Go payed off again! I just switched to my `fountain-codes` branch, with the new implementation of encoder and decoder, run `go generate` to execute both `gomobile` and `gopherjs` commands, and in a couple of seconds I had fountain codes implementation ready to use with Swift and in the browser.

I wonder if there any other language that can do this?

I launched my test setup, consisting from phone on a tripod and external monitor, configured testing parameters and launched automated testing, which lasted nearly half a day. This time I didn't change QR error recovery levels to save time, as they seem to have negligible effect on the result.

The results were more than impressive.

The record time for transferring ~13KB of data is now is half a second or **501ms** to be precise – it's almost 25kbps. This record was set for 12FPS and 1850 bytes per QR code with low error correction level. The variance of the time needed to decode plummeted significantly as there were no "expect the whole loop iteration" part as with repetition codes. Compare decoding times histogram between {{< col "orange" "repetition code">}} and {{< col "lime" "fountain code">}}:

{{< plotly 15 "time_histogram">}}

As you can see, most of the decoding attempts, with different values for FPS and chunk size, are still concentrated in the lower part of time axis – most of them less them 4 seconds.

And here is more detailed results breakdown:

{{< plotly 16 "time_vs_size">}}

Results were so good, so I decided to run tests with chunk sizes greater than 1000 bytes - up to 2000 bytes. And that brought me interesting results – there were a lot of decoding timeouts with chunk sizes between 1400 and 1700, but 1800-2000 bytes actually showed one of the bests results so far:

{{< plotly 7 "time_vs_size_2k">}}

FPS effect seems to be even more negligible this time, but showed much better results with in all values overall, and I even was able to increase upper cap to 15FPS:

{{< plotly 9 "time_vs_fps">}}

Here is a complete interactive 3D interactive graph with results:

{{< plotly 18 "3d_results">}}

# Conclusion

Fountain codes are definitely an exciting thing to work with. Being non-trivial, but simple, narrow scoped, but extremely useful, clever and fast, they're definitely fall into the "cool algorithms" bucket. They're one of those algorithms that can give you an awe once you understand how they work.

For txqr they offered dramatic performance and reliability improvements, and I'm looking forward to play with more efficient algorithms than LT codes.

Gomobile and Gopherjs, again, showed their awesomeness by decreasing the hassle of using already written and tested code in browser and mobile platforms to a lowest possible minimum.

# Links

- [Wikipedia: LT Codes](https://en.wikipedia.org/wiki/Luby_transform_code)
- [Wikipedia: Fountain Codes](https://en.wikipedia.org/wiki/Fountain_code)
- [Damn Cool Algorithms: Fountain Codes by Nick Johnson](http://blog.notdot.net/2012/01/Damn-Cool-Algorithms-Fountain-Codes)
- [Introduction to fountain codes: LT codes with Python by François Andrieux](https://franpapers.com/en/algorithmic/2018-introduction-to-fountain-codes-lt-codes-with-python/)
- [Michael Luby - Fountain Codes (video, 2004)](https://www.youtube.com/watch?v=s3lrmBczBTc)

