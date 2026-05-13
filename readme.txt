================================================================
  SLITHER (local) — implementation walkthrough
================================================================

The whole game lives in ONE file: index.html. It has three parts:
  1. <style>   — HUD, leaderboard, overlay menu
  2. <canvas>  — the game viewport + a separate minimap canvas
  3. <script>  — all the game logic (one IIFE, no libraries)

Open it by double-clicking index.html in Finder, or drag it into
Chrome. No build step, no server needed.

----------------------------------------------------------------
1. The big picture
----------------------------------------------------------------

A "game" here is just:
  - a WORLD of (x, y) coordinates
  - some SNAKES that live in it
  - some FOOD pellets scattered around
  - a CAMERA that follows the player
  - a LOOP that, ~60 times per second:
        update everything   →   detect collisions   →   draw

The world is a CIRCLE of radius WORLD_R (2400 units). The screen
is a window into that world. We draw by converting world (x, y)
into screen pixels:

    screenX = (x - camX) + viewW / 2
    screenY = (y - camY) + viewH / 2

So whatever the camera is centered on (camX, camY) ends up in the
middle of the screen. The camera smoothly chases the player's
head each frame:

    camX += (head.x - camX) * dt * 6   // exponential follow

----------------------------------------------------------------
2. The snake data model
----------------------------------------------------------------

Each Snake stores:
  - segs[]     — array of {x, y} points; segs[0] is the HEAD
  - dir        — current heading (radians)
  - targetDir  — where it WANTS to face; dir turns toward this
  - length     — how many segments it should have
  - speed      — px/sec
  - colorA/B   — outer + inner stripe colors

Movement is done in two steps every frame:

  (a) STEER:
      Compute the shortest angle from `dir` to `targetDir`,
      then rotate `dir` by at most TURN_RATE * dt radians.
      This is why turns feel smooth instead of snapping —
      see shortAngle() in the code.

  (b) ADVANCE THE BODY:
      Move the head forward:
          head.x += cos(dir) * speed * dt
          head.y += sin(dir) * speed * dt
      Then walk down the segment list: each segment i is pulled
      toward segment i-1 so that the distance between them is
      exactly SEG_SPACING. This "rope-follow" trick is what
      makes the body trail naturally behind the head without
      having to remember a long path history.

Growing means: length += N. The trim step at the end of update()
adds or removes tail segments to match the new length.

Player vs. bot: identical class. The only difference is that the
player's targetDir comes from the MOUSE, while bots compute their
targetDir in updateAI() (next section).

----------------------------------------------------------------
3. Input → steering (player)
----------------------------------------------------------------

Mouse handlers store the latest (mx, my) in `input`. Every frame:

    dx = input.mx - viewW/2        // mouse relative to screen center
    dy = input.my - viewH/2
    player.targetDir = atan2(dy, dx)

That's it. Because the camera follows the head and the head is
drawn at screen center, "point the mouse left" really means
"steer left." Touch events feed the same variables for phones.

BOOST: left mouse / Space sets player.boosting = true. While
boosting:
  - speed = BOOST_SPEED instead of BASE_SPEED
  - length drains at BOOST_COST_PER_SEC segments/sec
  - small food pellets are dropped behind the tail
This is what makes boosting a tactical trade-off instead of
free speed.

----------------------------------------------------------------
4. The AI bots
----------------------------------------------------------------

updateAI(s, dt) runs on a cheap timer — each bot picks a new
plan every 0.25–0.7s, not every frame. The priority ladder:

  1. If close to the world edge → steer back toward (0,0).
  2. Scan body segments of nearby snakes. If too close, build
     an "avoidance vector" pointing away from the danger,
     weighted by closeness. If the total threat is high,
     steer along that vector (and maybe boost away).
  3. Otherwise, find the nearest food within ~300 units and
     point toward it.
  4. If nothing interesting is around, wobble randomly.

It's deliberately not perfect — bots will sometimes wall-collide
or get eaten, which makes the game feel alive.

----------------------------------------------------------------
5. Collisions
----------------------------------------------------------------

Two kinds:

  FOOD (head ↔ pellet):
      For each snake, scan food pellets near the head. If a
      pellet is within (snake_radius + food_radius), eat it
      (length++, remove pellet). If just nearby, magnet-pull
      the pellet toward the head — visual polish.

  SNAKES (head ↔ other body):
      Naively checking every head against every segment is
      O(N²) and slow once snakes get long. So each frame:

        a) Build a SPATIAL HASH: divide the world into 64×64
           cells. For every body segment of every snake, insert
           it into the cell it lives in.
        b) For each head, look up only the cells around it
           (head_radius + max_radius). Compare distance to
           just those segments.

      If head_dist < head_r + seg_r → die().

      Note: we deliberately DON'T insert heads into the hash.
      That way you only die from hitting BODY, never head-on-head
      ambiguity. Whoever's head touches the other's body first
      dies — exactly the slither.io rule.

  Wall:
      In Snake.update(), if hypot(head) > WORLD_R, die().

When a snake dies, we drop food pellets along its body so that
killing a big snake leaves a feast for the killer.

----------------------------------------------------------------
6. Rendering tricks
----------------------------------------------------------------

Each snake body is drawn as TWO polylines through its segments:
  - a thick stroke in colorA (outer skin)
  - a thinner stroke in colorB on top (inner stripe)
Both use lineCap='round' and lineJoin='round', which gives the
classic rounded slither-io look without drawing each segment
as a separate circle.

Boost glow: an extra wider stroke drawn first with shadowBlur,
so the body appears to be radiating light when boosting.

Eyes: two white circles offset from the head perpendicular to
the heading, then black pupils nudged toward the steering
direction so the snake "looks where it's going."

Food: a circle with shadowBlur for a soft glow; the glow
pulses on a sine wave (each pellet has its own phase) so the
field shimmers.

Background grid: vertical and horizontal lines spaced 80 units
apart, offset by the camera position modulo 80, so the grid
slides under the player as they move.

Minimap: a second canvas (mini), drawn after the main scene.
Every snake head is plotted with world (x, y) scaled down to fit
inside the minimap's circle. The player is gold and bigger.

----------------------------------------------------------------
7. The game loop
----------------------------------------------------------------

requestAnimationFrame(tick) — Chrome calls back once per frame.
Inside tick():

    dt = (now - lastT) / 1000        // seconds since last frame
    if (dt > 0.05) dt = 0.05         // clamp big stalls (tab unfocused)

    1. read player input → set player.targetDir
    2. for each bot: updateAI(bot, dt)
    3. for each snake: update(dt)
    4. for each snake: handleFood()
    5. handleSnakeCollisions()
    6. respawn dead bots, top up food
    7. follow camera
    8. render() + renderLeaderboard()
    9. if player.alive, schedule next frame; else end game

`dt` everywhere means the game runs at the same speed regardless
of frame rate — 60fps and 30fps both move the same px/sec.

----------------------------------------------------------------
8. UI overlay
----------------------------------------------------------------

The start/death screen is plain HTML in #overlay. It's hidden
during play (`hidden` class) and shown again when the player
dies, with the final length/score injected. The name field is
remembered in localStorage so you don't retype it after dying.

The HUD (top-left) and leaderboard (top-right) are also plain
HTML — they update by setting textContent / innerHTML each
frame. Cheap because the strings are small.

----------------------------------------------------------------
9. Tuning knobs
----------------------------------------------------------------

Most numbers you'd want to fiddle with are constants near the
top of the <script>:

    WORLD_R              size of the play area
    FOOD_COUNT_TARGET    how many pellets exist at once
    BOT_COUNT            how many AI snakes
    SEG_SPACING          distance between body segments
    BASE_RADIUS          starting body thickness
    BASE_SPEED           normal speed (px/sec)
    BOOST_SPEED          boost speed
    TURN_RATE            rad/sec — lower = wider turns
    BOOST_COST_PER_SEC   length burned while boosting
    FOOD_VALUE           length per pellet

Change one, refresh the page, see the effect. That's the whole
fun of having no build step.

----------------------------------------------------------------
10. What's NOT in here
----------------------------------------------------------------

  - No multiplayer / networking — bots are local AI only.
  - No sound — drop an <audio> tag and play() on eat/die if you
    want some.
  - No persistence beyond your name. Add a high-score in
    localStorage if you like.
  - No mobile-tuned UI. Touch works but the screen is small.

Have fun.
