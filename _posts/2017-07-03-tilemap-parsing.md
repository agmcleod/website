---
title: Tilemap Parsing, Ground Detection
date: 2017-07-03 00:00:00
layout: post
---

The past few months have been a fair bit of learning on doing graphics programming, and how to use some of the existing libraries with Rust. Now that I have some foundation, I've been able to start on actual logic needed for a game.

The first steps of this game has been getting tile map rendering working, and interacting with that tile data. I'm a fairly big fan of using [Tiled](http://www.mapeditor.org/). It's a fairly versatile editor, and a number of engines have direct support for it. In this case I'm working with the data more directly, as there's no real core support for tiled in popular Rust libraries. I'm using an awesome crate called [tiled](https://crates.io/crates/tiled) to parse the data into types, so I have that covered. It gives me layers in an array, each layer containing the tile IDs, right from the XML data. Tiled exports a number of formats, and I'm using CSV format. For example:

```xml
<layer name="back" width="30" height="20">
  <data encoding="csv">
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,6,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,
    5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5,5
  </data>
</layer>
 ```

The layer is placing a number of tiles in various coordinates. It's one long list, but because the tilemap stores its number of tiles by width and height, it can keep track of what number is in what row & column fairly easily. From there, it's a matter of grabbing the 5th and 6th tile from my tileset. My tileset is an image of 92x64, with the tiles sized at 32x32, so that means 5 & 6 are on the 2nd row. Writing some simple code with division & modulous operators, one can map the UVs pretty easily:

```rust
let iw = image.width as u32;
let ih = image.height as u32;
// how many tiles is it wide & high? In this case it's 3x2
let tiles_wide = iw / (tileset.tile_width + tileset.spacing);
let tiles_high = ih / (tileset.tile_height + tileset.spacing);
// how much from 0-1 does the 32x32 take up of the source image
let tile_width_uv = tileset.tile_width as f32 / iw as f32;
let tile_height_uv = tileset.tile_height as f32 / ih as f32;
// cell is the number 5 or 6 from the example above
// subtract 1 to make it zero indexing. Then it's the same math for x & y. Just use width vs height, and modulous vs division.
let x = ((*cell as u32 - 1u32) % tiles_wide) as f32 + tileset.margin as f32 / iw as f32;
let y = ((*cell as u32 - 1u32) / tiles_wide) as f32 + tileset.margin as f32 / ih as f32;
let i = index as usize;
let tiles_wide = tiles_wide as f32;
let tiles_high = tiles_high as f32;
// now we just map the quad against those coords, i being the current index for the quad
vertex_data[i].uv[0] = x / tiles_wide;
vertex_data[i].uv[1] = y / tiles_high + tile_height_uv;
vertex_data[i + 1].uv[0] = x / tiles_wide + tile_width_uv;
vertex_data[i + 1].uv[1] = y / tiles_high + tile_height_uv;
vertex_data[i + 2].uv[0] = x / tiles_wide + tile_width_uv;
vertex_data[i + 2].uv[1] = y / tiles_high;
vertex_data[i + 3].uv[0] = x / tiles_wide;
vertex_data[i + 3].uv[1] = y / tiles_high;
```

That's the background layer, the next layer is the ground. This goes through the same code to figure out the drawing, but i've added some additional logic so we can build out a set of data for movement.

```xml
 <layer name="ground" width="30" height="20">
  <data encoding="csv">
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    2,2,2,2,2,2,2,2,2,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    1,1,1,1,1,1,1,1,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
    0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,2,2,2,0,0,0,0,0,
    2,2,2,2,2,2,2,2,2,2,0,0,0,0,0,0,0,2,2,2,2,2,1,1,1,2,2,2,2,2,
    1,1,1,1,1,1,1,1,1,1,2,2,2,2,2,2,2,1,1,1,1,1,1,1,1,1,1,1,1,1
  </data>
</layer>
```

0 value tiles are simply empty. They are full on ignored. So I just have a few tiles for a platform higher up, and then a base ground. In this game, i'm looking at making the movement point to click, so therefore use pathfinding, much like how you might make an AI go from point A to B. Initially I had a meta layer, where I placed a tile for each spot the player could move to, but that means for each new type, I'd have to add additional tiles, such as ledges, higher up areas that you should jump to, etc. So instead, why not create this data programatically?

When parsing a layer, the code is going in a top-down direction, so we iterate each row, then each cell across the columns for that row. This means that when parsing the ground tiles, we can track the open tiles above them to build a set of walkable areas.

The first step is walking through and building out this data.

```rust
let mut tile_map_render_data: Vec<PlaneRenderer<R>> = Vec::new();
// For the ground tiles, they will be stored x by y, in order to be able to group them. This is explained further down in this blog post.
// You'll note usage of LinkedHashMap. This is from a crate, not the standard library. It gives us an ordered map, so we can preserve the order we parse the ground tiles, and group them.
let mut ground_tiles: LinkedHashMap<i32, Vec<i32>> = LinkedHashMap::new();
// Storing the data y by x, as that's the order the layer is parsed in from a hierarchy perspective. Rows = y, cols = x
let mut unpassable_tiles: HashMap<usize, Vec<usize>> = HashMap::new();
for layer in map.layers.iter() {
    // I mentioned the meta layer earlier. Not in use right now, but keeping this around as it might come in handy still
    if layer.name != "meta" {
        // just building the render data
        let tilemap_plane = TileMapPlane::new(&map, &layer);
        tile_map_render_data.push(PlaneRenderer::new(factory, &tilemap_plane, tiles_texture, target));
        // collision layers being a static array that contains "ground"
        if COLLISION_LAYERS.contains(&layer.name.as_ref()) {
            // a simple function i created for iterating through a layer.
            for_each_cell(&layer, false, |x, y| {
                // building out the map of impassable tiles
                if unpassable_tiles.contains_key(&y) {
                    let mut xs = unpassable_tiles.get_mut(&y).unwrap();
                    xs.push(x);
                } else {
                    unpassable_tiles.insert(y, vec![x]);
                }
                // if not the first row, as nothing will be above it!
                if y > 0 {
                    // if above row above has collision data, hence the y - 1
                    if let Some(xs) = unpassable_tiles.get(&(y - 1)) {
                        // if it does not contain one for this column
                        if !xs.contains(&x) {
                            add_column_above_to_ground(x, y, &mut ground_tiles);
                        }
                    // or if it has zero collision data
                    } else {
                        add_column_above_to_ground(x, y, &mut ground_tiles);
                    }
                }
            });
        }
    }
}

fn add_column_above_to_ground(x: usize, y: usize, ground_tiles: &mut LinkedHashMap<i32, Vec<i32>>) {
    // it is open, so let's add it
    // we track x by y instead of y by x, as we need to go in that order for the tile grouping of grounds
    let x = x as i32;
    let y = (y - 1) as i32;
    if ground_tiles.contains_key(&x) {
        let mut ys = ground_tiles.get_mut(&x).unwrap();
        ys.push(y);
    } else {
        ground_tiles.insert(x, vec![y]);
    }
}
```

With that data together, we can break it down into groups. The reason for building out the ground_tiles as X by Y instead of Y by X, is to parse each column one at a time. This ensures that when checking a single tile, we can check the full range of the previous column to know what group it falls into. If you go Y by X, you have the current row's data as you go left to right across it, but you won't know if the row below is accessible by the current tiles.

```rust
// still storing Y by X, to keep it consistent
let mut groups: Vec<Vec<(i32, i32)>> = Vec::new();

for (col, rows) in ground_tiles.iter() {
    for row in rows {
        let mut found = false;
        let mut temp_row = 0;
        let mut temp_col = 0;
        let mut target_group_index = 0;

        // find whichgroup of (y, x)s can be used
        for (i, group) in groups.iter().enumerate() {
            let last_cell = &group[group.len() - 1];
            // if the X (col) is 0 or +1 to the right. If the Y (row) is between +1 and -1 from the last
            if (col - last_cell.1 == 0i32 || col - last_cell.1 == 1i32) && row - last_cell.0 < 2i32 && row - last_cell.0 > -2i32 {
                temp_row = *row;
                temp_col = *col;
                target_group_index = i;
                found = true;
                break
            }
        }

        // when its found, it means it can be added to a previous group, as it is within walkable range
        if found {
            groups.get_mut(target_group_index).unwrap().push((temp_row, temp_col));
        } else {
            groups.push(vec![(*row, *col)]);
        }
    }
}
```

With that data sorted out, we can then turn it into our usual Y by X map. Just faster to retrieve the data we need.

```rust
let mut hash_groups: Vec<HashMap<usize, Vec<usize>>> = Vec::new();

for group in groups {
    let mut coords: HashMap<usize, Vec<usize>> = HashMap::new();
    for (y, x) in group {
        let y = y as usize;
        let x = x as usize;
        if coords.contains_key(&y) {
            let mut xs = coords.get_mut(&y).unwrap();
            xs.push(x);
        } else {
            coords.insert(y, vec![x]);
        }
    }

    hash_groups.push(coords);
}
```
