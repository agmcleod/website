---
title: Snowball Effect, new mechanic
date: 2016-06-13 00:00:00
layout: post
---

One of my focuses with doing the new version of this game was to add abilities and other mechanics for the player to use. The first one I have now implemented is simply firing a little snowball on tap at a fire pit. This doesn't remove the fire pit from game, but changes its state so it no longer does damage to the player's snowball.

I played with an alternate input mechanism for this mechanic. I thought it would be neat to have the player swipe from either side of the screen to fire a snowball from that side of the screen at the closest target. This presented however a couple of problems:

1. With gyro/tilt controls off, touch input was needed to move the snowball. How to best differentiate between swipes and movement intent?
2. The closest target may not be the one the user wishes to avoid.

To further the last point, I tried playing with the code a fair bit to see if I could make it intelligent, but it never felt quite right. So I decided to go with the implementation that I settled on. Which is to simply tap the target. Upon doing so, a snowball flies across the screen. Upon collision with the fire pit, it changes to a disabled state:

![Example of a fire pit target](/assets/snowball-targets1.png)

Of course to prevent the user from staying still and constantly tapping at every hazard on screen, they need to gather a resource in order to fire a snowball. This is done by colliding with the blue shapes on the hill. As they do the bar on the right side fills up:

![Example of a chunk of snow the user can hit to get more resources](/assets/snowball-targets2.png)

## Next steps

I'm considering changing the pacing of the game a little bit with this change. It's hard to make out what the snowball actually is, as it flies across the screen rather quickly. Slowing the game down could make it more identifiable. Make fewer hazards and resource targets appear, but make them larger as well. Leading to more strategy on positioning instead of reaction. Not sure if this is the way I will take the game, just something I wish to play with.