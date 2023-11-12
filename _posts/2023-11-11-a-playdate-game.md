---
layout: post
title: A Playdate Game
date: 2023-11-11 23:38 -0500
---

I decided to take a break from another game project I started on years ago. During COVID I did a off and on work on it. It was tricky to find motivation to build things all the time, even though one would have the free time. I hope to return to it in the future, but since late last year I got my hands on a [Playdate](https://play.date/), which is a simple mobile handheld gaming device.

The Playdate uses a non-backlit screen with 1 bit graphics, meaning a pixel is either black or it's white. The input are not too disimilar from the original Gameboy, having two buttons and a dpad, and a menu button. But also it has a fully rotational crank.

So far it's been a fun system to build for. The scripting uses Lua, but you can also use C. I've thought about using Rust bindings, but given with games it's important to be able to build quickly I figure sticking with Lua makes sense until I run into performance limitations. I started off late last year with the SDK building a breakout clone, and experimenting with the API.

## But what game am I making?

Around that time I was also replaying the original [Diablo](https://www.gog.com/game/diablo). While it's very much an old feeling game, I like the simplicity of it. Its combat is very straight forward where you click an enemy and attack it with either a melee weapon or a bow. Or you have a spell selected and click to cast said spell in a direction. The itemization is also rather straight forward. Compared to playing my fare share of Path of Exile, it was a breath of fresh air. So I decided to make my own simple top down action RPG for the playdate.

![A screnshot of the game showing the player in the center, a dropped chest armor  next to them, and being attacked by an enemy](/assets/playdate-game1.png)

I'm keeping the scope down by limiting the gameplay to just being a simple warrior. You press A to attack enemies, turn the crank to use a potion to heal, and the D Pad to move around. Defeated enemies can drop items, which you can pick up & equip.

Items can have 1-3 modifiers, selecting from a range of offensive & defensive attributes. Damage, defense, attack speed, life, resistances, etc.

![A screnshot of the inventory in the game. It shows two items in the inventory, with a short sword being selected. The short sword has 2-7 base damage, and a modifier of 43% increased damage. It also shows the stats of the currently equipped weapon, so the player can compare](/assets/playdate-game2.png)

What I hope to have is a range of levels the player can go through, where they have a relaxing & fun time grinding away.

I released a demo this summer which can be found here: [https://agmcleod.itch.io/the-depths-demo](https://agmcleod.itch.io/the-depths-demo). Since then I've been working on implementing things like multiple floors, more enemies, etc.

## Building for the playdate

The SDK is free to use, offering both Lua & C APIs as I mentioned before. Using Lua has been interesting so far, as I effectively have everything in global scope across multiple files. So i've been doing more classic namespacing to avoid collisions. Once you get an understanding of the sprite system and different APIs available, it allows one to iterate quickly.

I'm using an importer library for the level editor LDTK, which is found here: [https://devforum.play.date/t/importer-for-ldtk-level-editor/1547](https://devforum.play.date/t/importer-for-ldtk-level-editor/1547).
