---
title: How to write GStreamer (1.0) elements in python
short-description: "Part 1: A test audio src"
github-issue-id: 2
...

# How to write GStreamer (1.0) elements in python

While it turns out writing meaningful elements using GStreamer through
[pygobject] was badly broken since [2014][boxed-bug], and it had never
been possible to [expose properties][properties-bug] on said elements
anyway, these minor details shouldn't stop us from leveraging some of
the unique and awesome packages at the disposal of the python developer
from GStreamer, and that's what we'll do in this series of posts.

## Disclaimer

Writing GStreamer elements in python is usually a terrible idea:

* Python is slow, actual data processing should be avoided at all cost,
  and instead delegated to C libraries such as [numpy], which is exactly what
  we'll do in this part.

* The infamous [GIL] enforces serialization, which means python elements will
  not be able to take advantage of the multithreading capabilities of modern
  platforms.

The only valid reasons for ignoring these restrictions are, to the best of
my knowledge:

* Python is the only language you know how to use.

* You want to use a python package that has no equivalent elsewhere, for
  example for scientific computing.

* Python rocks, and you don't intend to do anything CPU-intensive anyway.

* All of the above.

The obvious recommendation these days, if you do not want to deal with
low-level concerns such as data races and memory safety, is [Rust]. More
information can be found [here][gstreamer-rs-doc] and in
[this series of posts][gstreamer-rs-blogposts] from Sebastian Dröge.

## Some code right off the bat

``` python
import gi

gi.require_version('Gst', '1.0')
gi.require_version('GstBase', '1.0')
gi.require_version('GstAudio', '1.0')

from gi.repository import Gst, GLib, GObject, GstBase, GstAudio
import numpy as np

OCAPS = Gst.Caps.from_string (
        'audio/x-raw, format=F32LE, layout=interleaved, rate=44100, channels=2')

SAMPLESPERBUFFER = 1024

DEFAULT_FREQ = 440
DEFAULT_VOLUME = 0.8
DEFAULT_MUTE = False
DEFAULT_IS_LIVE = False

class AudioTestSrc(GstBase.BaseSrc):
    __gstmetadata__ = ('CustomSrc','Src', \
                      'Custom test src element', 'Mathieu Duponchelle')

    __gproperties__ = {
        "freq": (int,
                 "Frequency",
                 "Frequency of test signal",
                 1,
                 GLib.MAXINT,
                 DEFAULT_FREQ,
                 GObject.ParamFlags.READWRITE
                ),
        "volume": (float,
                   "Volume",
                   "Volume of test signal",
                   0.0,
                   1.0,
                   DEFAULT_VOLUME,
                   GObject.ParamFlags.READWRITE
                  ),
        "mute": (bool,
                 "Mute",
                 "Mute the test signal",
                 DEFAULT_MUTE,
                 GObject.ParamFlags.READWRITE
                ),
        "is-live": (bool,
                 "Is live",
                 "Whether to act as a live source",
                 DEFAULT_IS_LIVE,
                 GObject.ParamFlags.READWRITE
                ),
    }

    __gsttemplates__ = Gst.PadTemplate.new("src",
                                           Gst.PadDirection.SRC,
                                           Gst.PadPresence.ALWAYS,
                                           OCAPS)

    def __init__(self):
        GstBase.BaseSrc.__init__(self)
        self.info = GstAudio.AudioInfo()

        self.freq = DEFAULT_FREQ
        self.volume = DEFAULT_VOLUME
        self.mute = DEFAULT_MUTE

        self.set_live(DEFAULT_IS_LIVE)
        self.set_format(Gst.Format.TIME)

    def do_set_caps(self, caps):
        self.info.from_caps(caps)
        self.set_blocksize(self.info.bpf * SAMPLESPERBUFFER)
        return True

    def do_get_property(self, prop):
        if prop.name == 'freq':
            return self.freq
        elif prop.name == 'volume':
            return self.volume
        elif prop.name == 'mute':
            return self.mute
        elif prop.name == 'is-live':
            return self.is_live
        else:
            raise AttributeError('unknown property %s' % prop.name)

    def do_set_property(self, prop, value):
        if prop.name == 'freq':
            self.freq = value
        elif prop.name == 'volume':
            self.volume = value
        elif prop.name == 'mute':
            self.mute = value
        elif prop.name == 'is-live':
            self.set_live(value)
        else:
            raise AttributeError('unknown property %s' % prop.name)

    def do_start (self):
        self.next_sample = 0
        self.next_byte = 0
        self.next_time = 0
        self.accumulator = 0
        self.generate_samples_per_buffer = SAMPLESPERBUFFER

        return True

    def do_gst_base_src_query(self, query):
        if query.type == Gst.QueryType.LATENCY:
            latency = Gst.util_uint64_scale_int(self.generate_samples_per_buffer,
                    Gst.SECOND, self.info.rate)
            is_live = self.is_live
            query.set_latency(is_live, latency, Gst.CLOCK_TIME_NONE)
            res = True
        else:
            res = GstBase.BaseSrc.do_query(self, query)
        return res

    def do_get_times(self, buf):
        end = 0
        start = 0
        if self.is_live:
            ts = buf.pts
            if ts != Gst.CLOCK_TIME_NONE:
                duration = buf.duration
                if duration != Gst.CLOCK_TIME_NONE:
                    end = ts + duration
                start = ts
        else:
            start = Gst.CLOCK_TIME_NONE
            end = Gst.CLOCK_TIME_NONE

        return start, end

    def do_create(self, offset, length):
        if length == -1:
            samples = SAMPLESPERBUFFER
        else:
            samples = int(length / self.info.bpf)

        self.generate_samples_per_buffer = samples

        bytes_ = samples * self.info.bpf

        next_sample = self.next_sample + samples
        next_byte = self.next_byte + bytes_
        next_time = Gst.util_uint64_scale_int(next_sample, Gst.SECOND, self.info.rate)

        if not self.mute:
            r = np.repeat(
                    np.arange(self.accumulator, self.accumulator + samples),
                    self.info.channels)
            data = ((np.sin(2 * np.pi * r * self.freq / self.info.rate) * self.volume)
                    .astype(np.float32))
        else:
            data = [0] * bytes_

        buf = Gst.Buffer.new_wrapped(bytes(data))

        buf.offset = self.next_sample
        buf.offset_end = next_sample
        buf.pts = self.next_time
        buf.duration = next_time - self.next_time

        self.next_time = next_time
        self.next_sample = next_sample
        self.next_byte = next_byte
        self.accumulator += samples
        self.accumulator %= self.info.rate / self.freq

        return (Gst.FlowReturn.OK, buf)


__gstelementfactory__ = ("audiotestsrc_py", Gst.Rank.NONE, AudioTestSrc)
```

## Discussion

To make that element available, assuming [gst-python] is installed, the code
above needs to be placed in a `python` directory, anywhere gstreamer will look
for plugins (eg `GST_PLUGIN_PATH`):

``` bash
$ ls python/
srcelement.py
$ GST_PLUGIN_PATH=$GST_PLUGIN_PATH:$PWD gst-inspect-1.0 audiotestsrc_py
Factory Details:
  Rank                     none (0)
  Long-name                CustomSrc
[...]
```

At the moment of writing, the master branches from both [pygobject][pygobject-git]
and [gstreamer][gstreamer-git] need to be installed.

Let's study some of the interesting parts now.

### Imports

``` python
from gi.repository import Gst, GLib, GObject, GstBase, GstAudio
import numpy as np
```

Nothing unfamiliar here, assuming you've already done some pygobject
programming, note that unlike an application that would usually initialize
GStreamer here with a call to `Gst.init()`, we don't need to do that.

We will use numpy to generate samples in a reasonably efficient manner,
more on that below.

### Registration

Using gst-python, we implement new elements as python classes, which we
need to register with GStreamer. The python plugin loader implemented
by gst-python will import our module, and look for an attribute with
the well-known name `__gstelementfactory__`.

The value of this attribute should be a tuple consisting of a factory-name,
a rank, and the class that implements the element.

> If the module needs to register multiple elements, it can do so by
> assigning a tuple of such tuples instead.

``` python
__gstelementfactory__ = ("audiotestsrc_py", Gst.Rank.NONE, AudioTestSrc)
```

The class that we register is expected to hold a `__gstmetadata__` class
attribute:

``` python
class AudioTestSrc(GstBase.BaseSrc):
    __gstmetadata__ = ('CustomSrc','Src', \
                      'Custom test src element', 'Mathieu Duponchelle')
```

The contents of this tuple will be used to call
`gst_element_class_set_metadata`, you'll find more information in
its [documentation][meta-doc]

### Inheritance

``` python
class AudioTestSrc(GstBase.BaseSrc):
    # [...]
    def __init__ (self):
        GstBase.BaseSrc.__init__(self)
```

Our element will be a GStreamer source: it will not have any sink pads, and
will output data on a single source pad.

There is a base class in GStreamer for that type of elements,
[GstBase.BaseSrc][GstBaseSrc]. It handles state changes, supports live sources,
push and pull-mode scheduling, and more.

Inheritance is standard, the subclass needs to chain up in its `__init__`
function if it implements it.

Overriding virtual methods can be done by prefixing the name of the virtual
method as declared in C with `do_`, more on that later.

### Initialization

The `__init__` method should obviously only be called once over the lifetime
of the element.

This means we only need to initialize here those variables that will not
need to be reinitialized when the element switches states. We only declare
and (re)initialize other variables in the `do_start` virtual method
implementation.

Note that linters might complain when attributes are declared outside of
the `__init__` function, as we do in the `do_start` virtual method, if you
wish to strictly comply you will want to declare them in `__init__` as well,
we didn't do so here for the sake of brevity.

> As our base class declares a `start` vmethod, we implement it by defining
> a `do_start` method in our class.

### Capabilities, negotiation

In this example, we implement an element that will only output a single
format:

``` python
OCAPS = Gst.Caps.from_string (
        'audio/x-raw, format=F32LE, layout=interleaved, rate=44100, channels=2')

# [...]

class AudioTestSrc(GstBase.BaseSrc):
    # [...]

    __gsttemplates__ = Gst.PadTemplate.new("src",
                                           Gst.PadDirection.SRC,
                                           Gst.PadPresence.ALWAYS,
                                           OCAPS)
```

`__gsttemplates__` is another well-known name that the python plugin loader
will look up, it matches the arguments to
[`gst_pad_template_new`][template-doc]; here we declare that we will expose
a single source pad named "src" that will output data in the format specified
by `OCAPS`: 2 channels of interleaved float samples, at a rate of 44100 audio
frames (so 88200 samples) a second.

As that format is fixed, we won't have to concern ourselves with negotiation
in that element, this will be automatically handled by our parent classes.

``` python
    def do_set_caps(self, caps):
        self.info.from_caps(caps)
        self.set_blocksize (self.info.bpf * SAMPLESPERBUFFER)
        return True
```

We technically could have done this directly in `__init__`, as we already know
what the result of the negotiation will be, however if in the future we decided
to make things more dynamic, for example by supporting multiple sample formats,
the audio info would need to be initialized at the end of the negotiation
process, as we do here.

The next blog post in this series will present an element implementing dynamic
negotiation, a good exercise for the reader could be to port this element to
support a range of supported output channels, or a second sample format, eg
32-bit integers.

### Processing

We chose to generate samples in our `do_create` implementation for no
particular reason, the default implementation would call `do_alloc` then
`do_fill`, we should only have to implement the latter if we wished to
use that approach, as we have called `GstBase.BaseSrc.set_blocksize` in
our `set_caps` implementation.

I will not discuss the implementation details here, we generate an array
of float samples forming a sine wave using numpy, and keep track of where
the waveform was at in the `accumulator` attribute, this is all pretty simple
stuff.

We could of course generate the samples in a for loop, but performance would
be abysmal.

The interesting part here is that `GstBaseSrc` expects us to return a tuple
made of `(Gst.FlowReturn.OK, output_buffer)` if everything went well,
otherwise typically `(Gst.FlowReturn.ERROR, None)` if there was an issue
generating the data.

It is the responsability of the `create` vmethod implementation to .. create
the output buffer, which is just what we do with
`Gst.Buffer.new_wrapped (bytes(data))`.

### Properties

It is possible with pygobject to declare GObject properties with a decorator,
however if one wants to specify minimum, maximum or default values, or
provide some documentation, to be for example presented in the `gst-inspect`
output, one needs to use a more verbose form:

```
class AudioTestSrc(GstBase.BaseSrc):
    # [...]
    __gproperties__ = {
        "freq": (int,
                 "Frequency",
                 "Frequency of test signal",
                 1,
                 GLib.MAXINT,
                 DEFAULT_FREQ,
                 GObject.ParamFlags.READWRITE
                ),
        # [...]
    }

    # [...]

    def do_get_property(self, prop):
        if prop.name == 'freq':
            return self.freq

    [...]
    def do_set_property(self, prop, value):
        if prop.name == 'freq':
            self.freq = value
```

Some interesting improvements here could be to declare the freq property as
[controllable], or expose a property allowing to change the shape of the
waveform (sine, square, triangle, ...)

### Liveness

Three things are needed to output data in "live" mode:

* Calling `GstBase.BaseSrc.set_live(True)`

* Reporting the latency by handling the LATENCY query, which is what we do
  in `do_gst_base_src_query`. The attentive reader might have noticed that
  even though the `GstBaseSrc` virtual method is named `query`, we didn't
  implement it as `do_query`: that is because `GstElement` also exposes
  a virtual method with the same name, and we have to lift the ambiguity.
  Try implementing `do_query` and see what happens.

* Implementing `get_times` to let the base class know when it should actually
  push the buffer out.

Our element does all three things, and exposes a property named `is-live` to
control that behaviour, you can verify it as follows:

``` bash
GST_PLUGIN_PATH=$GST_PLUGIN_PATH:$PWD gst-launch-1.0 -v audiotestsrc_py ! \
fakesink silent=false
```

as opposed to:

``` bash
GST_PLUGIN_PATH=$GST_PLUGIN_PATH:$PWD gst-launch-1.0 -v audiotestsrc_py is-live=true ! \
fakesink silent=false
```

> In this context, we are not really producing data live, but simply simulating
> by having the base class wait

## Conclusion

We have implemented a simplified version of `audiotestsrc` here, the reader
can update the code to support more features and familiarize themselves with
the GstBaseSrc API, or alternatively try to implement a video test src.

In the next post, we will present a GstBaseTransform implementation, that
accepts audio as an input and outputs a plot generated with matplotlib. There
will be dynamic negotiation, decoupling of the input and output, and more
interesting things.

[pygobject]: https://pygobject.readthedocs.io/en/latest/
[boxed-bug]: https://gitlab.gnome.org/GNOME/pygobject/merge_requests/10
[properties-bug]: https://gitlab.gnome.org/GNOME/pygobject/merge_requests/8
[numpy]: http://www.numpy.org/
[GIL]: https://wiki.python.org/moin/GlobalInterpreterLock
[Rust]: https://www.rust-lang.org/en-US/
[gstreamer-rs-doc]: https://sdroege.github.io/rustdoc/gstreamer/gstreamer/
[gstreamer-rs-blogposts]: https://coaxion.net/blog/2018/01/how-to-write-gstreamer-elements-in-rust-part-1-a-video-filter-for-converting-rgb-to-grayscale/
[gst-python]: https://gstreamer.freedesktop.org/modules/gst-python.html
[pygobject-git]: https://gitlab.gnome.org/GNOME/pygobject
[gstreamer-git]: https://cgit.freedesktop.org/gstreamer/gstreamer
[meta-doc]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstElement.html#gst-element-class-set-metadata
[GstBaseSrc]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer-libs/html/GstBaseSrc.html
[template-doc]: https://gstreamer.freedesktop.org/data/doc/gstreamer/head/gstreamer/html/GstPadTemplate.html#gst-pad-template-new
[controllable]: https://gstreamer.freedesktop.org/documentation/application-development/advanced/dparams.html
