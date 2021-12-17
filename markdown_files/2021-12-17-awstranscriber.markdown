---
title: awstranscriber
short-description: "Live Speech To Text with GStreamer and AWS"
github-issue-id: 7
feedgen-published: 2021-12-18
...

# awstranscriber, a GStreamer wrapper for AWS Transcribe API

If all you want to know is how to use the element, you can head over [here].

I actually implemented this element over a year ago, but never got around to
posting about it, so this will be the first post in a series about speech-to-text,
text processing and closed captions in GStreamer.
 
Speech-to-text has a long history, with multiple open source libraries implementing
a variety of approaches for that purpose<sup><a href="#stt-libs">\[1]</a></sup>,
but they don't necessarily offer either the same accuracy or ease of use as proprietary
services such as [Amazon's Transcribe API].

My overall goal for the project, which `awstranscriber` was only a part of, was the
ability to generate a transcription for live streams and inject it into the video
bitstream or carry it alongside.

The main requirements were to keep it as synchronized as possible with the content,
while keeping latency in check. We'll see how these requirements informed the design
of some of the elements, in particular when it came to closed captions.

My initial intuition about text was, to quote a famous philosopher: "*How hard can
it be?*"; turns out the answer was "**actually more than I would have hoped**".

<p id=stt-libs>
    <sup>[1] <code>pocketsphinx</code>, <code>Kaldi</code> just to name a few</sup>
</p>

[Amazon's Transcribe API]: https://aws.amazon.com/transcribe/
[here]: 2021-12-17-awstranscriber.markdown#quick-example

## The element

In GStreamer terms, the `awstranscriber` element is pretty straightforward: take
audio in, push timed text out.

The Streaming API for AWS is (roughly) synchronous: past a 10 second buffer duration,
the service will only consume audio data in real time, I thus decided to make
the element a live one by:

* synchronizing its input to the clock
* returning `NO_PREROLL` from its state change function
* reporting a latency

Event handling is fairly light: The element doesn't need to handle seeks in
any particular manner, only consumes and produces fixed caps, and can simply
disconnect from and reconnect to the service when it gets flushed.

As the element is designed for a live use case with a fixed maximum latency,
it can't wait for complete sentences to be formed before pushing text out. And
as one intended consumer for its output is closed captions, it also can't just
push the same sentence multiple times as it is getting constructed, because that would
completely overflow the CEA 608 bandwidth (more about that in later blog posts, but
think roughly 2 characters per video frame maximum).

Instead, the goal is for the element to push one word (or punctuation symbol)
at a time.

## Initial implementation

When I initially implemented the element, the Transcribe API had a pretty
significant flaw for my use case: while it provided me with "partial" results,
which sounded great for lowering the latency, there was no way to identify
partial results between messages.

Here's an illustration (this is just an example, the actual output is more
complex).

After feeding five seconds of audio data to the service, I would receive a first
message:

``` JSON
{
  words: [
    {
      start_time: 0.5,
      end_time: 0.8,
      word: "Hello",
    }
  ]

  partial: true,
}
```

Then after one more second I would receive:

``` JSON
{
  words: [
    {
      start_time: 0.5,
      end_time: 0.9,
      word: "Hello",
    },
    {
      start_time: 1.1,
      end_time: 1.6,
      word: "World",
    }
  ]

  partial: true,
}
```

and so on, until the service decided it was done with the sentence and started a
new one. There were multiple problems with this, compounding each other:

* The service seemed to have no predictable "cut-off" point, that is it would sometimes
  provide me with 30-second long sentences before considering it finished (`partial: false`)
  and starting a new one.

* As long as a result was partial, the service could change *any* of the words it had
  previously detected, even if they were first reported 10 seconds prior.

* The actual timing of the items could also shift (slightly)

This made the task of outputting one word at a time, just in time to honor the user-provided
latency, seemingly impossible: as items could not be strictly identified from one
partial result to the next, I could not tell whether a given word whose end time matched with
the running time of the element had already been pushed or had been replaced with
a new interpretation by the service.

Continuing with the above example, and admitting a 10-second latency, I could
decide at 9 seconds running time to push "Hello", but then receive a new partial
result:

``` JSON
{
  words: [
    {
      start_time: 0.5,
      end_time: 1.0,
      word: "Hey",
    },
    {
      start_time: 1.1,
      end_time: 1.6,
      word: "World",
    },
    ...
  ]

  partial: true,
}
```

What to then do with that "Hey"? Was it a new word that ought to be pushed?
An old one with a new meaning arrived too late that ought to be discarded?
Artificial intelligence attempting first contact?

Fortunately, after some head scratching and ~~some~~lots of blankly looking at the
JSON, I noticed a behavior which while undocumented seemed to always hold true:
while any feature of an item could change, the start time would never grow past
its initial value.

Given that, I finally managed to write some quite convoluted code that ended up
yielding useful results, though punctuation was very hit and miss, and needed some
more complex conditions to (sometimes) get output.

You can still see that code in all its glory [here], I'm happy to say that it
is gone now!

[here]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs/-/blob/5439f14e57ad865e8e19b45ac191ed98743d8e2b/net/rusoto/src/aws_transcriber/imp.rs#L201-392

## Second iteration

Supposedly, you always need to write a piece of code three times before it's
good, but I'm happy with two in this case.

6 months ago or so, I stumbled upon an innocuously titled [blog post] from
AWS' machine learning team:

> Improve the streaming transcription experience with Amazon Transcribe partial results stabilization

And with those few words, all my problems were gone!

In practice when this feature is enabled, the individual words that form a
partial result are explicitly marked as stable: once that is the case, they will
no longer change, either in terms of timing or contents.

Armed with this, I simply removed all the ugly, complex, scarily fragile code
from the previous iteration, and replaced it all with a single, satisfyingly
simple `index` variable: when receiving a new partial result, simply push all
words from `index` to `last_stable_result`, update `index`, done.

The output was not negatively impacted in any way, in fact now the element
actually pushes out punctuation reliably as well, which doesn't hurt.

I also exposed a property on the element to let the user control how
aggressively the service actually stabilizes results, offering a
trade-off between latency and accuracy.

[blog post]: https://aws.amazon.com/blogs/machine-learning/amazon-transcribe-now-supports-partial-results-stabilization-for-streaming-audio/

## Quick example

If you want to test the element, you'll need to build [gst-plugins-rs]<sup><a href="#gst-rusoto-tip">\[1]</a></sup>,
set up an AWS account, and [obtain credentials] which you can either
store in a credentials file, or provide as environment variables to
[rusoto].

Once that's done, and you have installed the plugin in the right place or set
the `GST_PLUGIN_PATH` environment variable to the directory where the plugin
got built,you should be able to run such a pipeline:

``` shell
gst-launch-1.0 uridecodebin uri=https://storage.googleapis.com/www.mathieudu.com/misc/chaplin.mkv name=d d. ! audio/x-raw ! queue ! audioconvert ! awstranscriber ! fakesink dump=true
```

Example output:

``` shell
Setting pipeline to PAUSED ...
Pipeline is live and does not need PREROLL ...
Pipeline is PREROLLED ...
Setting pipeline to PLAYING ...
New clock: GstSystemClock
Redistribute latency...
Redistribute latency...
Redistribute latency...0.0 %)
00000000 (0x7f7618011a80): 49 27 6d                                         I'm             
00000000 (0x7f7618011ac0): 73 6f 72 72 79                                   sorry           
00000000 (0x7f7618011b00): 2e                                               .               
00000000 (0x7f7618011e10): 49                                               I               
00000000 (0x7f76180120c0): 64 6f 6e 27 74                                   don't           
00000000 (0x7f7618012100): 77 61 6e 74                                      want            
00000000 (0x7f76180127a0): 74 6f                                            to              
00000000 (0x7f7618012c70): 62 65                                            be              
00000000 (0x7f7618012cb0): 61 6e                                            an              
00000000 (0x7f7618012d70): 65 6d 70 65 72 6f 72                             emperor         
00000000 (0x7f7618012db0): 2e                                               .               
00000000 (0x7f7618012df0): 54 68 61 74 27 73                                That's          
00000000 (0x7f7618012e30): 6e 6f 74                                         not             
00000000 (0x7f7618012e70): 6d 79                                            my              
00000000 (0x7f7618012eb0): 62 75 73 69 6e 65 73 73                          business
```

I could probably recite that whole "The dictator" speech by now by the
way, one more clip that is now ruined for me. The [predicaments] of multimedia
engineering!

`gst-inspect-1.0 awstranscriber` for more information on its properties.

<p id=gst-rusoto-tip>
    <sup>[1] you don't need to build the entire project, but instead just<code>cd /net/rusoto</code>
    before running <code>cargo build</code>
    </sup>
</p>

[predicaments]: https://www.youtube.com/watch?v=aqz-KE-bpKQ

## Thanks

* Sebastian Dröge at Centricular (gst Rust goodness)

* Jordan Petridis at Centricular (help with the initial implementation)

* [cablecast] for sponsoring this work!

[cablecast]: https://www.cablecast.tv/

## Next

In future blog posts, I will talk about closed captions, probably make
a few mistakes in the process, and explain why text processing isn't
necessarily all that easy.

Feel free to comment if you have issues, or actually end up implementing
interesting stuff using this element!

[gst-plugins-rs]: https://gitlab.freedesktop.org/gstreamer/gst-plugins-rs
[obtain credentials]: https://console.aws.amazon.com/transcribe/
[rusoto]: https://github.com/rusoto/rusoto/blob/master/AWS-CREDENTIALS.md
