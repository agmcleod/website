---
title: Spring Update on Energy Grid
date: 2018-05-06 00:00:00
layout: post
---

Back in the fall I started on a roadmap for Energy Grid, to figure out where take it. The main components that I set to focus on was rendering the tech tree. This led me to having to build out a number of systems, which led me to new challenges such as font/text wrapping.

Here’s where I’m at in the road map.

- [x] Replace original tech upgrade system with 3 nodes in the tech tree, one for coal, oil, and solar power. With coal already being researched. As you sell resources, you receive money. Money should be able to be used to upgrade the other two types.
- [x] Show tile researching in game menu
- [x] Create new tile types: Cities, Rivers, Swaps/lakes. Update the map generation to place these in little hubs. Coal, oil, and solar have to be built on blank tiles.
- [x] Add hydro power, only buildable on river tiles.
- [x] Add the pollution system. Coal & Oil pollute adjacent tiles with rivers, swaps, or cities in them. Hydro pollutes the river tile it is on. The more tiles polluted, the more amount of money you get taxed on per second.
- [ ] Implement the rest of the tech tree nodes. The tech tree itself is layed out, but many of the bonuses do not work yet. The remaining bonuses to be implemented can be seen in my last update.
- [ ] Sell solar panels for money. Enabled via the tech tree, but acts as another source of income, instead of being a passive bonus.
- [ ] Additional cities to power. This is to ramp up difficulty as you progress
- [ ] Create final assets for the game.
- [ ] Create menu screens
- [ ] Create tutorial/intro to the game

Here’s the game in action as of today:

![Gameplay of energy grid. Showing active resource gatherers, as well as purchasing tech.](/assets/eg-spring_update.gif)

## Technical challenges
### Drawing the Tech Tree
With the tech tree having over a dozen nodes, with connecting lines, I wanted to accomplish two things:

**The first**, manage the tech tree with a data file. Managing positions and all the information in code would have been tricky. Given the ECS library I’m using has a bit boilerplate to create a new entity with a set of components, managing this with a dataset instead would be easier to adjust and make changes.

To make this happen, I created an upgrade struct, and decorated it with `serde` so JSON can be deserialized into it.

```rust
#[derive(Serialize, Deserialize)]
pub struct Upgrade {
    pub buff: Buff,
    pub time_to_research: f32,
    #[serde(default)]
    pub current_research_progress: f32,
    pub cost: i32,
    pub status: Status,
}
```

Then in the JSON file, if a given object has an array of `children`, my rust code would iterate through that to establish the parent/child relationship. As the research system completes an in progress upgrade, it checks this tree of parent/child to update the status of the child upgrades.

The position of each node in the tree is based on simple number values in the JSON objects. The Y depth I initially tried to do it based on node depth in the JSON tree, but that led to some rows being too packed. So instead I define the Y value as a tier. 1, 2, 3, 4, etc. This is then multiplied to position them in a nicely spaced out way. X is 0->1 value, where 0 is the furthest left in the tech tree container, 1 is the furthest right. Determining the numbers here required a bit more consideration to position the nodes evenly.

**The second thing**: create an arbitrary shape drawing API. Because I’m using a wrapper of code around OpenGL, I need to draw things with triangles. For a rectangle this is pretty straight forward to put together, but for a polygon with 7 sides, knowing how the triangles should make up the shape becomes complicated. This is known as tessellation. Thankfully [Lyon](https://github.com/nical/lyon/) has a tessellation crate to give me this information. I was able to use it to produce the vertices I need to draw an arbitrary shape.

The recursive function that goes through determining the x & y coordinates of each node, this is then used to determine the 4 points of each line, going from node to node.

```rust
let last_half_x = last_position.x + SIZE_F / 2.0;
let last_half_y = last_position.y + SIZE_F / 2.0;
let half_x = x + SIZE_F / 2.0;
let half_y = y + SIZE_F / 2.0;
let points = vec![
    Vector2::new(last_half_x, last_half_y),
    Vector2::new(half_x, half_y),
    Vector2::new(half_x + 2.0, half_y),
    Vector2::new(last_half_x + 2.0, last_half_y),
];
let entity = world
    .create_entity()
    .with(Shape::new(points, [0.7, 0.7, 0.7, 1.0]))
    .with(Transform::visible_identity())
    .build();
```

The `Shape` struct is my piece of code that leverages `lyon` to do the tessellation. Building out the vertices is done via:

```rust
let mut path_builder = Path::builder();
for (i, point) in points.iter().enumerate() {
    let p = lyon_point(point.x, point.y);
    if i == 0 {
        path_builder.move_to(p);
    } else {
        path_builder.line_to(p);
    }
}

path_builder.close();

let path = path_builder.build();
let mut buffers = VertexBuffers::new();

// Create the tessellator.
let mut tessellator = FillTessellator::new();

// Compute the tessellation.
tessellator
    .tessellate_path(
        path.path_iter(),
        &FillOptions::default(),
        &mut BuffersBuilder::new(&mut buffers, VertexCtor { color }),
    )
    .unwrap();
```

Then i use the buffers variable inside my renderer, and pass it along with the Color data to my shader.

### Map Generation

Procedural generation is not something I’ve done very much of. I knew that just looping through the 10x10 grid and selecting a tile type based on a random number would not be ideal, and would likely lead to not fun scenarios. So I started thinking about how one can make small dense areas on the map of the different types.

It got me thinking of a map with small hills, where ground level would be empty, ground+1 would be swamps, ground+2 rivers, ground+3 would be cities. To keep it simple, why not pick a node with the 8 surrounding nodes free, pick a random value between 2-4, and set the tile based on that number. Then set the 8 remaining tiles 0-(n), n being the value the center tile was. So it cannot be higher than the middle tile, but it can be lower, even empty.

```rust
let mut x = 0;
let mut y = 0;
// find the center first
loop {
    x = rng.gen_range(1, 9);
    y = rng.gen_range(1, 9);

    let mut all_nodes_free = true;

    'check_nodes: for i in 0..3 {
        for j in 0..3 {
            if set_nodes.contains_key(&(x + i, y + j)) {
                all_nodes_free = false;
                break 'check_nodes;
            }
        }
    }

    if all_nodes_free {
        break;
    }
}
```

First, I just do a naive random check to find an open set of 9 nodes.

Then i choose the value of the center node. These random numbers I am likely to change to better balance the map generation.

```
let weight: u32 = rng.gen_range(0, 101);
let mut highest = 1;
let tile_type = if weight >= 90 {
    highest = 4;
    TileType::City
} else if weight >= 75 {
    highest = 3;
    TileType::River
} else {
    highest = 2;
    TileType::EcoSystem
};
```

`EcoSystem` is what I called the swamp internally.

Then it was a matter of looping through the 3x3 grid in this 9 tile space to select the new types. Just using some random numbers for the different values.

```rust
for i in 0..3 {
    for j in 0..3 {
        if x + i == center_x && y + j == center_y {
            continue;
        }
        let tile_type = if highest == 4 {
            let weight: u32 = rng.gen_range(0, 101);
            if weight >= 90 {
                TileType::City
            } else if weight >= 75 {
                TileType::River
            } else if weight >= 55 {
                TileType::EcoSystem
            } else {
                TileType::Open
            }
        } else if highest == 3 {
            let weight: u32 = rng.gen_range(0, 101);
            if weight >= 75 {
                TileType::River
            } else if weight >= 50 {
                TileType::EcoSystem
            } else {
                TileType::Open
            }
        } else if highest == 2 {
            let weight: u32 = rng.gen_range(0, 101);
            if weight >= 60 {
                TileType::EcoSystem
            } else {
                TileType::Open
            }
        } else {
            TileType::Open
        };

        set_nodes.insert((x + i, y + j), (tile_type, None));
    }
}
```

We skip center, as that is already set. Then a different set of random numbers are used depending on what the center number was. The reason it’s done in a long set of if statements is so that it is easier to adjust. I could use an array of numbers to make this cleaner in code, but I didn’t want to couple the generation with the implementation when it’s still relatively easy to follow.

## What's Next?

The next major focus is implementing the tech tree passives, and then testing them out. See how things work out balance wise, and how the game feels around the changes. After that, I will work towards making the city tiles what you are powering, and reserve tiles on screen for other cities to require power.