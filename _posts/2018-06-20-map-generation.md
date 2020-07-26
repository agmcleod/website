---
title: Energy Grid Map Generation
date: 2018-06-20 00:00:00
layout: post
---

As i've been balancing the numbers for the game. I realized that the way to win was to tech up as fast as possible, and just put solar panels everywhere. This just didn't have much in terms of strategy. I want the positioning of your gatherers to matter more. The simple way to do that is reduce the number of places you can build them.

The existing map generation is really simple. It finds an open 3x3 grid on the map where it can build a set of tiles, using a random number. The tile types currently in game are: Swamp, City, River, Open. The center of the 3x3 would always be something other than Open, and the surrounding tiles would be dependent on the center. Rivers more likely around a city, swamps more likely around a river. To reduce the available space problem, I tried increasing the number of 3x3 grids, but i could only do up to 4. A number of instances the game would never start as it couldnt find enough space for 5. So I decided to rethink how I generate the map.

The first option was to use the same 3x3 generation, but break the map out into 5 sections so it could reliably do this. Much like so:

![10x10 grid layout divided into 5 sections](/assets/eg-mapgen1.png)

Then 5 3x3 grids would be able to fit, so long as they confine to the section they're in. However when I started filling out different patterns, I wasn't feeling too keen on it as a solution.

![3x3 grids in each of the 5 sections, which large chunks of space in the map](/assets/eg-mapgen2.png)

This scenario for example leaves some large chunks of space, not really creating confined areas that the user is forced to place in gatherers. The idea of constricting the space is so the user has to consider how much pollution they create. So I had another thought. What if I set the open space using defined shapes? What about the letter L. I keep the width of the L set to two spaces, using a variable length for the long part and the short foot of the letter. I then also sketched out some on how this might look, going with the assumption I would apply four "L"s to the map.

![4 L shapes, rotated differently, blocking off different parts of the grid](/assets/eg-mapgen3.png)

I realize this looks confusing, as a number of the "L"s overlap there, but the yellow space is blocked, the open space is where you can build. In most cases, you're almost always touching protected space. Meaning pollution is hard to avoid. But there's still enough squares for you to work with to get enough energy.

To pull this off in code, the main piece was tracking horizontal & vertical "L"s to ensure i dont have them overlap 100%. So if a vertical rotated L sits on more of the left side of the map, make sure the next one sits  on the right side somewhere. Otherwise it was just a matter of mapping the X & Y values to these Ls to build out the list of open squares.

Then the yellow space is filled with protected tile types. Some remaining code to sort out is generating those protected tiles more logically. As right now it's a bit of a mess!

![Screen shot of being in use in game, a number of river & city tiles](/assets/eg-mapgenresult.png)

