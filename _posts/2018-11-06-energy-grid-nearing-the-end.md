---
title: Energy Grid Update - Nearing The End
date: 2018-11-06 00:00:00
layout: post
---

After my last update, I have since balanced a lot of the numbers, giving the player a choice in how to collect resources. Pollution in the game exists as a way to punish the player for building unclean energy gatherers. Pollution has been changed to be a tax percentage rather than a flat amount. Pollution as a flat amount could cripple the player at the start, and not matter later. Now it scales much better. This means that as your gatherers collect resources, and those resources are sold, pollution tax is deducted from that income as a percentage.

Another major change is infinite upgrade. The upgrades that increase the resources gatherers get per tick have this feature enabled. As you research the upgrade, the cost goes up by 10% each time. This can really allow you to take the game far.

After making these changes, I noticed an issue. As you would collect resources the power bar would show a number. This number is power demand - power output.

![Power bar with demand number shown](/assets/power-bar.png)

When it’s in the negative, you’re slowly running out. When it’s in the positive the number is filling up. When you would hit the positive, the game would add another city which increases the power demand, putting it back into the negative. As the player goes through the game and more cities get added, the rate of power consumption would keep increasing. Eventually it would consume faster than I could click sell. So what I decided to do was get rid of having you click the button to sell, and make it part of the gather operation. This means that when resources are gathered, they are sold immediately. To better illustrate this, I updated the side panel UI to show gathering rates and income.

![Gathering UI showing current count of resources](/assets/gathering-ui.png)

To keep the UI updates in sync, I had to update the games systems to do the calculations of gathering, pollution, selling, and power provided in a single pass, rather than across multiple systems. Without this, even with the automated selling, you’d hit a lose condition from the power demand getting too fast.

Further more, I added a button in for you to add another city to provide power too, rather than it being automatic,. So you can control the difficulty as you progress.

## UI improvements around pollution

Another nice little improvement I worked on recently, is showing the player which tiles will be effected by pollution as you build different gatherers. You can see the effected tiles below in red.

![User clicked tile to build, ecosystem tiles show red over top that they would be polluted](/assets/pollution-selection.png)

## What’s next?

The game is getting fairly close to done. I’m working on the menu screen, and am looking to make adjustments to the settings UI. Then work on an end game screen as well.

I also plan to take another pass at the music, as I haven’t updated it since the original Ludum Dare entry.
