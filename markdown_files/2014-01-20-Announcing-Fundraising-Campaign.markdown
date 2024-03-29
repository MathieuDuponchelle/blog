---
title: Announcing pitivi's fundraising campaign !
short-description: We are launching a crowdfunding campaign, and you, yes *you* can help !
feedgen-published: 2014-01-20
...

# Our team

I've been a part of the Pitivi story on and off since three years now, whenever I could find time really. I've loved
every moment I've spent in the community, I've made friends, I've learnt good engineering practices, how to fit in a team, how
to communicate clearly, and so much more I can't even start to list.

Not gonna tell the whole story because that would be boring to write and even more to read, but eventually and naturally
I became a maintainer alongside Thibault and Jean-François. Jean-François has been around the project for 10 years now, he's awesome,
a really dedicated guy. Thibault and I are friends since a long time, well before I started programming, I've been his padawan for the
best part of my initiation to programming and Free Software, and he's a great Jedi !

Recently, Alexandru Balut has also started to work with us again, I don't know him as closely as I know Jeff and Thib, but he's
commited so much things in the last two months that we've had a hard time reviewing it and preparing the campaign at the same time !

I don't know him as well on a personal level as I know Jeff and Thibault, but I like working with him a lot, he makes great and
clean patches and has a seemingly boundless dedication to cleaning up the code and making it elegant.

All this to say I'm proud and happy to be part of such a team. Free Software's most important asset isn't the code, the bug
trackers, the continuous integration servers, it's the people, and these folks are great, I can't stress that enough.

# Our project

You might have guessed by now that our project isn't to grow genetically modified potatoes in the Southern part of Italy,
even though that seems like a compelling idea at first sight. Give it a second thought and you'll realize the hygrometry
of that region is absolutely not appropriate, give it a third one for good measure then forget about it.

I'll briefly explain what it is that makes of the Pitivi project the most exciting open source video editing project in my 
very humble opinion.
The reason goes down to our design choices. We've played the long game, and based ourselves on the gstreamer set of
libraries and plugins. For a visual and greatly simplified explanation of how that choice is a good thing, you can refer
to [this animation](http://fundraiser.pitivi.org/gstreamer), then have a look at the 
[impressive](http://gstreamer.freedesktop.org/documentation/plugins.html) list of plugins that we can tap into.
GStreamer is where most of the companies interested in open source multimedia invest their money and their time, it's where
most of the exciting stuff happens and it's definitely where an ambitious video editing application has to look at.

We also have made the choice of clearly splitting our editing core, our model, from our view, and made it a library,
with an awesome API, gstreamer-editing-services, directly usable from C, C++, javascript, python and every language
supported by introspection, and possibly any other language provided someone writes bindings for it.
That choice was the right one, decoupling components always pays off in the long term, and we are finally starting
to see the benefits of that choice: Pitivi has seen its size divided by two, while gaining in stability.

This makes it much easier for new contributors to come in, and for us to maintain it.

tl; dr: GStreamer rocks, and GES is great.

With that said, we are aware that the stabilization is not yet over. Pitivi is in a beta state, and it still needs intensive
work to make it so we kill the bugs and they never come back. To do this, we must extend our test suites, we must
continue collaborating with GStreamer devs, we must create better ways for users to share with us failing scenarios. For all
this we've got great ideas, but what we miss is being able to work full-time on the project, which basically means we need
money, for reasons I don't think I have to detail !

I'm afraid this might sound a little boring, as we all tend to be more attracted to feature promises and shiny things,
and that's obviously what we all deserve, but I think that's not what we need right now (hope I got the quote right).

Fortunately we estimate that phase to be around 6 months long for one person full time, we did *a lot* of the groundwork
already, and we just have to expand on that, and track the corner cases cause the devil is in the details, and he knows
how to hide damn well.

After that, we will be ready to unleash GStreamer's power, and come up with great features in no time, and ride on the
work of others to get for example hardware acceleration basically for free. From that moment, when we'll have released
1.0, things will get seriously real, and our backers will be able to vote on the features they care the most about.

I've worked on the voting system and I think it's a great thing to have, I'm really impatient to see it used in real life
(and hopefully not break), I think I'll write a more technical blogpost on its implementation.

## How you can help.

I'm writing this the day before launching the campaign, and I have the website in the background, taunting me with its
"0 € raised, 0 backers" message. Fortunately I also have the spinning social widgets to cheer me up a bit, but it's not
exactly enough to get me rid of my anxiousness.

I know that what we do is right, and that requesting money for stabilization first is the correct and honest thing to do.

Obviously, I hope that you will donate to the campaign, but I also hope that after taking the time to read that rather 
lengthy blogpost in its entirety, you will be able to spread the message, and explain why what we do is important and good.

Free and Open Source video editing is something that can help make the world a better place, as it gives people all
around the world one more tool to express themselves, fight oppression, create happiness and spread love.

Hoping you'll spread the love too, thanks for reading !
