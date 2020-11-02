---
layout: post
title:  "Notes on MascotCapsule emulation efforts"
emoji: "🎮"
comments: true
---

_Cross-posted from a [GitHub comment](https://github.com/hex007/freej2me/issues/27#issuecomment-611225597)._

Since my efforts haven't progressed much since [last year](../../../2018/12/29/mascotcapsule.html), and nobody else seems to have picked up on them either, it might be appropriate to at least document the tools I have built along the way.

## Command Log Renderer

For a 3D rendering API, MascotCapsule v3 has a quite small [API surface](https://j2me-preservation.github.io/MascotCapsule/javadoc/). This is great -- it lets us stub all the classes fairly quickly and make games at least execute. Much of the complexity that is not immediately obvious hides in Graphics3D's command list capability, and in the data formats.

To avoid having to start with writing a rasterizer from scratch, my first steps of implementing MC3D in freej2me were to record all draw calls and save them in a _command log_ file, along with any necessary resources. The idea, greatly inspired by [Dolphin's FIFO logs](https://dolphin-emu.org/docs/guides/fifo-player-documentation-testers-and-developers/), was to create self-contained test cases that can be more easily analyzed (and rendered) using a stand-alone tool; [in this case one written in Python and using opengl](https://github.com/j2me-preservation/MascotCapsule-CommandLog-Renderer).

You can see a snippet from such command log below:

    (Figure "44e2f324"
        (data "4d420500020203019600c00000000100000000000100010000...")
    )
    (Texture "2f06e8e7"
        (data "424d7824000000000000760000002800000060000000600000...")
    )
    (drawFigure
        (figure "44e2f324"
            (texture "2f06e8e7")
            (pattern 0)
            (actionTable "null")
            (postureAction 0)
            (postureFrame 0)
        )
        (x 0)
        (y 0)
        (layout
            (affineTrans 4086 -187 96 0 190 4082 -160 0 -88 165 4091 0)
            (projection "PERSPECTIVE_FOV")
            (center 120 160)
            (perspective 100 31767 800)
        )
        (effect
            (light "null")
            (shadingType "NORMAL_SHADING")
            (texture "null")
            (toonHigh 0)
            (toonLow 0)
            (toonThreshold 0)
            (transparency 0)
        )
    )

Unfortunately, in practice it is not so straightforward to guarantee that the command list is correct in the first place. The MC3D API contains not only rendering functions, but also classes for math, vectors, matrices... If a wrong result is produced in one of those, the game might end up taking a wrong decision somewhere, and produce a nonsensical sequence of commands. In addition, some of the values in the command log, such as affine transforms, are results of calculations done by the games through the MC3D API, so any mistakes in our implementation can also propagate through there.

Ultimately, this led to the creation of a second tool.

## Fuzzer / Black-box identification tool

Although we have full documentation of the Micro3D API, it doesn't bother to go into much detail about things like rounding rules and depth resolution. However, games *do* make assumptions about these, and if you violate them, things will break -- we are speaking about things like ensuring that `sin^2(x) + cos^2(x) == 1.0` in the API's fixed-point format. Since we are fortunate enough to have the Sony Ericsson J2ME emulator which implements Micro3D (e.g. our _black-box_), we can compare behavior between the emulator and our re-implementation.

To this end, I made a J2ME program which takes a list of expressions, and evaluates them against the Micro3D API. Do not expect anything fancy, the input looks like this:

    Util3D.sin 1234
    Util3D.sin 2345
    Util3D.cos 1234
    ...

and the output is another file including the results:

    Util3D.sin 1234 9876
    Util3D.sin 2345 8765
    Util3D.cos 1234 5432
    ...

The point is: since it is a standard J2ME application, you can also run it in freej2me and compare the outputs.

The fuzzer can be found [here](https://github.com/j2me-preservation/MascotCapsule-Fuzzer).
