---
title: Projection in snowball fixed
date: 2015-12-05 00:00:00
layout: post
---

Pretty happy with this progress. I fixed the numbers and managed to get the projection working correctly. Though a major problem i had was the original code I referred to for this style assumed an anchor point of being smack in the middle of the sprite. MelonJS like a lot of 2d game engines defaults to top left for the anchor point.

In MelonJS 3, a lot of work was done to fix the anchor point and improve it's capabilities. So once I updated from 2.1.3 to 3.0, and fixed my code to follow the changes, the projection essentially fixed itself. So now, it looks like this:


![Trees added to side of the snow pathway](/assets/snowball-projection.png)

The updated game is playable here: [http://projects.agmprojects.com/snowballeffect](http://projects.agmprojects.com/snowballeffect)