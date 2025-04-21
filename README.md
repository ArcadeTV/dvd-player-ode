# dvd-player-ode
Optical Drive Emulator for a SONY DVP-S315 (or similar) standalone DVD Player

## Why build an “Optical Drive Emulator” for an obsolete system like a standalone DVD player?

I originally just wanted to watch videos in 480i/576i on a CRT, but my HTPC setup—with CRT‑Emudriver and an ATI HD5450—never quite delivered. Interlaced resolutions couldn’t be generated perfectly, and video sources vary so wildly that you can’t rely on VLC (or any other media player) to magically fix everything. Interlaced files in particular gave me headaches, since the timing between the video stream and the display never lined up 100%.

Subjectively, as someone who grew up in the ’80s and ’90s, I think DVD/MPEG‑2 was the highest‑quality consumer format ever made for tube TVs.

As a tech‑savvy retro fan with a background in electronics, I wondered how far I could get by gathering every scrap of information about my old DVD player and feeding it into an AI.

Once I sketched out the idea and tossed it into the AI, my ambition grew: I wanted to see it through and make it real. An emulator that could mount video data and ISO files, giving the old player a brand‑new lease on life. I was thrilled to find the firmware sitting on a socketed MBM29F800B‑90, which I was able to dump thanks to a matching programmer + adapter.

Before you get too excited, note that this repository is currently just an information pool. Its accuracy still needs to be verified and turned into hardware

_—but hey, it’s a start!_
