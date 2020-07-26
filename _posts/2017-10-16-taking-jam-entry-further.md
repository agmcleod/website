---
title: Taking My Jam Entry Further
date: 2017-10-16 00:00:00
layout: post
---

My last post on here was about an archery game I started. That’s still a game I want to build, but at the end of July I decided to participate in [Ludum Dare](https://ldjam.com/). Knowing I could re-purpose components and systems I built for the archery game, I felt confident that I could do a game jam with Rust.

You can play the game I created here: [https://ldjam.com/events/ludum-dare/39/energy-grid](https://ldjam.com/events/ludum-dare/39/energy-grid)

Somethings went smoothly, others not so much. I had to figure out how to include things like fonts and sound in an async context when neither of those are thread safe. The short answer is to not include them in an async piece of code at all! The Rust compiler is pretty good at making sure you avoid race conditions, so really it’s best to listen to its errors, and re-think your design.  In the end I had moved the actual font texture creation & playing the music into the main loop, which runs synchronously. The game systems written on top of [Specs - Parallel ECS](https://github.com/slide-rs/specs) would simply flag data as being ready to draw new text, or play new sounds.

After completing this, and making some improvements to the draw code, I ported those back into the archery game. While my overall score in the game jam wasn’t that great, I had some really encouraging comments on my [entry](https://ldjam.com/events/ludum-dare/39/energy-grid). So I’ve since then decided to expand it.

I’ve spent the last couple of months implementing their feedback, as well as my own changes:

1. Allowing one to make any of the researched gatherers.
2. Changing the selling energy concept to give you money, which you use to create more gatherers, or advance in tech.
3. Implemented a proper scene graph.
4. Gave the game a proper restart, instead of just exiting the process.

Now that I have the game in a good spot, I figured it’s time to start working on a proper game design. While I’ve always valued planning when it comes to projects at work, it’s not something I’ve done in depth for a side project. My game Snowball Effect was done at a high level when I worked on V2, but I didn’t have it all flushed out from the get go, I did it in a very agile way. So I’ve decided to really dive deep on this game, and not just figure out what the features are at a high level, but really think it through, even the numbers.

The map instead of just one tile type  will have multiple. Ones that you can’t build on, can’t pollute, ones that you can only build certain types next to, etc. This will be used to create a new random map whenever you start a new game. Meaning saved games will also be a thing.

There will be a tech tree of how to go to other technologies, and how to research passive bonuses. The numbers in the below tree are place holder, I’ve worked out more accurate numbers that roll over better (to avoid remainders). I’m sure beyond that it will still require balancing.

![Tech tree of upgrade/technology system](/assets/EnergyGridTechTree.png)

I’m in process on figuring out a road map, and the UI design for the tech tree, along with new features. I look forward to sharing the gameplay of these changes once it’s ready.
