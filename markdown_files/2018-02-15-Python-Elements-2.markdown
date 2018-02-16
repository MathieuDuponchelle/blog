---
title: How to write GStreamer (1.0) elements in python (Part II)
short-description: "An audio plotter"
github-issue-id: 3
...

# Implementing an audio plotter

In the [previous post], I presented a test audio source, and used it to
illustrate basic `gst-python` concepts and present the `GstBase.BaseSrc` base
class.

This post assumes familiarity with said concepts, it will expand on some more
advanced topics, such as caps negotiation, present another base class,
[GstBase.BaseTransform], and a useful object, [GstAudio.AudioConverter].

The example element will accept any sort of audio input on its sink pad,
and output a waveform as a series of raw video frames. The output framerate
and resolution will not be fixed, and instead negotiated with downstream
elements.

## Example result

The following video was generated with:

```
gst-launch-1.0 matroskamux name=mux ! progressreport ! filesink location=out.mkv \
compositor name=comp background=black \
sink_0::zorder=1 sink_0::ypos=550 sink_1::zorder=0 ! \
videoconvert ! x264enc tune=zerolatency bitrate=15000 ! queue ! mux. \
uridecodebin uri=file:/home/meh/devel/gst-build/python-plotting.mp4 name=dec ! \
audio/x-raw ! tee name=t ! queue ! audioconvert ! audioresample ! volume volume=10.0 ! \
volume volume=10.0 ! audioplot window-duration=3.0 ! video/x-raw, width=1280, height=150 ! \
comp.sink_0 \
t. ! queue ! audioconvert ! audioresample ! opusenc ! queue ! mux. \
dec. ! video/x-raw ! videoconvert ! deinterlace ! comp.sink_1
```

<iframe width="770" height="433" src="http://www.youtube.com/embed/o3hjosK1sRQ?feature=player_detailpage" frameborder="0"> </iframe>

This is the video most related to python plotting I could find
<sup><small> please don't stone me</small></sup>

## Implementation

{{audioplot.py}}

## Discussion

At the moment of writing, the master branches from both [pygobject][pygobject-git]
and [gstreamer][gstreamer-git] need to be installed.

The python libraries we will use for the purpose of plotting are [matplotlib]
and [numpy_ringbuffer] to help decoupling our input and output. Both are
installable with `pip`:

``` shell
python3 -m pip install matplotlib numpy_ringbuffer
```

You can test the element as follows:

``` bash
$ ls python/
audioplot.py
$ GST_PLUGIN_PATH=$GST_PLUGIN_PATH:$PWD gst-launch-1.0 audiotestsrc ! \
audioplot window-duration=0.01 ! videoconvert ! autovideosink
```

### Caps negotiation

#### Pad templates

For our audio test source example, I chose to implement the simplest form of
[caps negotiation]\: fixed negotiation. The element stated that it would output
a specific format on its source pad, and its base classes handled the rest.

For this example however, the element will accept a wide range of input formats,
and propose a wide range of output formats as well:

{{audioplot.py[19:31]}}

{{audioplot.py[43:51]}}

Let's see what `gst-inspect-1.0` tells us about its pad templates here:

``` json
Pad Templates:
  SINK template: 'sink'
    Availability: Always
    Capabilities:
      audio/x-raw
                 format: { (string)S8, (string)U8, (string)S16LE, (string)S16BE, (string)U16LE, (string)U16BE, (string)S24_32LE, (string)S24_32BE, (string)U24_32LE, (string)U24_32BE, (string)S32LE, (string)S32BE, (string)U32LE, (string)U32BE, (string)S24LE, (string)S24BE, (string)U24LE, (string)U24BE, (string)S20LE, (string)S20BE, (string)U20LE, (string)U20BE, (string)S18LE, (string)S18BE, (string)U18LE, (string)U18BE, (string)F32LE, (string)F32BE, (string)F64LE, (string)F64BE }
                 layout: interleaved
                   rate: [ 1, 2147483647 ]
               channels: [ 1, 2147483647 ]

  SRC template: 'src'
    Availability: Always
    Capabilities:
      video/x-raw
                 format: ARGB
                  width: [ 1, 2147483647 ]
                 height: [ 1, 2147483647 ]
              framerate: [ 1/1, 2147483647/1 ]
```

The element states that it can accept any audio format, with any rate and any
number of channels, the only restriction that we place is that samples should
be interleaved.

On the output side, once again we place a single restriction, and state that
the element will only be able to output ARGB data, this because ARGB is the
only alpha-capable pixel format the matplotlib API we will use proposes.

#### Virtual methods

When inheriting from [Gst.Element], negotiation is implemented by receiving
and sending [events] and [queries] on the pads of the element.

However, most if not all other GStreamer base classes take care of this
aspect, and instead let their subclasses optionally implement a set of
virtual methods adapted to the base class' purpose.

In the case of BaseTransform, the base class assumes that input and output
caps will depend on each other: imagine an element that would crop a video
by a set number of pixels, it is easy to see that the resolution of the
output will depend on that of the input.

With that in mind, the virtual method we need to expose is the aptly-named
`do_transform_caps`:

{{audioplot.py[146:156]}}

In our case, there is no dependency between input and output: receiving
audio with a given sample format will not cause it to output video in a
different resolution.

Consequently, when asked to transform the caps of the sink pad, we simply
need to return the template of the source pad, potentially intersected
with the optional `filter` argument (this parameter is useful for reducing
the complexity of the overall negotiation process).

An example of an element where input and output are interdependent
is [videocrop].

Implementing this virtual method is enough to make negotiation succeed
if upstream and downstream elements have compatible capabilities, but
if for example downstream also accepts a wide range of resolutions, the
default behaviour of the base class will be to pick the smallest possible
resolution.

This behaviour is known as `fixating` the caps, and `BaseTransform` exposes
a virtual method to let the subclass pick a sane default value in such cases:

{{audioplot.py[157:170]}}

We do not have a preferred input format, and as a consequence we use the
default `caps.fixate` implementation.

However if for example the element is offered to output its full resolution range,
we are going to try and pick the resolution closest to our preferred default,
this is what the calls to `fixate_field_nearest_int` achieve.

This will have no effect if the field is already fixated to a specific value.

If the field was set to a range *not* containing our preferred value, fixating
would result in picking the allowed value closest to it, for example given
our preferred width `640` and the allowed range `[800, 1200]`, the final value
of the field would be `800`:

``` bash
gst-launch-1.0 audiotestsrc ! audioplot window-duration=0.01 ! \
capsfilter caps="video/x-raw, width=[ 800, 1200 ]" ! videoconvert ! autovideosink
```

All that remains to do is for the element to initialize its state based on
the result of the caps negotiation:

``` python
    def do_set_caps(self, icaps, ocaps):
        in_info = GstAudio.AudioInfo()
        in_info.from_caps(icaps)
        out_info = GstVideo.VideoInfo()
        out_info.from_caps(ocaps)
	# [...]
```

The meat of that function is omitted due to its sausage factory nature, amongst
other things it creates a matplotlib figure with the correct size
(`set_size_inches` is one of the worst API I've ever seen), initializes some
counters, a ringbuffer, etc ..

### Converting the input

As I decided to support any sample format as the input, the most straightforward
(and reasonably performant) approach is to use [GstAudio.AudioConverter]:

``` python
        self.convert_info = GstAudio.AudioInfo()
        self.convert_info.set_format(GstAudio.AudioFormat.S32,
                                     in_info.rate,
                                     in_info.channels,
                                     in_info.position)
        self.converter = GstAudio.AudioConverter.new(GstAudio.AudioConverterFlags.NONE,
                                                     in_info,
                                                     self.convert_info,
                                                     None)
```

We initialize a converter based on our input format, as explained above this is
best done in `do_set_caps`:

``` python
        _, info = inbuf.map(Gst.MapFlags.READ)
        res, data = self.converter.convert(GstAudio.AudioConverterFlags.NONE,
                                            info.data)
        data = memoryview(data).cast('i')
```

By setting the required output format to `GstAudio.AudioFormat.S32`, we ensure
that the endianness of the converted samples will be the native endianness of
the platform the code runs on, which means that we can in turn cast our
memoryview to `'i'` (memoryview.cast doesn't let its user select an endianness).

The best alternative I'm aware of is possibly to use python's [struct] module
in combination with the pack and unpack functions exposed on
[GstAudio.AudioFormatInfo], however those are not yet available in the python
bindings.

### Decoupling input and output

The initial version of this element only implemented `do_transform`, and simply
plotted one output buffer per input buffer. This produced a kaleidoscopic effect
and slaved the framerate to `samplerate / samplesperbuffer`.

`BaseTransform` exposes a virtual method that allows producing 0 to N output
buffers per buffer instead, `do_generate_output`:

``` python
    def do_generate_output(self):
        inbuf = self.queued_buf
	# [...]
```

When a new buffer is chained on the sink pad, `do_generate_output` is called
repeatedly as long as it returns `Gst.FlowReturn.OK` and a buffer: thanks to that
we can fill our ringbuffer and only return a frame once we have processed
enough new samples to reach our next time. Conversely we can produce multiple
frames if the size of the input buffer warrants it.

Here again, the rest of the function is made up of implementation details,
an important point to note is that we still expose `do_transform`, as
`BaseTransform` assumes otherwise that the element will operate in passthrough
mode, which obviously creates some interesting problems.

## Conclusion

Some improvements could be made to this element:

* It could, instead of averaging channels, use one matplotlib figure per channel,
  and overlap them, to provide an output similar to audacity. This would however
  introduce a dependency between input and output formats!

* Styling is hardcoded, properties such as transparency, line-color, line-width,
  etc.. could be exposed.

* matplotlib is atrociously slow and not really meant for real-time usage. Some
  effort was made to optimize its usage (`blit`, `thinning_factor`), however
  performance is still disappointing. [vispy] might be an alternative worth
  exploring.

On a more positive note, it should be noted that while our previous element
had a more capable equivalent (`audiotestsrc`), this element does not really
have one, and its implementation is satisfyingly concise!

I don't have an idea yet for the next post in the series, the most interesting
scientific python packages I can think of are machine-learning ones such as
tensorflow, but I have no experience with these, ideally a new post should
also explore a different base class (GstAggregator, GstBaseSink?).

Suggestions welcome!

[previous post]: 2018-02-01-Python-Elements.markdown
[GstBase.BaseTransform]: https://lazka.github.io/pgi-docs/GstBase-1.0/classes/BaseTransform.html
[GstAudio.AudioConverter]: https://lazka.github.io/pgi-docs/GstAudio-1.0/classes/AudioConverter.html
[pygobject-git]: https://gitlab.gnome.org/GNOME/pygobject
[gstreamer-git]: https://cgit.freedesktop.org/gstreamer/gstreamer
[caps negotiation]: https://gstreamer.freedesktop.org/documentation/plugin-development/advanced/negotiation.html
[matplotlib]: https://matplotlib.org/
[numpy_ringbuffer]: https://pypi.python.org/pypi/numpy_ringbuffer/0.2.0
[Gst.Element]: https://lazka.github.io/pgi-docs/Gst-1.0/classes/Element.html
[events]: https://lazka.github.io/pgi-docs/Gst-1.0/classes/Event.html
[queries]: https://lazka.github.io/pgi-docs/Gst-1.0/classes/Query.html
[videocrop]: https://github.com/GStreamer/gst-plugins-good/blob/master/gst/videocrop/gstvideocrop.c#L631-L712
[GstAudio.AudioFormatInfo]: https://lazka.github.io/pgi-docs/GstAudio-1.0/classes/AudioFormatInfo.html
[audacity]: https://www.audacityteam.org/
[vispy]: http://vispy.org/
[struct]: https://docs.python.org/3/library/struct.html
