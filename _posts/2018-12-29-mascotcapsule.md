---
layout: post
title:  "Peeking further into MascotCapsule formats"
comments: true
---

About a year ago, I began to look into [adding emulation of MascotCapsule Micro3D API to freej2me](https://github.com/hex007/freej2me/issues/27). As it turns out, the API was only ever implemented in native code, the media formats used are undocumented, [the company](https://www.hicorp.co.jp/en/) pretends it never exited and so on -- nothing new in the world of emulation. Fortunately, I had [saved](https://github.com/minexew/MascotCapsule_Archaeology/tree/master/Docs_Resources_SDK) the vendor's public documentation and tools before they were removed from their website. This allowed some fuzzing of the converter, which ultimately proved insufficient, and IDA was brought in.

<figure>
![M3DConverter IDA]({{ site.url }}/images/2018-12-29-mascotcapsule/ida.png)
<figcaption>Yeah...</figcaption>
</figure>

MBAC
----

[MBAC](https://github.com/minexew/MascotCapsule_Archaeology/blob/master/Format_Descriptions/MBAC.md) is the format that describes polygonal models as implemented by the `com.mascotcapsule.micro3d.v3.Figure` class.

It uses several kinds of value packing to maximize efficiency. We don't yet have full understanding of all the fields, but it's enough to dump geometry and texture mapping. Interestingly enough, a MBAC file doesn't actually tell you which texture belongs to it. When loading the Figure in game, you have to manually bind a texture through the `Figure.setTexture` method.

In practice, while Galaxy of Fire has 46 different MBAC models, it only uses 5 textures for all of them.

![Galaxy on Fire ships.bmp]({{ site.url }}/images/2018-12-29-mascotcapsule/gof-ships.png)
<center><i>This 256x256x8-bit texture is used for all the spaceships, asteroids and a few other effects in GoF</i></center>

![alien_battleship_01.jpg]({{ site.url }}/images/2018-12-29-mascotcapsule/alien_battleship_01.jpg)
<center><i>A textured Vossk battleship.</i></center>

MTRA
----

MTRA files (`com.mascotcapsule.micro3d.v3.ActionTable`) contain animation data for Figures.

Although they don't use any bit-packing tricks like MBAC, it's still pretty difficult to guess what is going on just from looking through a hex editor; remember, there are no floats to help you find your way, everything is done in fixed point.

![mtra-dump.png]({{ site.url }}/images/2018-12-29-mascotcapsule/mtra-dump.png)
<center><i>After staring into IDA's disassembly for hours, we get something the looks like this.</i></center>

The next step will be to document this mess and then make a small script that will go over a directory of MBAC files and render a nice gallery of previews. Because why not.
