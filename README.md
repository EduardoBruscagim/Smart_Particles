# Smart Particles — Optimized Edition (Minecraft 26.2 Fabric)

![Java 25](https://img.shields.io/badge/Java-25-ED8B00?logo=openjdk&logoColor=white)
![Minecraft](https://img.shields.io/badge/Minecraft-26.2%20Snapshot-62B47A?logo=minecraft&logoColor=white)
![Fabric Loom](https://img.shields.io/badge/Fabric%20Loom-1.15-DBD0B4)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

> **Upstream Credit:** This repository is an optimized fork and Minecraft 26.2 port of [Smart Particles](https://github.com/chedidandrew/Smart_Particles), originally created by **chedidandrew**. All credit for the original concept, configuration screen, and base mod architecture belongs to the original author.

---

## Overview

**Smart Particles (Optimized Edition)** ports the mod to **Minecraft 26.2 (Java 25 / Fabric Loom 1.15 non-obfuscated Mojang mappings)** and re-engineers the core particle culling pipeline inside [`ParticleManagerMixin.java`](Fabric/Smart_Particles_26.2_Fabric/src/main/java/sp/mixin/ParticleManagerMixin.java) to minimize CPU tick overhead, eliminate redundant iterations, and reduce Garbage Collector (GC) pressure.

---

## Algorithmic & Memory Optimizations Implemented in This Fork

| Optimization | Upstream Implementation | Optimized Fork Implementation | Technical Impact |
|---|---|---|---|
| **Single-Pass Eviction** | Full second pass over all particles to remove entries exceeding the heap threshold | Marks evicted heap root dead (`heapParticles[0].remove()`) in-place; Vanilla's next `tick()` cleans it from the `Queue` and updates `trackedParticleCounts` | Eliminates an entire $O(n)$ iteration pass every tick |
| **Linear Fast-Path ($N \le K$)** | Runs $O(n \log k)$ max-heap scoring on every tick regardless of particle count | Pre-counts total particles; when `totalParticles <= limit`, executes a pure $O(n)$ frustum-only pass (`cullFrustumOnly`) without touching heap arrays | Reduces normal-gameplay culling from $O(n \log k)$ to $O(n)$ |
| **FOV Trigonometry Caching** | Recalculates `Math.cos(Math.toRadians(fov))` on every client tick | Caches `cachedFov` and `cachedFrustumThresholdSq`, recomputing trigonometry only when the player's FOV slider changes | Eliminates transcendental math calls in the per-tick setup |
| **Inner-Loop Hoisting** | Repeatedly invokes virtual getters (`camera.position()`, `camera.forwardVector()`) inside the particle loop | Hoists camera coordinates (`camX`, `camY`, `camZ`) and forward unit vectors (`camDirX`, `camDirY`, `camDirZ`) into `final` primitive locals | Tighter JIT inner loop with zero object allocations per particle |
| **Dynamic Heap Shrinking** | `heapParticles` and `heapScores` arrays only grow (`< limit`) and never release memory when the user lowers the limit | Shrinks internal heap buffers when `heapParticles.length > limit * 2` | Prevents permanent heap bloat after temporary stress spikes |
| **GC Reference Clearing** | Retains `Particle` references inside `heapParticles[0..heapSize]` across ticks | Nulls out `heapParticles[0..heapSize - 1]` immediately after the tick pass | Allows dead `Particle` instances to be collected in Young Gen without promotion |

---

## Architecture & Minecraft 26.2 Port Details

1. **Non-Obfuscated Mojang Mappings (`net.fabricmc.fabric-loom` 1.15):**
   - Migrated build script (`build.gradle`) to Minecraft `26.2-snapshot-1` and Java `25` (`JavaVersion.VERSION_25`).
   - Updated Mixin targets and `@Accessor` interfaces ([`SPAccessor.java`](Fabric/Smart_Particles_26.2_Fabric/src/main/java/sp/mixin/SPAccessor.java) and [`ParticleGroupAccessor.java`](Fabric/Smart_Particles_26.2_Fabric/src/main/java/sp/mixin/ParticleGroupAccessor.java)) to match 26.2's `ParticleEngine` and `Particle` field signatures (`trackedParticleCounts`, `particles`, `x`, `y`, `z`).
2. **Bounded Max-Heap Culling ($O(n \log k)$ Worst-Case):**
   - When active particles exceed `maxParticles` ($k$), a bounded binary max-heap (`heapSiftUp` / `heapSiftDown`) maintains the $k$ lowest-score (closest/in-view) particles using primitive `double[]` scores (`heapScores`) paired with `Particle[]` references (`heapParticles`), avoiding `PriorityQueue` boxing overhead.

---

## Compatibility

- **Compatible with:** Sodium, Iris, Lithium, FerriteCore, ModernFix, ImmediatelyFast, and ModMenu.
- **Incompatible with:** AsyncParticles (due to concurrent access on `ParticleEngine`'s particle queues).

---

## Building from Source

Requires **JDK 25**.

```bash
cd Fabric/Smart_Particles_26.2_Fabric
./gradlew build
```

Compiled JAR artifacts are generated in `Fabric/Smart_Particles_26.2_Fabric/build/libs/`.

---

## License and Credits

- **Original Author:** [chedidandrew](https://github.com/chedidandrew/Smart_Particles)
- **26.2 Port & Algorithmic Optimizations:** [Eduardo Guilherme Bruscagim](https://github.com/EduardoBruscagim)
- **License:** [MIT License](LICENSE)
