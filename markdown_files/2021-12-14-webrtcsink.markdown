---
title: webrtcsink
short-description: "A new GStreamer element for WebRTC streaming"
github-issue-id: 5
...

# webrtcsink, a new GStreamer element for WebRTC streaming

`webrtcsink` is an all-batteries included GStreamer WebRTC producer, that tries
its best to do The Right Thing™.

Following up on the last part of my [last blog post], I have spent some time
these past few months working on a [WebRTC sink element] to make use of the
various mitigation techniques and congestion control mechanisms currently
available in [GStreamer].

This post will briefly present the implementation choices I made, the current
features and my ideas for future improvements, with a short demo at the end.

Note that `webrtcsink` requires latest GStreamer main at the time of writing,
all required patches will be part of the 1.20 release.

[last blog post]: 2020-10-09-SMPTE-2022-1-2D-Forward-Error-Correction-in-GStreamer.markdown
[WebRTC sink element]: https://github.com/centricular/webrtcsink
[GStreamer]: https://gstreamer.freedesktop.org/

## The element

The choice I made here was to make this element a simple sink: while it wraps
[webrtcbin], which supports both sending and receiving media streams, webrtcsink
will only offer sendonly streams to its consumers.

The element, unlike `webrtcbin`, only accepts raw audio and video streams, and
takes care of the encoding and payloading itself.

Properties are exposed to let the application control what codecs are offered
to consumers (and in what order), for instance `video-caps=video/x-vp9;video/x-vp8`,
and the choice of the actual encoders can be controlled through the GStreamer
feature rank mechanism.

This decision means that `webrtcsink` has direct control over the encoders,
in particular it can update their target bitrate according to network conditions,
more on that [later].

[webrtcbin]: https://gstreamer.freedesktop.org/documentation/webrtc/index.html
[later]: 2021-12-14-webrtcsink.markdown#congestion-control

## Signalling

Applications that use `webrtcsink` can implement their own signalling mechanism,
by implementing a [rust API], the element however comes with its own default
signalling protocol, implemented by the default signaller alongside a standalone
signalling server script, written in python.

The protocol is based on the protocol from the gst-examples, extended to support
a 1 producer -> N consumers configuration, it is admittedly a bit ugly but does
the job, I have plans for improving this, see [Future prospects].

[rust API]: https://github.com/centricular/webrtcsink/blob/main/plugins/src/webrtcsink/mod.rs#L16
[Future prospects]: 2021-12-14-webrtcsink.markdown#future-prospects

## Congestion control

`webrtcsink` makes use of the statistics it gathers thanks to the [transport-cc]
RTP extension in order to modulate the target bitrate produced by the video encoders
when congestion is detected on the network.

The heuristic I implemented is a hybrid of a Proof-of-Concept Matthew Waters
[implemented] recently and the [Google Congestion Control algorithm].

As far as my synthetic testing has gone, it works decently and is fairly
reactive, it will however certainly evolve in the future as more real-life
testing happens, more on that later.

[transport-cc]: https://datatracker.ietf.org/doc/html/draft-holmer-rmcat-transport-wide-cc-extensions-01
[implemented]: https://gitlab.freedesktop.org/ystreet/gst-examples/-/commits/bw-management
[Google Congestion Control algorithm]: https://datatracker.ietf.org/doc/html/draft-ietf-rmcat-gcc-02

## Packet loss mitigation techniques

`webrtcsink` will offer to honor retransmission requests, and will propose
sending ulpfec + red packets for Forward Error Correction on video streams.

The amount of FEC overhead is modified dynamically alongside the bitrate in
order not to cause the peer connection to suffer from self-inflicted wounds:
when the network is congested, sending *more* packets isn't necessarily the
brightest idea!

The algorithm to update the overhead is very naive at the moment, it could
be refined for instance by taking the roundtrip time into account: when that
time is low enough, retransmission requests will usually be sufficient for
addressing packet loss, and the element could reduce the amount of FEC packets
it sends out accordingly.

## Statistics monitoring

`webrtcsink` exposes the statistics from `webrtcbin` and adds a few of its
own through a property on the element.

I have implemented a simple server / client application as an [example],
the web application can plot a few handpicked statistics for any given
consumer, and turned out to be quite helpful as a debugging / development
tool, see [the demo video] for an illustration.

[example]: https://github.com/centricular/webrtcsink/tree/main/plugins/examples
[the demo video]: 2021-12-14-webrtcsink.markdown#demo

## Future prospects

In no particular order, here is a wishlist for future improvements:

* Implementing the default signalling server as a rust crate. This will allow
  running the signalling server either standalone, or letting `webrtcsink`
  instantiate it in process, thus reducing the amount of plumbing needed for
  basic usage. In addition, that crate would expose a trait to let applications
  extend the default protocol without having to reimplement their own.

* Sanitize the default protocol: at the moment it is an ugly mixture of JSON
  and plaintext, it does the job but could be nicer.

* More congestion control algorithms: at the moment the element exposes a property
  to pick the congestion control method, either `homegrown` or `disabled`,
  implementing more algorithms (for instance [GCC], [NADA] or [SCReAM]) can't hurt.

* Implementing [flexfec]: this is a longstanding wishlist item for me, ULP FEC
  has shortcomings that are addressed by flexfec, a GStreamer implementation would
  be generally useful.

* High-level integration tests: I am not entirely sure what those would look like,
  but the general idea would be to set up a peer connection from the element to
  various browsers, apply various network conditions, and verify that the output
  isn't overly garbled / frozen / poor quality. That is a very open-ended task
  because the various components involved can't be controlled in a fully
  deterministic manner, and the tests should only act as a robust alarm mechanism
  and not try to validate the final output at the pixel level.

[GCC]: https://datatracker.ietf.org/doc/html/draft-ietf-rmcat-gcc-02
[NADA]: https://datatracker.ietf.org/doc/html/draft-ietf-rmcat-nada
[SCReAM]: https://datatracker.ietf.org/doc/html/rfc8298
[flexfec]: https://datatracker.ietf.org/doc/html/draft-ietf-payload-flexible-fec-scheme

## Demo

<iframe width="560" height="315" src="https://www.youtube.com/embed/eJpxqVr_tzQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Thanks

This new element was made possible in part thanks to the contributions from

* Matthew Waters at Centricular (webrtcbin)

* Sebastian Droege at Centricular (GStreamer rust goodness)

* Olivier from Collabora (RTP stack)

* The good people at Pexip (RTP stack, transport-cc)

* [Sequence] for sponsoring this work

This is not an exhaustive list!

[Sequence]: https://sequence.film/
