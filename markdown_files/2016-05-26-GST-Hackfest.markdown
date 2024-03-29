---
title: Back from the GST hackfest
short-description: My experience at the GStreamer hackfest
feedgen-published: 2016-05-26
...

# Back from the GST hackfest

I was once again kindly sponsored by [Collabora](https://www.collabora.com/) to temporarily relocate my hacking environment to a place filled with awesome people.
This time the place was [Thessaloniki](https://en.wikipedia.org/wiki/Thessaloniki), and the people were GStreamer developers.

## Getting there

I hadn't taken any vacation since I've started working for Collabora, two years ago (mandatory "geez, time flies"), apart from one day or two for Christmas-related
activities, so I decided I'd take two weeks, and do a little road trip across Europe. Strangely enough my girlfriend, who when I went to the Staines (near London) hackfest
only asked when I'd come back, was already packing her swimsuit this time.

After a one-day stop in Venice (fortunately not too crowded at this time of year), and a three-day visit of [Dubrovnik](https://en.wikipedia.org/wiki/Dubrovnik) aka King's landing,
our intention was to get to Thessaloniki through Albania and arrive on the evening before the hackfest started. Albanian customs, however, didn't seem to share this intention at all,
and we ended up having to cross Montenegro from South to North, get in Serbia, go East and be refused entry in Kosovo, go East again and finally get to Thessaloniki through Macedonia.

Needless to say we arrived at 10 A.M. on the day the hackfest started.

I briefly logged in to the IRC channel, told everyone I'd have to be excused for the day, got greeted by Edward Hervey mumbling stuff about "equations everywhere", and
others suggesting I should have engaged in bribery (I will not name names here), logged off and slept for 15 hours.

## Second (actually my first) day

I arrived at the hackfest to find a nice, two-storey building reserved for our usage, and [Lubosz Sarnecki](https://lubosz.wordpress.com/) in the middle of
kilometers of cables, and enough hardware to open a small VR shop.
He needed a powerful GPU to set up the leapmotion's [blocks demo](https://developer.leapmotion.com/gallery/blocks), and we figured my laptop would
do the trick. We set everything up, then struggled for some time until accepting it really wouldn't be up to the task.

We ended up borrowing a desktop PC, which proved to be up to the task, and after some fiddling the demo was finally set up and working smoothly.

It was well worth the wait: leapmotion's tech is amazing, and after some time making and manipulating blocks, whoever isn't convinced about the future of VR might well
reconsider! I'm really curious to see the games people will come up with, a VR remake of [Black & White](https://www.youtube.com/watch?v=Ue4i44e_7W8) would
be pretty awesome if you ask me.

More to the point, Lubosz is currently writing a GStreamer video filter to transform streams for [Insert VR headset here] consumption, but as far as I understand
his work is still in progress, and not yet demoable.

It was already late, and after a few people had taken turns trying out the demo, the day was already gone.

## Third (actually my second) day

My original goal for this hackfest was to implement a hotdoc extension for documenting GStreamer plugins, but with one day left that was hardly
a completable goal, and after talking with Tim-Philipp Müller, it appeared he had more immediate concerns in terms of documentation, namely
porting existing content to markdown. I briefly talked him into sticking to the CommonMark specification, and using hotdoc to arrange it all
together. Olivier Crete started porting the contents of gstreamer.com, and I started work on the Plugin Writer's manual.

It was decided a new repository for the documentation would be created, and that we would update https://gstreamer.freedesktop.org/ incrementally
with its contents, once maintainers are happy with the proposed solution. I still need to finalize my port of the manual, hopefully by the end
of this week I'll have something decent to propose.

## Conclusion

I'm afraid I don't have much to report on the GStreamer side of the GStreamer hackfest, hopefully others will fill that gap, I'll update this page
with links to other people's posts.

* Sebastian wrote about [writing gstreamer elements in rust](https://coaxion.net/blog/2016/05/writing-gstreamer-plugins-and-elements-in-rust/)
