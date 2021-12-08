---
title: Advent of Code 2018 - My Journey of Day 15
date: 2019-01-06 00:00:00
layout: post
---

[Advent of Code](https://adventofcode.com) is a yearly advent calendar of code challenges. The idea that you get a new code challenge from the 1st through 25th of December. Each day has a part one, and a part two. Part two builds on part one, and depending on how you solved part one it can be easy or hard to figure out. Advent of Code started in 2015, and has continued every year since. I’m still in progress on finishing 2018, but have completed the other three years.

There have been days/problems in the past years that were quite challenging for me. [One](https://adventofcode.com/2016/day/11) where I first got to learn about breadth first search. [Another](https://adventofcode.com/2016/day/13) Where I got to use A* pathfinding algorithm. Overall, the more i’ve done them, the better I’ve gotten at advent of code.

For the most part I would say that rings true for this year. Though many of the problems have had little nuanced difficulties, and very specific rules that can trip you up. Day 15 is no exception to this. Though generally speaking I did enjoy the problem.

**Warning** this blog post will contain spoilers on how to solve this problem. So if this is something you wish to solve yourself, maybe come back to read this after you do.

You can read the full problem here: https://adventofcode.com/2018/day/15, but here is a rough description.

The idea is you have two teams of units, goblins vs elves. They’re in a top down, 2d map of tiles. Where units can move & attack in four directions. (Up/Down/Left/Right). Tiles are either:

1. occupied by a unit (elf or goblin)
2. open
3. unpassable (wall)

The map is always surrounded by a wall, but has walls in the middle of the map as well. The idea is you need to simulate combat and determine what team won, and with how much left over HP, and in how many turns. (goblins and elves both start with 200hp, and deal 3 damage per attack).

Order of turns being taken is done by what’s called the “reading order”. The reading order is you essentially sort by [y, x] of a given unit. So units in a lower Y coordinate go first. For any ties by the lowest y coordinate, fallback to lowest X coordinate.

Once you have a unit that takes a turn, it needs to find the closest enemy. Instead of targeting an enemy directly though, you target the 4 surrounding tiles around each enemy (if they are open).

You then find the shortest travel path to these adjacent tiles. If there’s a tie in distance, you then follow the path which is first in the reading order. So if two paths were equal distance from 0,0 to 10,10. One starts at 0,1 the other at 1,0. You would take the one that does 1,0 as it’s first in reading order.

The unit moves only one tile per turn.

Now if the move a unit took moves puts it in range of an enemy (it must be adjacent to a unit), the unit attacks. If there are two or more enemies in range, the enemy with lowest HP is chosen. If there’s a tie with the HP, you fall back to reading order of enemies.

Likewise at the start of a turn, if the unit is already within attacking range, it does not move, it just does the attacking logic described above.

The answer is then remaining hp * rounds.

Counting rounds is a bit strange, and this tripped me up for a while. A round is counted when all units on both teams have taken their turn. So if there are 3 elves and 1 goblin, if elf #2 kills the goblin, and elf 3 hasn’t gone, the round counter does NOT increment. But if elf 3 were to kill the goblin, after elf 1 & 2 took their turns, the round would count.

## The journey of development
My first approach was to use A* path finding. There are reasons why this didn’t work, but I also ignored the importance of finding the distance to the adjacent tiles, and instead targeted the unit itself. This was a mistake.

I stored two hash maps for the units, goblins and elves separately. `HashMap<(x, y), Unit>`. Unit would store the type (elf/goblin), as well as the HP it had. I would then create two lists of `Vec<(x, y)>` of Elves & goblins from these two hash maps. I would check if either one had zero length (meaning one team lost), and then count the hp sum * rounds, and exit the game.

If the game didn’t end yet, I combined the two vectors into one, and sort them by reading order. Iterating them from there, I would pull either the goblin or elf out of the respected hash maps, and have it select a target.

The select target used `A*` with the manhattan distance heuristic. If the distance (path length) was greater than zero, it would move to the spot before the end of the path (how i hoped to handle the adjacent tile thing), and then keep track of the path by its distance length. So each target would be checked, and the path with least distance would be selected. Of course if there was a tie, I would check which one won in the reading order, and then replace the min distance target with those x & y values.

After that was done, if the target was 1 away, it would choose to attack. This code I got mostly right, and it didn’t require much change throughout the work on the program. The only thing I missed was to exclude dead units from taking their turn. Do’h! My code here for selecting reading order was convoluted and hard to understand, so i’m glad I had to re-write it since!

**Anyways** when running the examples, I kept getting wrong answers. I would see that the units would sometimes move in the wrong direction, and i figured out why. Manhattan distance does not work for this problem. A path could be equal distance, but maybe a unit moves from 3,1 to 3,2 instead of to 2,1. Even though they have the same distance (because of a wall), 3,2 is closer to the target down at 5,5. But 2,1 is before 3,2 in reading order.

From the example set, here’s round 24 & 25. The only elf is at the bottom right, so the goblin at 3,1 wants to move to 5,5. It should move to 2,1 as shown in round 25, as 2,1 and 3,2 will take the same distance. The other goblins move too, but can look at the one at the top row for now.
```
After 24 rounds:
#######
#..G..#
#...G.#
#.#G#G#
#...#E#
#.....#
#######

After 25 rounds:
#######
#.G...#
#..G..#
#.#.#G#
#..G#E#
#.....#
#######
```

But mine looked like:

```
After 25 rounds:
#######
#...G.#
#..G..#
#.#.#G#
#..G#E#
#.....#
#######
```

Basically both goblins ended up moving wrongly.

So I had to abandon `A*`. I would read on the sub reddit about folks using Breadth First Search, and someone at work also doing this problem used BFS as well, so I decided to go that route.

That said, I kinda did it wrong at first.

## Breadth First Search (or Depth First Search by accident)
This solution was done using recursion, which what led me down the wrong path. As neighbours for a tile would be found, I would call the `find_paths()` on those neighbours instead of adding them to a stack. Meaning it would go far down a single path, and find the end, even though it might be the wrong way. This means I was actually doing Depth First Search.

Another problem I had was performance. In the recursive calls, I would clone my scanned list, meaning other branches in the DFS would not know if another coordinate was scanned already. I did this on purpose, as I was worried that I would not cover all cases. However, this leads it to run a crazy number of iterations, unnecessarily. What fixed this was changing `get_neighbours` to return me the neighbours of a tile in reading order. I could then keep a single HashSet of scanned coordinates while ensuring all of the options get covered.

```rust
pub fn get_neighbours(
    scanned_coords: &HashSet<Coord>,
    pos: &Coord,
    tiles: &Vec<Vec<TileType>>,
) -> Vec<(usize, usize, TileType)> {
    let mut neighbours: Vec<(usize, usize, TileType)> = Vec::with_capacity(4);

    // we push coords in reading order
    if pos.1 > 0 && !scanned_coords.contains(&(pos.0, pos.1 - 1)) {
        let tile_type = &tiles[pos.1 - 1][pos.0];
        if *tile_type == TileType::Open || *tile_type == TileType::Unit
        {
            neighbours.push((pos.0, pos.1 - 1, tile_type.clone()));
        }
    }

    if pos.0 > 0 && !scanned_coords.contains(&(pos.0 - 1, pos.1)) {
        let tile_type = &tiles[pos.1][pos.0 - 1];
        if *tile_type == TileType::Open || *tile_type == TileType::Unit
        {
            neighbours.push((pos.0 - 1, pos.1, tile_type.clone()));
        }
    }

    if pos.0 < tiles[0].len() - 1 && !scanned_coords.contains(&(pos.0 + 1, pos.1)) {
        let tile_type = &tiles[pos.1][pos.0 + 1];
        if *tile_type == TileType::Open || *tile_type == TileType::Unit
        {
            neighbours.push((pos.0 + 1, pos.1, tile_type.clone()));
        }
    }

    if pos.1 < tiles.len() - 1 && !scanned_coords.contains(&(pos.0, pos.1 + 1)) {
        let tile_type = &tiles[pos.1 + 1][pos.0];
        if *tile_type == TileType::Open || *tile_type == TileType::Unit
        {
            neighbours.push((pos.0, pos.1 + 1, tile_type.clone()));
        }
    }

    neighbours
}
```

You might also notice in this code sample, i’m allowing the coordinate to be a Unit. This was initially left over from pathing to a target, but I used that for finding local targets to attack too. So when scanning for paths to move to, if i found a neighbour that was an enemy unit, i would then have a unit not longer worry about path finding, but instead go into attack mode, collecting the potential attackers, and priortized based hp -> reading order.

```rust
fn take_turn(empty_map: &HashSet<Coord>, tiles: &mut Vec<Vec<TileType>>, unit_collection: &mut HashMap<Coord, Unit>, targets: &mut HashMap<Coord, Unit>, unit_coord: &Coord, min_distance: &mut usize, distances: &mut HashMap<usize, SelectionData>) {
    let mut attack_targets = Vec::new();
    for (target_coord, target) in targets.iter_mut() {
        if target.hp == 0 {
            continue;
        }

        let neighbours = get_neighbours(&empty_map, target_coord, &tiles);
        for neighbour in &neighbours {
            // if unit is next to a target ATTACK!!!!!!!!!! ⚔️
            if (neighbour.0, neighbour.1) == *unit_coord && neighbour.2 == TileType::Unit {
                attack_targets.push(target_coord.clone());
                break
            }

            // no need to do expensive path finding
            if attack_targets.len() > 0 {
                continue
            }

            // potential check here to
            if neighbour.2 == TileType::Open {
                select_target(
                    &tiles,
                    unit_coord,
                    &(neighbour.0, neighbour.1),
                    target,
                    target_coord,
                    min_distance,
                    distances,
                );
            }
        }
    }

    if attack_targets.len() > 0 {
        attack_targets.sort_by(|a, b| {
            let target_a = targets.get(a).unwrap();
            let target_b = targets.get(b).unwrap();

            match target_a.hp.cmp(&target_b.hp) {
                Ordering::Equal => reading_order(a, b),
                Ordering::Less => Ordering::Less,
                Ordering::Greater => Ordering::Greater,
            }
        });

        let attack_target = attack_targets.get(0).unwrap();
        attack(tiles, targets, attack_target, unit_collection.get(unit_coord).unwrap());
    } else {
        perform_move(
            tiles,
            unit_collection,
            unit_coord,
            &min_distance,
            targets,
            &distances,
        );
    }
}
```

The `perform_move` still does complex logic on tracking min distance, whether it moves into attack range, etc. But this code I was starting to feel happy with.

The `match` stuff in rust is also quite awesome. The fact that I could put `reading_order` into a utility function that works with the `Ordering` trait is just awesome.

But still, this was depth first search, so I need to move away from that.

## Breadth First Search
`find_paths` moved away from being a recursive function. Instead, I had a complex `Vec<>` type where it would store the path. I would grab each neighbour, and push it to the stack. Along with it, i would clone the current path, and push that neighbour coord to the path. So when the stack pops that coordinate, it knows what the path is including that coordinate.

If the coord that comes off the stack is the target, it would set the minimum path length using the min function, and push that path to the possible paths `Vec<>`. If the current path exceeds the length of the minimum path, it exits from that coordinate, as we’ve already beat it.

```rust
fn find_paths(
    tiles: &Vec<Vec<TileType>>,
    coord: &Coord,
    target: &Coord,
) -> Vec<Vec<Coord>> {

    let mut scanned_coords = HashSet::new();
    scanned_coords.insert(coord.clone());

    let mut paths = Vec::new();

    let mut stack = vec![FindNextData::new(scanned_coords.clone(), vec![coord.clone()])];

    let mut min_path_length = 10_000;

    while stack.len() > 0 {
        let current = stack.remove(0);
        if current.get_coord() == target {
            min_path_length = cmp::min(min_path_length, current.path.len());
            paths.push(current.path.clone());
            continue
        }

        if current.path.len() > min_path_length {
            break
        }

        let neighbours = get_neighbours(&scanned_coords, current.get_coord(), tiles);
        for neighbour in &neighbours {
            scanned_coords.insert((neighbour.0, neighbour.1));
        }

        for neighbour in &neighbours {
            if neighbour.2 == TileType::Unit {
                continue
            }
            let neighbour = (neighbour.0, neighbour.1);
            let mut path = current.path.clone();
            path.push(neighbour);
            stack.push(FindNextData::new(current.scanned_coords.clone(), path));
        }
    }

    paths
}
```

At this point a lot of the code started getting more organized. Keeping track of the data for which target spot is “winning” became easier, and so did finding the targets. Though I’m still cloning the scanned coords `HashSet`.

## Part One working
The remaining piece to get part one working was the round counting, as well as fix that cloned `HashSet` problem. So each unit by coordinates would store if they took a turn. This also got a bit tricky, as i mentioned I had two `HashMap<(x, y), Unit>` for goblins & elves. I kept those up to date, but if the sorted coordinate list was then going into a coordinate where a unit died in, and another unit moved into, a unit could up going twice. So when looping through those coordinates to take turns, I would check if `took_turn` on the Unit was false or not. Then when the unit goes through moving/attacking, the flag would get set to true.

## Part Two
After working on part one for over a week, I was able to move on to part two. The idea of part two was to find the lowest amount of damage that elves could do (they start with 3) where they win without taking losses. Instead of simulating it by doing 4 damage, 5 damage, 6 damage, etc. I decided to pick an arbitrary number of 20, with the last damage of 0 (for simplicity) and check if it was a win or loss. The “game” would also exit if a single elf dies. If the game won, it would go to the half way point between current damage & last damage. If it was a loss, it would increase by the current damage rate, which was 20. Basically I was doing a binary search to find the number faster.

And that’s it! Whew, it was quite the problem, frustrating at times, but I learned more about Rust as I went through it.

Thanks for reading, and Happy New Year!

Full source: [https://github.com/agmcleod/adventofcode-2018/blob/master/15/src/main.rs](https://github.com/agmcleod/adventofcode-2018/blob/master/15/src/main.rs)
