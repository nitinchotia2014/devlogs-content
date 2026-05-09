---
title: How I taught the dungeon to listen back
subtitle: Replacing a naive radius check with a real wave propagation system that bounces off walls.
date: 2024-05-28
readTime: 9
tags: [echohall, audio, procgen]
project: echohall
author: Nitin Chotia
---

Echolocation as a game mechanic sounds clean on paper. The player emits a pulse, the dungeon lights up around them, enemies are revealed. Simple.

What I shipped in week three was anything but. It was a circle. A literal `distance < radius` check rendered as a pulsing white ring. It worked, technically. It felt like garbage.

The problem wasn't visual — it was _spatial_. Sound doesn't travel through walls. My circle did. A monster two rooms away, behind solid stone, would briefly flicker into view every time the player pinged. The dungeon felt like it was cheating.

This week I threw out the circle and built something that actually obeys physics.

## Why a radius check fails

The intuition behind `distance < radius` is that sound spreads outward uniformly. In open air, roughly true. In a dungeon with corridors and doors and alcoves — completely wrong.

Sound waves travel _through connected space_. They bend around corners (poorly). They stop dead at walls. They lose energy the further they travel, but that energy loss follows the path taken, not the straight-line distance.

A tile two steps away through a door hears the ping loud and clear. A tile two steps away through stone hears nothing.

> [!insight] The radius check worked fine in my head because I was imagining an open room. The moment I added corridors the illusion collapsed — players instantly noticed that corners "lit up wrong." Trust your playtesters earlier than I did.

## Flood fill as a sound model

The replacement is a breadth-first flood fill starting from the player's position, spreading only through open tiles, with a decaying "energy" value at each step.

```typescript
function propagateSound(
  grid: TileGrid,
  origin: Vec2,
  maxEnergy: number,
): Map<string, number> {
  const visited = new Map<string, number>();
  const queue: Array<{ pos: Vec2; energy: number }> = [
    { pos: origin, energy: maxEnergy },
  ];

  while (queue.length > 0) {
    const { pos, energy } = queue.shift()!;
    const key = `${pos.x},${pos.y}`;

    if (visited.has(key)) continue;
    if (energy <= 0) continue;
    if (grid.isSolid(pos)) continue;

    visited.set(key, energy);

    for (const neighbour of grid.neighbours(pos)) {
      const cost = grid.isOpen(neighbour) ? 1 : 999;
      queue.push({ pos: neighbour, energy: energy - cost });
    }
  }

  return visited;
}
```

Each tile gets an energy value between 0 and `maxEnergy`. That value drives the brightness of the reveal effect — tiles close to the player pulse bright, distant corridors barely flicker.

## Handling diagonal movement

Pure 4-directional BFS clips corners too aggressively. Moving diagonally through an open doorway felt like the sound was being blocked by invisible geometry.

The fix is to allow diagonal neighbours but charge them extra energy — `Math.SQRT2` instead of `1`. It's not acoustically accurate but it _feels_ right, which is the only metric that matters here.

```typescript
function neighbours(
  grid: TileGrid,
  pos: Vec2,
): Array<{ pos: Vec2; cost: number }> {
  const dirs = [
    { dx: 1, dy: 0, cost: 1 },
    { dx: -1, dy: 0, cost: 1 },
    { dx: 0, dy: 1, cost: 1 },
    { dx: 0, dy: -1, cost: 1 },
    { dx: 1, dy: 1, cost: Math.SQRT2 },
    { dx: -1, dy: 1, cost: Math.SQRT2 },
    { dx: 1, dy: -1, cost: Math.SQRT2 },
    { dx: -1, dy: -1, cost: Math.SQRT2 },
  ];

  return dirs
    .map((d) => ({ pos: { x: pos.x + d.dx, y: pos.y + d.dy }, cost: d.cost }))
    .filter(({ pos }) => grid.inBounds(pos));
}
```

## The result in motion

The dungeon now responds correctly. Firing a ping down a corridor illuminates the path ahead in a clean wave. Rooms off to the side catch a faint echo only if they share a doorway. Solid walls block completely.

![Sound propagation visualised — bright tiles close, dim tiles far, walls dark](https://placehold.co/800x400/0a0a0c/cdfb6f?text=sound+propagation+visualised)

You can see it running live in the clip below — the white pulse follows the actual floor geometry instead of bleeding through stone.

:embed[https://www.youtube.com/embed/dQw4w9WgXcQ]

## What's next

The system works, but it runs every frame the pulse is active. On a 60×60 dungeon that's a 3,600-tile BFS per tick, which is fine at 60fps but will hurt once I add multiple simultaneous sound sources (the player, enemies, traps).

Next week: dirty flagging so only tiles adjacent to changed geometry get re-evaluated, and a pooled queue to avoid allocating on every ping.
