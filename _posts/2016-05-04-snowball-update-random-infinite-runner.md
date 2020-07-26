---
title: Snowball update, random infinite runner
date: 2016-05-04 00:00:00
layout: post
---

The game has not changed much looks wise since I last posted, but I've gotten some important functional changes done. The game now works with a randomly generated route, instead of a fixed one. What this means is the game can be truly infinite, and dynamic as I add more permutations and type of terrain.

To get into the technical changes for a bit, the original code uses a z position, which goes up in value each frame to move through the path. Now though once you pass a segment of the ground, it gets removed from the scene, and put back into the object pool. Once a full section is passed, such as a: hill, straight away, curve, or s curve, a new random section is added. This will go on indefinitely.

Originally each segment stored its index, in terms of where it is in relation to each other segment. I dropped this, in favour of using its array index instead. This allowed me to pull the first element of the array off easily, and add new ones to the end, without having to update a property. However, I did have to update the "world position" of each segment when a segment is removed. The world z position is based off of its array index, so when the array index changes, so must the world position. This is an extra O(n) operation I would really prefer to avoid, so it's something I'm going to have to continue to think on.

```javascript
cleanupSegments() {
  const baseSegmentIndex = this.findSegmentIndex(position - SlopeBuilder.SEGMENT_LENGTH);
  const segment = this.children[baseSegmentIndex - 1];
  if (segment) {
    me.pool.push(segment);
    this.children.splice(baseSegmentIndex - 1, 1);
    this.setTrackLength();
    position -= SlopeBuilder.SEGMENT_LENGTH;

    // TODO: Think if there's a way to avoid looping through all segments
    for (let i = 0; i < this.children.length; i++) {
      this.children[i].resetWorldZ(i, SlopeBuilder.SEGMENT_LENGTH);
    }
  }
}
```

As you can see here, I find the segment behind the current camera position. Fetch the segment from that. If a segment is returned, it gets pushed to the pool, removed from the collection. The setTrackLength method takes the `children.length` by the `SEGMENT_LENGTH` to determine the current length of the track, and cache it accordingly. The camera position is then negated by the `SEGMENT_LENGTH` as well. While one solution is to increment the position forever, numbers do have maximum storage values. So I went for the route of subtracting the position, and update the world position on each segment in the game world.

While the route generation is mostly random, down hill has a higher weight, so it will spawn a majority of the time. This is to give the feeling of still going down a large slope. I re-implemented the fire pit hazard, along greatly increased the rate at which collidable objects spawn on the route. Something I'm going to do soon is replace the art assets of both objects. I'm planning to simplify them on detail and number of colours, make it look more stream lined with the rest of the game, and go back to vector art for one of the items. The downside right now is the firepits or objects you collide into spawn in 1 of 3 positions in the lane. It looks a little too static, or programmed in. I hope to play with some variance on this, make it look more organic. The frequency of the objects is really high too, so I might try to put in some more non man made hazards, such as rocks and trees.

Thanks for reading. This game has my focus outside of my fulltime job now, so I should be able to post fairly regular updates.