---
title: Fun with videomixer
short-description: using gstreamer-editing-services to make the world a better place.
...

# Introduction

When you've spent the whole week painstakingly fixing bugs, coding something just for fun is a welcome breath of fresh air.
These last weeks, one of my areas of work has been the gstreamer "videomixer" element. It needed some love, and still needs,
but I've been able to fix some of the issues we had.
When we first ported gstreamer-editing-services and gnonlin to gstreamer 1.0, even the most basic editing became impossible.
That was quite frustrating to say the least, and being able to do edition once again feels extremely good !

One of the great things with the extraction of pitivi's edition code to GES is that you can now write fancy scripts to make
automated edition, and with a little luck you won't encounter a bug on your way.
At the end of this article, you will find a video showing an example result.

There haven't been much tutorials about using GES, the only way to learn that is either looking at the examples on the git repo,
or to directly look at pitivi's source code.
With that blogpost I'm gonna try to present that cool library, while coding something fun. The idea from the script came from
that video : http://vimeo.com/35770492, linked to me by Nicolas Dufresne. We won't be able to reproduce the most advanced
bits of this video, as it also seems to be content-aware at some points, but we will make a fun script nevertheless !

## Sounds sweet, where's the code ?

``` python
from gi.repository import GstPbutils
from gi.repository import Gtk
from gi.repository import Gst
from gi.repository import GES
from gi.repository import GObject

import sys
import signal

def handle_sigint(sig, frame):
    Gtk.main_quit()

def busMessageCb(unused_bus, message):
    if message.type == Gst.MessageType.EOS:
        print "eos"
        Gtk.main_quit()

def duration_querier(pipeline):
    print pipeline.query_position(Gst.Format.TIME)
    return True

def mylog(x):
    return (x / (1 + x))

def createLayers(timeline, asset):
    step = 1.0 / int(sys.argv[2])
    alpha = step
    for i in range(int(sys.argv[2])):
        layer = timeline.append_layer()
        clip = layer.add_asset(asset, i * Gst.SECOND * 0.3, 0, asset.get_duration(), GES.TrackType.UNKNOWN)
        for source in clip.get_children(False):
            if source.props.track_type == GES.TrackType.VIDEO:
                break

        source.set_child_property("alpha", alpha)
        alpha += step

if __name__ =="__main__":
    if len(sys.argv) < 4:
        print "usage : " + sys.argv[0] + " file:///video/uri number_of_layers file:///audio/uri [file:///output_uri]"
        print "If you specify a output uri, the pipeline will get rendered"
        exit(0)

    GObject.threads_init()
    Gst.init(None)
    GES.init()

    timeline = GES.Timeline.new_audio_video()

    asset = GES.UriClipAsset.request_sync(sys.argv[1])
    audio_asset = GES.UriClipAsset.request_sync(sys.argv[3])

    createLayers(timeline, asset)

    timeline.commit()

    layer = timeline.append_layer()

    layer.add_asset(audio_asset, 0, 0, timeline.get_duration(), GES.TrackType.AUDIO)

    pipeline = GES.Pipeline()
    pipeline.set_timeline(timeline)

    container_profile = \
        GstPbutils.EncodingContainerProfile.new("pitivi-profile",
                                                "Pitivi encoding profile",
                                                Gst.Caps("video/webm"),
                                                None)

    video_profile = GstPbutils.EncodingVideoProfile.new(Gst.Caps("video/x-vp8"),
                                                        None,
                                                        Gst.Caps("video/x-raw"),
                                                        0)

    container_profile.add_profile(video_profile)

    audio_profile = GstPbutils.EncodingAudioProfile.new(Gst.Caps("audio/x-vorbis"),
                                                        None,
                                                        Gst.Caps("audio/x-raw"),
                                                        0)

    container_profile.add_profile(audio_profile)

    if len(sys.argv) > 4:
        pipeline.set_render_settings(sys.argv[4], container_profile)
        pipeline.set_mode(GES.PipelineFlags.RENDER)

    pipeline.set_state(Gst.State.PLAYING)

    bus = pipeline.get_bus()
    bus.add_signal_watch()
    bus.connect("message", busMessageCb)
    GObject.timeout_add(300, duration_querier, pipeline)

    signal.signal(signal.SIGINT, handle_sigint)
    Gtk.main()
```

## Looks fine, explain it now !

I'll select the meaningful bits, assuming you know python well enough. If not, this is easily translatable to C,
or any language that can take advantage of GObject introspection's dynamic bindings.

First, let's look at the main.

```
    timeline = GES.Timeline.new_audio_video()
```

This convenience function will create a timeline with an audio and a video track for us.

```
    asset = GES.UriClipAsset.request_sync(sys.argv[1])
    audio_asset = GES.UriClipAsset.request_sync(sys.argv[3])
```

This is part of the new API. Thanks to that, GES will only discover the file once, discovering meaning learning
what streams are contained in the media, how long it lasts and other infos. Previously, we would discover
the file each time we created an object with it, which was not optimized. request_sync is not what you would use
in a GUI application, instead you would want to request_async, then take action in a callback.

Now, let's look at createLayers, which is where the magic happens.

```
    layer = timeline.append_layer()
```

A timeline is a stack of layers, with ascending "priorities". Thanks to these layers, we are able for example
to decide if a transition has to be created between two track objects, or, if two clips have an alpha of 1.0,
which one will be the "topmost" one.

```
    clip = layer.add_asset(asset, i * Gst.SECOND * 0.3, 0, asset.get_duration(), GES.TrackType.UNKNOWN)
```

This code is very interesting. We are basically asking GES to : create a clip based on the asset we 
discovered earlier, set its start a i * 0.3 seconds, its inpoint (the place in the file from which it will
be played) to 0, and its duration to the original duration of the file.
The last argument means : for every kind of stream you find, add it if the timeline contains an 
appropriate track (here, audio and video).
We could have decided to only keep the VIDEO, but that was a good occasion to show that.

With that logic, we can now see that the resulting timeline is gonna be sort of a "canon":
one video mixed with n earlier versions of itself.

```
    for source in clip.get_children():
        if source.props.track_type == GES.TrackType.VIDEO:
        break
 
    source.set_child_property("alpha", alpha)
    alpha = mylog(alpha)
```

Here, I browse the children of my timeline element, and when I find a video element, I set the
alpha of an element inside it, and update the alpha. The log here makes it so each layer
has the same perceived opacity at the end.

Afterwards, we create a pipeline to play our timeline, and if needed we set it to the render mode,
that code is quite self-explanatory.

We now just have to wait until the EOS, or until the user interrupts the program.

I use Gtk.main() out of pure laziness, a GLib mainloop would work as well.

## How does it look like then ?

I really hope this example made you want to learn more about GES, it's a great library that lets
you do awesome stuff in very few lines of code, we're in active development and the best is still to come !

Here is the promised video:

<iframe width="640" height="360" src="http://www.youtube.com/embed/grTxE6sFIJM?feature=player_detailpage" frameborder="0"> </iframe>

Notice the code was only tried with mp4 containing h264, feel free to report any issues with other codecs on my github !
