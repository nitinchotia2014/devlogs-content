---
title: Building a tile-based dungeon generator that doesn't suck
subtitle: Three weeks in, the rooms still felt random — not generated.
date: 2024-05-02
readTime: 8
tags: [devlog, echohall, procgen]
project: echohall
author: Nitin Chotia
---

Three weeks into the prototype, something felt off. The rooms existed, but the dungeon had no _logic_.

## The problem with pure randomness

Random room placement produces noise. Players feel it immediately — there's no spine, no sense that the level was _designed_, even algorithmically.

> [!insight] Constraints are what make procedural generation feel authored. Without them, you get chaos. With too many, you get predictability. The sweet spot is a small set of hard rules and a large set of soft preferences.

## Building the spine first

The constraint solver now plants a "main path" before anything else grows.

```typescript
function buildSpine(grid: Grid, start: Vec2, depth: number): Vec2[] {
  const path: Vec2[] = [start];
  let current = start;
  for (let i = 0; i < depth; i++) {
    const next = pickBiasedNeighbour(current, grid);
    path.push(next);
    current = next;
  }
  return path;
}
```

## The result

![Dungeon generation output](https://placehold.co/800x450/0a0a0c/cdfb6f?text=dungeon+screenshot)

Here's a live demo of the constraint solver running in the browser:

:embed[https://www.youtube.com/embed/dQw4w9WgXcQ]
