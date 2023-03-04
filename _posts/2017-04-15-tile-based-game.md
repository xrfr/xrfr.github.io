---
title: Tile-based Multiplayer Game
---

### Table of Contents
1. [Introduction](#intro)
1. [Components](#components)
1. [Deploying](#deploying)
1. [Map Generation](#map)

> Aprl 15th, 2017

<span id="intro"></span>
### Introduction

This is the base for a 2D multiplayer tile-based game.

<span id="components"></span>
### Components

Here's an overview of the components:

- Backend server (haskell)
  - procedurally generates a map, consisting of several layers (background, foreground, collision)
  - populates map with NPC characters, currently just clouds and rabbits
  - creates players as they connect to websocket, allows player to choose character
  - as players and NPC move around and commit actions, a seperate thread relays messages to other connected clients

- Frontend game engine (js, phaser)
  - tile based map and movement system
  - handles animations and keeping the player facing in the right direction
  - retro graphics stolen from old pokemon spritesheets

<span id="deploying"></span>
### Deploying

Configure nginx to point `/vedicmath/client` to the web frontend and `/vedicmath/ws` to the websocket port.
The websocket code is from the [nginx docs](https://www.nginx.com/blog/websocket-nginx/).

```nginx
http {
  ...
  upstream vedicmath_ws {
    server 127.0.0.1:10001;
  }

  map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
  }

  server {
    listen 80;
    server_name sachin.rudraraju.xyz www.sachin.rudraraju.xyz;
    root /home/sachin/sachin.rudraraju.xyz;

    location /vedicmath/client {
      alias /home/sachin/vedicmath/frontend/www;
      index index.html index.htm;
      try_files $uri.html $uri $uri/ =404;
    }

    location /vedicmath/ws {
      proxy_pass http://vedicmath_ws;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection $connection_upgrade;
    }
  }
}
```

<span id="map"></span>
### Map Generation

We want to have a path that cuts through the map that the player can follow. Afterwards, we'll place
signposts along the way that teach the player and provide practice problems. To complicate things,
the maps in this game are randomly generated at startup.  

I did some googling and found a thead about [procedural path generation in a grid](https://forum.unity3d.com/threads/procedural-path-generation-in-a-grid.304675/#post-1988551), which outlines an algorithm we can try. 

_The remainder of this post assumes some familiarity with haskell. I'm no expert and mostly learned
through reading [Learn You a Haskell](http://learnyouahaskell.com/)._

---

It would be beneficial to walk through existing map generation code before trying to build on it. This will also serve as a reference for myself in the future.

Firstly, the map consists of several `Layers`, each of which is a two-dimensional list (read Sequence of Sequences) of Ints.

```haskell
import qualified Data.Sequence         as S
...
data Layer = Layer {
  name :: T.Text,
  tiles :: S.Seq (S.Seq Int)
} deriving (Generic, Eq, Show)
```

And this is the tilemap spritesheet that we will be [using](https://www.spriters-resource.com/search/?q=pokemon). Each Int represents a tile to use from this spritesheet.

![tilemap]({{ '/assets/img/vedicmath-tilemap.png' | asset_url }}){:.image-default-size}

We Will be using perlin noise to generate terrain. Perlin noise is smooth, compared to a random
function, so it will create islands of tiles instead of randomly placing thems through the map. 
The function `newTileMap` takes a random seed, which is from the current timestamp (out of view). This
way we can recreate a certain map if necessary. The rest of the function so far is setting up the
perlin noise function, and a helper array `coords`, which has numbers from 0-99.

```haskell
import qualified Numeric.Noise.Perlin           as P
...
newTileMap :: Double -> [Layer]
newTileMap s = 
  let seed = lowerBits s 
      octaves = 5
      scale = 0.05
      persistance = 0.5
      perlinNoise = P.perlin seed octaves scale persistance    
      coords = [1..fromIntegral 100]
```

Next, we define a function `getTile`, which takes an `x` and `y` coordinate and returns an Int
corresponding to a tile. We set up two noise functions here, `noise1` for grass, and `noise2` for
trees. As you can see, the tree noise function is set to sample at half the frequency as the grass
one. This is because trees are made up of four tiles instead of one, either 4, 5, 14 and 15
normally, or 4, 5, 16 and 17 if there happens to be another tree directly underneath.

```haskell
      getTile (x::Int) (y::Int) = 
        let tidBelow = getTile x (y-1)
            noise1 = P.noiseValue perlinNoise (fromIntegral x, fromIntegral y, 100)
            noise2 = P.noiseValue perlinNoise (div x, div y, 200) where
                      div a = fromIntegral $ ceiling $ (fromIntegral a)/2
```

This function takes care of placing the correct four tiles for a tree and also the other tiles
depending on the value of the two noise functions at that location. You can think of us using a
threshold function on top of the perlin noise to determine whether to place trees or grass.
Trees take priority over grass and come first. 

If the tree noise function is above 0.4, we place a tree in the 2x2 square at that point.
Because the division being done above floors the x and y corrdinates, those four tile locations are
garunteed to also have a noise value above 0.4. Then, we mod the x and y coordiates with 2 to get
the coordiates for tiles _within_ each tree. The top two tiles are always 4 and 5. The decision for
the bottom two tiles takes into account what tiles are directly below them, which we determine by
just doing a call to `getTile` with an offset y coordinate in the code block above.

```haskell
            id | noise2 > 0.4 = case (x `mod` 2, y `mod` 2) of
                                  (1, 0) -> 4 
                                  (0, 0) -> 5
                                  (1, 1) -> if tidBelow == 1 then 14 else 16
                                  (0, 1) -> if tidBelow == 1 then 15 else 17
```

Next, we determine whether or not to place grass based on the grass noise function. Actually the
information above is false, since trees are technically made up of six tiles if you include the very
tops of their branches. Similarly to how we decide to include the braches above, we use tiles 6 and
7 if there is a tree below.

```haskell
               | noise1 > 0.6 = 2
               | tidBelow == 4 = 6
               | tidBelow == 5 = 7
               | otherwise = 1
        in  id
```

We have three layers in our map, `default` is the background, `overlay` is the foreground, and
`blocked` is the collision layer. First, we create a 100x100 array and fill it with tiles using
`getTile` (actually sequences).  We define a simple function `collision` which just checks if a tile
is a tree tile or not, as players should free to walk through grass. We use this function to create
the collision layer, which is composed of just 0's and 1's.

The foreground contains the tiles that appear in front of the player. A -1 means no tile. The
foreground spritesheet has tiles starting from 41. The overlay function translates background tiles to
their related tiles in the foreground spritesheet. Here, when we see 2, we translate that to 0 and
add the 41 offset.

```haskell
      collision x = x `elem` [4, 5, 14, 15, 16, 17]

      tiles = S.fromList [ S.fromList [getTile x y | x <- coords] | y <- coords ]
      blocked = (fmap . fmap) (\x -> if collision x then 1 else 0)
      overlay = (fmap . fmap) (\x -> maybe (-1) (41+) $ L.elemIndex x [2])
      
      l1 = Layer {name = "default", tiles = tiles}
      l2 = Layer {name = "blocked", tiles = blocked tiles}
      l3 = Layer {name = "overlay", tiles = overlay tiles}
  in [l1, l2, l3]
```

And this is the result. The player we'll get to later, but he illustrates the foreground versus
background (half his body is hidden behind the leaf).

![output1]({{ '/assets/img/vedicmath-map1.png' | asset_url }})

Now, we can finally get to the algorithm we found earlier. Let's start with some helper functions.
`walkable` is an extension of `collision` that also avoids grass (and other future tiles that
we haven't used yet). `bounds` takes and Int `b` and a point, and makes sure that the point is
within a square of size `b`. `@@` is just an operator we define to lookup a location in a layer.

```haskell
      walkable  x = not (collision x) && not (x `elem` [2])
      bounds b (x,y) = x >= 0 && x < b && y >= 0 && y < b
      (@@) m (x,y) = flip S.index y $ S.index m x
```

We'll define a function `findPath` that takes a layer of the map, a finishing point, and a
path--which we will initially fill with the starting point. It will either return a `Nothing` if
there is no path to be found, or `Just` a list a points that make up the path.

```haskell
      
      findPath :: (S.Seq (S.Seq Int)) -> (Int, Int) -> [(Int, Int)] -> Maybe [(Int, Int)]
      findPath ts (fx, fy) ps = 
        let (sx, sy) = last ps
```

Now to create a list of steps to extend the current path. We'll start with all the combinations of
`[-1,0,1]`. First, we filter out `(0,0)`. Then, we filter out any possible steps that violate the
bounds of the map. Next, we cross out any steps that walk onto grass. And finally, we make sure that
we aren't stepping into part of the path that we already have. 

_Note, this is written as a depth first search instead of breadth first like suggested. Also, we
desperately need to randomize `ds`, but it's quite hard to do in pure way in haskell (without the `IO`
monad). I'm thinking of using some pseudo-random function to shuffle it._

```haskell
            ds = [(x,y) | x <- [-1,0,1], y <- [-1,0,1],
                  not (x == 0 && y == 0),
                  bounds 100 (sx+x, sy+y),
                  walkable (ts @@ (sx+x, sy+y)),
                  not ((sx+x,sy+y) `elem` ps)]
```

If one of the steps we've found is the finishing point, then add it to the path and return,
otherwise, recurse on the rest of possible paths that we just generated. From the `Maybe` library,
`catMaybes` takes a list of `maybe` values, and filters out all the `Nothing`'s. `listToMaybe` takes
a list and returns the first `Just` value or a `Nothing` if the list is empty.

```haskell
        in  if (fx,fy) `elem` ds
                then Just $ (fx,fy):ps
                else listToMaybe $ catMaybes $ map (\n -> findPath ts (fx, fy) n:ps) ds

```

Now to actually add the path. We need to find two points in the map to create a path between. For
the starting point, we choose the first point along the top of the map--walking left to right--that
satisfies `walkable`, and similarily for the finishing point but walking in the opposite direction.

```haskell
      addPath :: (S.Seq (S.Seq Int)) -> [(Int, Int)] -> (S.Seq (S.Seq Int))
      addPath ts excl =
        let sp = (0, head $ filter f coords) where
                  f x = not ((x, 0) `elem` excl) && (walkable $ ts @@ (x, 0))
            fp = (head $ filter f $ reverse coords, 99) where
                  f x = not ((x, 99) `elem` excl) && (walkable $ ts @@ (x, 99))
```

As you can see, we also take in a list of points called `excl`. If the starting and finishing points
that we choose don't have a valid path between them, perhaps because they are surrounded by trees,
we have to try different starting or ending points. We first try excluding the starting point and
start the process over, then toss out the finishing point as well. When we do find a path, we
iterate over all tiles in the layer and replace the path with tile `21`.

_Note: This isn't perfect and we should actually try all finishing points with a starting point
before excluding it. Also, if we can't find any paths at all, we need to recreate the map. To be
implemented later._

```haskell
            repl x y path = if (x,y) `elem` path then 21 else (ts @@ (x,y))
        in  case findPath ts fp [sp] of
              Just p -> S.fromList [ S.fromList [repl x y p | x <- coords] | y <- coords ]
              Nothing -> addPath ts $ (if sp `elem` excl then fp else sp):excl

      tiles' = addPath tiles []
```

### Demo

<!-- ![output1]({{ '/assets/img/vedicmath-map2.png' | asset_url }}) -->

![output1]({{ '/assets/img/vedicmath-test.gif' | asset_url }})

