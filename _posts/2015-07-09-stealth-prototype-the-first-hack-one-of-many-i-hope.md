---
title: Stealth Prototype - The first hack, one of many i hope
date: 2015-07-09 00:00:00
layout: post
---

As I mentioned in the last post the focus of this prototype is going to be sneaking, and hacking to avoid enemies as supposed to killing. So I wanted to add a hackable wall that allows you to circumvent the enemies. It's a vent duct of sorts that you can sneak into. The hacking for it is super simple, just a press button with a timer.

![Press E to hack](/assets/first-hack1.png)

Note that nothing visually differs the wall to show it's an area. This is something i plan to change, but I still want to keep it a bit secret. You need to brush up against a small section, roughly two tiles high, and then you can press E like the UI says. Once you do, a bar starts filling up:

![Door now unlocked](/assets/first-hack2.png)

Due to the fact it takes time, you're at risk while standing there. As displayed before, an enemy will either chase you, or shoot you straight away. So you need to make sure to time it properly. This is one of the things I really enjoy about game development: tweaking and cleaning up the gameplay. Making it so it's challenging, but fair. It's hard to pull off in some cases, but feels great when it works. Planning is very important. I sketch out level ideas and type out design concepts, but you won't know what works out until you try.

Once the hack finishes, you can go through the wall:

![Player moving through the ventilation system](/assets/first-hack3.png)

As you can probably tell, the walls on each end still look like walls. Their needs to be a way to identify the wall tiles on both sides of the vent are now passable. Something along the lines of hide or change their image when the wall is hacked. Without that, it's hard to know what the hack actually did. For all the player knows, maybe the hack disabled a turret further down the path. That's the next thing I plan to work on.

Hacking needs to be more interesting than this. Nothing coded yet, but did a sketch on the UI I'm thinking of:

![Pen & paper sketch of a number pad connected to a door](/assets/first-hack4.jpeg)

The first sketch is a bit messy due to edits. But the left section is meant to be a metallic keypad of values 1 through 9. The second device attached by a cable scans the code over time. Filling the numbers, similar to the green numbers filling in at the start of The Matrix. Type in the number wrong and click the confirm button, the alarm goes and guards will come to your position.

The second sketch involves a metal plate that has a screen on it. The attachment takes a few seconds, and scans the unlock pattern. A similar unlock mechanism to that of android phones. You join the dots together to unlock it. 2-3 failed attempts will cause it to fail, and guards will come after you.