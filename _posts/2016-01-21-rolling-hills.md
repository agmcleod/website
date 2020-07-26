---
title: Snowball, rolling hills
date: 2016-01-21 00:00:00
layout: post
---

I took a break from development on Snowball Effect for around a month. I participated in LibGDX Jam. You can check out my entry [here](http://itch.io/jam/libgdxjam/rate/51147) if you wish.

Before the holidays I implemented hills into the generation system. This evening I finally got the sprites clipping properly, which you can see here!

![Example of an upward slope in the track](/assets/snowball-rolling-hills.png)

This was accomplished by referencing a few tutorials on building an oldschool racer from the NES & SNES days: [codeincomplete](http://codeincomplete.com/posts/2012/6/30/javascript_racer_v4_final/). The terrain is in segments. A segment has two pieces of data, each pertaining to the top & bottom sides of the segment. Each side contains an object storing the position, width and scale. It also stores a Vector3 for its original world starting position. This is never changed. The world Vector3 stores the z coordinate, an index of where it sits relative to other segments. It also stores a y coodinate, which is its z coordinate * height.

It then has a Vector3 called camera, which is used to cache the calculated starting values stored in world, and add them against the game's camera position. This is what makes them "move" along the screen.

These vectors are used to calculate the screen position & scale needed each frame. It uses set values I have making up the game world & height.

Segments are drawn from the bottom of the screen, up to the middle where they cap out. As it goes along in a given frame, it stores the lowest Y value achieved. This Y value is stored on each segment. This Y value is then used when the sprites are drawn, from top to bottom, so they draw over top of each other properly.

Drawing a sprite takes its segements top Y value, subtracting its scaled height. It then calculates the amount to clip off by doing:

`let clipH = clipY ? Math.max(0, ypos + scaledHeight - clipY) : 0;`

So if there's a clipY coordinate, it finds the bottom of the sprite, subtracts the coordinate. Then cap the clipH so it doesn't surpass the sprite's scaled height.

The source image coordinates are relative to the sprite sheet, but the height is then dependent on the clip value.

`spriteHeight - (spriteHeight * clipH / scaledHeight)`

Height in this case is the sprite's true height, not the scaled value. It subtracts itself, from the percentange that the clip amount takes off of the scaled height. This ensures the amount drawn from the image respects the projection scaling.

The destination height is then simply:

`scaledHeight - clipH`.

Initially I was drawing from the center of the sprite, so positioned in the middle instead of top left. This lead to a lot of calculation problems with the scaling.
