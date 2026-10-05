# SDF Text curve performance in 0.10.0

Curvature runs inside TMP's mesh generation callback. It rotates each visible glyph as a rigid quad, then existing SDF effects consume that mesh. No new renderers, meshes, materials or per-frame curve loop are added. Assigning the existing Curve Angle is a no-op. Disabled or zero-angle curves exit immediately. Radius and inverse radius are calculated once per line; the squared sine uses multiplication.

## Measurement

100 non-overlapping labels reading `Level 12345`, one font atlas/preset and Canvas, three Normal underlay layers. Unity 6000.0.83f1 / URP 17.0.4, Windows Development Player, Mono / Direct3D 11, i5-13600KF / RTX 3060, 1280 × 720 offscreen render. Each case uses 90 warmup frames and 240 measured frames.

CPU is the first Canvas callback cycle plus property/SetText update time. It is not whole-frame or GPU time. Offscreen rendering invokes three Canvas cycles per frame. Content changes update all 100 labels; angle changes update all 100 Curve Angle properties between 20° and 59°.

| Case | Mean CPU | P95 CPU | Draw calls | Whole-frame managed allocations |
| --- | ---: | ---: | ---: | ---: |
| Straight, static | 0.0518 ms | 0.0808 ms | 2 | 332.1 bytes/frame |
| Curved 30°, static | 0.0433 ms | 0.0489 ms | 2 | 331.5 bytes/frame |
| Straight, content changes | 4.4869 ms | 4.8956 ms | 2 | 2731.5 bytes/frame |
| Curved 30°, content changes | 4.7458 ms | 5.1937 ms | 2 | 2731.5 bytes/frame |
| Curved, angle changes | 5.1588 ms | 5.7519 ms | 2 | 2731.5 bytes/frame |

Curved content changes added 0.2589 ms (about 5.8%) for 100 labels in this capture. Static results are equivalent within measurement noise; the lower curved number is not evidence of a speed improvement. Angle animation regenerates TMP and effect geometry every frame, so animate only the labels that need it.

Every non-empty case retained 200 active renderers, two shared materials, 200 meshes, 1,766,400 renderer-reported mesh bytes and 8,800 triangles. Existing effect merging and batching remain intact. Curvature changes positions, not topology.

A separate warmed allocation probe invoked the curve callback 10,000 times on a ten-glyph label, restoring preallocated source arrays each time: **zero managed bytes allocated**, 35.6057 ms total including the array restores and delegate calls. This is a focused curve probe; whole-frame counters above also include TMP, effect rebuilding and the benchmark harness. Their dynamic allocations are unchanged between straight and curved content updates.

## Validation and limits

An automated idle check confirms that 30 unchanged Canvas cycles and reassignment of the existing angle trigger no mesh-generation callback. Changing the angle triggers one rebuild and retains the face source mesh, effect renderer and merged renderer/vertex counts.

No mobile device, GPU timing, thermals or battery measurements were performed. Text spacing and preferred size use TMP's straight layout; leave room for the curved result. Decorations retain the straight layout. These measurements cover a single font atlas and compatible labels; masks, fallback atlases, ordering and material presets can split batches.

Raw data: [CSV](Performance-0.10.0.csv), [environment](Performance-0.10.0-environment.txt), [allocation probe](Curve-0.10.0-allocations.txt). See [validation](../VALIDATION.md).
