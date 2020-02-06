---
name: ORCΛ UDP MIDI Translator
published: true
description: "A UDP-to-MIDI Pd translator utility for the ORCΛ live-coding environment"
icons:
  - sound
  - code
---
Because I can't just enjoy nice, free music software, I spent a few days trying to recreate and extend my live-coding setup on an iPad. Here's what worked, what failed completely, and what I learned along the way.

# What are all these things?

[ORCΛ](https://github.com/hundredrabbits/Orca) is a visual programming language for building sequencers and pattern machines. It is truly delightful, addictive, and expressive, and anyone who enjoys computers that make music should try it out. For a better idea of what ORCΛ can do, click on these:
- [Orca Workshop](https://www.youtube.com/watch?v=WIzI_PSBw6o)
- [ORCΛ Time Crystal](https://www.youtube.com/watch?v=w9do-qFNAmQ)
- [#ORCΛ](https://twitter.com/search?q=ORC%CE%9B&src=typed_query)
- [another sapling, another tree.](https://twitter.com/Johannes_Knop/status/1220113861451427842?s=20)

[Pure Data (Pd)](http://puredata.info/) is a visual programming language for multimedia. I mainly use it as multipurpose audio/MIDI/network glue when I want music apps to communicate. It's great for making audio samplers, [synthesizers](https://www.automatonism.com/), live visuals, and [Electric Singing Bowls](electric-singing-bowl).

# Why?

Some background: I've been happily using the [C port](https://github.com/hundredrabbits/Orca-c) of ORCΛ, as it runs just as well on a Raspberry Pi over SSH as it does on my 10-year-old Macbook Pro. As a true testament to [ORCΛ's creators' lean development philosophy](https://100r.co/site/philosophy.html), I was even able to build and run it inside of [iSH](https://ish.app/), an emulated Linux shell that runs on iOS. What a dream! ORCΛ running on a mobile device with access to all of the music software that I've paid actual money for.

Except [APK](https://wiki.alpinelinux.org/wiki/Alpine_Linux_package_management) doesn't have a portmidi package — an optional ORCΛ dependency that enables MIDI support — which meant that my iSH build of ORCΛ was totally cut off from the rest of my apps; totally useless. I could have attempted to build portmidi inside iSH, but even if that was successful it seemed unlikely that any MIDI coming out of iSH would ever find its way out of the sandbox and into another iOS app.

However, ORCΛ speaks OSC ([open sound control](https://en.wikipedia.org/wiki/Open_Sound_Control)) and UDP with its ```=``` and ```;``` operators, and there was nothing preventing network communication!

[![osc and udp operators](../assets/images/tools/orca-udp-midi/osc-udp-operators.png)](../assets/images/tools/orca-udp-midi/osc-udp-operators.png)

The only thing I needed was a way to listen to embedded ORCΛ's network messages and translate them into usable MIDI output — a perfect job for Pure Data, provided by danomatika's wonderful [PdParty](https://github.com/danomatika/PdParty) app.

# Issues

Initially I planned to use the ```=``` operator and send Pd messages in this form:

[![=a505Czz](../assets/images/tools/orca-udp-midi/osc-issue-1.png)](../assets/images/tools/orca-udp-midi/osc-issue-1.png)

as ORCΛ's ```=``` needs to start with a path parameter (the ```a```), and a number indicating the amount of items included in the OSC message (```5```). This was hardly ideal, as it would be in a different format than a normal ORCΛ MIDI message, and the extra characters would add a bit too much mental overhead for ORCΛ-on-iPad to still feel like a fun time.

Additionally, Pd's ```oscparse``` object wouldn't distinguish between a capital or lowercase letter sent from ORCΛ. 

[![=a505Czz](../assets/images/tools/orca-udp-midi/osc-issue-2.png)](../assets/images/tools/orca-udp-midi/osc-issue-2.png)

So only natural notes would be allowed, unless I included another parameter for sharp/flat:

[![=a505Czz](../assets/images/tools/orca-udp-midi/osc-issue-3.png)](../assets/images/tools/orca-udp-midi/osc-issue-3.png)

where I could add  a ```b``` (flat) or ```s``` (for sharp? — ```#``` already being the syntax for ORCΛ comments) after the root note. Needless to say, MIDI-over-OSC was not gonna happen.

The only solution was to use ```;``` and parse the raw binary output from ```netreceive```. Not only would this resolve the capital/lowercase problem, it would allow for UDP MIDI notes to be visually identical to ORCΛ's MIDI notes, just with a ```;``` instead of a ```:```

[![=a505Czz](../assets/images/tools/orca-udp-midi/osc-issue-4.png)](../assets/images/tools/orca-udp-midi/osc-issue-4.png)

# Mods

[![working patch](../assets/images/tools/orca-udp-midi/udp-fixing-1.png)](../assets/images/tools/orca-udp-midi/udp-fixing-1.png)

With functioning UDP MIDI set up to work exactly the same as ORCΛ's MIDI operator, I decided to address some issues I had encountered with ORCΛ itself — namely the 0-indexed MIDI channel numbers, a lack of omni-channel messages, and an inability to introduce off-beat timing in between ORCΛ's 'frames' without resorting to weird double-time BPM hacks.

_video of weird doubletime junk_

The first adjustment was simple enough: instead of treating channel ```0``` as channel 1 (which feels like a programmer quirk to me), ```0``` would be a new omni channel (thanks to Pd), and channel ```1``` would actually mean channel 1.

# Breaking the grid

To encourage ORCΛ to play notes a bit more loosely, one extra parameter would be allowed: delay. The sixth character in an ORCΛ UDP MIDI message would add small increments of time to offset the attack of the current note. 

[![working patch](../assets/images/tools/orca-udp-midi/udp-fixing-2.png)](../assets/images/tools/orca-udp-midi/udp-fixing-2.png)

Determining how this timing would interact with the current BPM  was a bit tricky, but I ultimately settled on steps of 1/18 of a quarter note. A delay of ```z``` on a note being hit once per measure would be offset exactly two beats (don't ask me why but it seemed sensible at the time) and ```0–9``` would be a nice subtle range of delay values for expressive keyboard control.

_video of delay parameter doing cool things_

I'm really pleased with the results of the extra delay parameter. Combined with unpredictable network lag and random UDP packet loss between iSH and PdParty, it yielded something close to one of GarageBand's goofy automatic drummers, but with a stripped-down, mildly chaotic style.

_sound sample from GarageBand_

# More Issues

Aside from UDP reliability, I encountered one final flaw — sending a single ```;``` message without any parameters to my patch would immediately crash Pure Data (often taking IAC Driver down with it). It isn't great when a wrong note also kills your whole rig!

Thankfully, the [mrpeach](https://puredata.info/downloads/mrpeach) ```udpreceive``` external worked as a drop-in replacement for ```netreceive```. Only now I couldn't run my translator in PdParty at all as it only supports a subset of the mrpeach externals. 😞

With a Pd-vanilla version that used ```netreceive``` prone to crashes from incomplete data (and a solid mrpeach version that wouldn't run on the iPad at all), the project's original intent met my new ambition: a portable ORCΛ machine that could drum poorly on purpose.

_video of ORCΛ running on iPad sort of well_

You can clone or download the patch [here](https://github.com/anderspollack/orca-udp-midi).

# Moving Forward

To make this thing more useful and to help justify the weekend of work that went into finding and solving nearly all of these problems, the next step for the (non-vanilla) ORCΛ UDP MIDI Translator is adding Ableton Link support via [```abl_link~```](https://github.com/libpd/abl_link), allowing ORCΛ to magically change everything else's tempo as well as its own. Update soon!
