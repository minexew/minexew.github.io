---
layout: post
title:  "Peeking further into MascotCapsule formats"
comments: true
---

About a year ago, I began to look into [adding emulation of MascotCapsule Micro3D API to freej2me](https://github.com/hex007/freej2me/issues/27). As it turns out, the API was only ever implemented in native code, the media formats used are undocumented, [the company](https://www.hicorp.co.jp/en/) pretends it never exited and so on -- nothing new in the world of emulation. Fortunately, I had [saved](https://github.com/minexew/MascotCapsule_Archaeology/tree/master/Docs_Resources_SDK) the vendor's public documentation and tools before they were removed from their website. This allowed some fuzzing of the converter, which ultimately proved insufficient, and IDA was brought in.

{% include figure.html alt="M3DConverter IDA" src="{{ site.url }}/images/2018-12-29-mascotcapsule/ida.png" caption="Yeah..." %}

MBAC
----

[MBAC](https://github.com/minexew/MascotCapsule_Archaeology/blob/master/Format_Descriptions/MBAC.md) is the format that describes polygonal models as implemented by the `com.mascotcapsule.micro3d.v3.Figure` class.

It uses several kinds of value packing to maximize efficiency. We don't yet have full understanding of all the fields, but it's enough to dump geometry and texture mapping. Interestingly enough, a MBAC file doesn't actually tell you which texture belongs to it. When loading the Figure in game, you have to manually bind a texture through the `Figure.setTexture` method.

In practice, while Galaxy of Fire has 46 different MBAC models, it only uses 5 textures for all of them.

{% include figure.html alt="Galaxy on Fire ships.bmp" url="{{ site.url }}/images/2018-12-29-mascotcapsule/gof-ships.png" caption="This 256x256x8-bit texture is used for all the spaceships, asteroids and a few other effects in GoF" %}

{% include figure.html alt="alien_battleship_01.jpg" src="{{ site.url }}/images/2018-12-29-mascotcapsule/alien_battleship_01.jpg" caption="A textured Vossk battleship." %}

MTRA
----

MTRA files (`com.mascotcapsule.micro3d.v3.ActionTable`) contain animation data for Figures.

A MTRA will contain data for each bone (aka "segment") in the corresponding MBAC. In the simplest case, this will be just one byte that says "the transformation matrix for this bone at any given time is identity". In more complex cases, you have the usual bunch of keyframes for translation, rotation and scaling. If you know how glTF does animation, the principle is the same. The data model in binary MTRA corresponds directly to the source TRA format, for which we fortunately [have documentation](https://github.com/minexew/MascotCapsule_Archaeology/blob/master/Docs_Resources_SDK/data_format_tra4_2_1.zip).

Although MTRA files don't use bit-level packing like MBAC, it's still pretty difficult to guess what is going on just from looking with an hex editor; remember, there are no floats to help you find your way, everything is done in fixed point.

{% include figure.html alt="hex editor" src="{{ site.url }}/images/2018-12-29-mascotcapsule/hex.png" caption="Want to make a guess? Hint: the 20-byte trailer is something we've seen before" %}

{% include figure.html alt="mtra-dump.png" src="{{ site.url }}/images/2018-12-29-mascotcapsule/mtra-dump.png" caption="After staring into IDA's disassembly for hours, we get something the looks like this." %}

The next step will be to document this mess and then make a small script that will go over a directory of MBAC files and render a nice gallery of previews. Because why not.
