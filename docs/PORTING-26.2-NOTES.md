# Minecraft 26.1 → 26.2 porting notes

Empirical notes, not a spec. Every entry below was hit while porting real mods
onto NeoForge 26.2.0.67 and fixed against the decompiled 26.2 sources:
AllTheOres, Mekanism, Cucumber, Mystical Agriculture, Flywheel, Ponder/catnip.

NeoForge does not publish a mod-facing migration guide, so the practical way to
answer "what changed" is to diff the release branches in this repo:

```bash
git fetch upstream '+refs/heads/26.1.x:refs/remotes/upstream/26.1.x'
git fetch upstream '+refs/heads/1.21.1:refs/remotes/upstream/1.21.1'

# NeoForge's own API surface
git diff upstream/26.1.x..upstream/26.2.x -- src/main/java/net/neoforged/neoforge

# the vanilla surface, via the patch set
git diff upstream/26.1.x..upstream/26.2.x -- patches
```

Between `26.1.x` and `26.2.x` that is ~50 files of NeoForge API and ~548 files
of vanilla patches. Note both branches are still maintained, so the diff carries
backports in both directions.

---

## Mappings: Mojang stopped publishing them

The 26.x version manifests have no `client_mappings` / `server_mappings`:

| version | `downloads` keys |
|---|---|
| 1.21.1 | client, **client_mappings**, server, **server_mappings** |
| 26.1, 26.1.2, 26.2, 26.3-snapshots | client, server |

Anything asking Loom for `officialMojangMappings()` fails with
`Failed to find official mojang mappings for 26.2`, and no Loom release fixes
it. ModDevGradle is unaffected (NeoForm has its own pipeline), and Fabric Loom
is fine because intermediary does not depend on Mojang's mappings.

## Rendering

The largest cluster of breaks.

| 26.1 | 26.2 |
|---|---|
| `MultiBufferSource`, `MultiBufferSource.BufferSource` | **deleted outright.** Emit from inside `SubmitNodeCollector.submitCustomGeometry(poseStack, renderType, (pose, buffer) -> …)`. `RenderBuffers` now only exposes section-building. |
| `BufferUploader` | deleted |
| `Minecraft.getMainRenderTarget()` | `gameRenderer.mainRenderTarget()` |
| `Minecraft.renderBuffers()` | deleted |
| `Minecraft.useShaderTransparency()` | `gameRenderer.gameRenderState().useShaderTransparency()` — **moved, not removed** |
| `levelRenderer.getItemEntityTarget()` | `levelRenderer.itemEntityTarget()` |
| `gameRenderer.getGameRenderState()` | `gameRenderer.gameRenderState()` |
| `gameRenderer.getMainCamera()` | `gameRenderer.mainCamera()` |
| `gameRenderer.getLighting()` | `gameRenderer.lighting()` |
| `LevelRenderer.allChanged()` | `Minecraft.levelExtractor.allChanged()` — **moved.** Not the same as `invalidateCompiledGeometry(...)`, which skips the tint-cache clearing. |
| `LevelRenderer.getLightCoords(level, pos)` | `net.minecraft.util.LightCoordsUtil.getLightCoords(...)` |
| `GlTexture.getFbo(dsa, depth)` | `device.frameBufferCache().getFbo(dsa, List.of(colour), depth)` |
| `GlStateManager._enableBlend()` / `_disableBlend()` | now take a draw-buffer index |
| `new GlFence()` | constructor is package-private: `RenderSystem.getDevice().createCommandEncoder().createFence()` |
| `CustomBlockOutlineRenderer.render(state, MultiBufferSource.BufferSource, PoseStack, boolean, LevelRenderState)` | `render(state, SubmitNodeCollector, PoseStack, LevelRenderState)` |
| `PictureInPictureRenderer` ctor took a `BufferSource`; `renderToTexture(T, PoseStack)` | no-arg ctor; `renderToTexture(T, PoseStack, SubmitNodeCollector)`. NeoForge registers factories as `Supplier<PictureInPictureRenderer<T>>`. |
| `ParticleFeatureRenderer` | `QuadParticleFeatureRenderer`, driven by the feature pipeline. Call `ParticlesRenderState.submit(collector, camera)` and delete any hand-rolled render passes. |

`RenderLevelStageEvent` deliberately does **not** expose a `SubmitNodeCollector`
— use `SubmitCustomGeometryEvent`, which carries one.

### Vertex formats

`VertexFormatElement` is now `record (String name, int offset, GpuFormat format)`
with no static constants, and `VertexFormat.builder` takes a step rate:

```java
VertexFormat.builder(0)
    .addAttribute("Position", GpuFormat.RGB32_FLOAT)
    .addAttribute("Color",    GpuFormat.RGBA8_UNORM)
    .addAttribute("UV0",      GpuFormat.RG32_FLOAT)
    .addAttribute("UV2",      GpuFormat.RG16_SINT)
    .addAttribute("Normal",   GpuFormat.RGBA8_SNORM)  // absorbs the old padding byte
    .build();
```

- `VertexFormat.Mode` → `com.mojang.blaze3d.PrimitiveTopology`
- `VertexFormatElement.Type` is gone; carry your own GL enum if you bind raw arrays
- `RenderPipeline.Builder.withVertexFormat(fmt, mode)` split into
  `withVertexBinding(index, fmt)` + `withPrimitiveTopology(topology)`
- `RenderType.Builder.bufferSize(...)` removed

## Data generation

- `IntrinsicHolderTagsProvider` deleted — extend `TagsProvider<T>` directly
- `TagAppender<T, T>` → `TagAppender<T>`, and it only accepts `ResourceKey<T>`.
  Use `value.builtInRegistryHolder().key()`, `registry.getResourceKey(v).orElseThrow()`,
  or `DeferredHolder#getKey()`
- `net.minecraft.advancements.Criterion` → `net.minecraft.advancements.triggers.Criterion`
- `net.minecraft.advancements.CriteriaTriggers` → `…advancements.triggers.CriteriaTriggers`
- Recipe output drops now-default fields: `category`, `bonus_rolls`, `cookingtime`

## World / entity / misc

- `BlockPos.getCenter()` removed → `Vec3.atCenterOf(pos)`
- `LivingEntity.knockback(power, xd, zd)` → `knockback(power, xd, zd, DamageSource, float damage)`
- `EntityType.create(…, EntitySpawnReason)` → takes `new EntitySpawnRequest(reason, ignoreChecks)`
- vanilla entity constants moved to `EntityTypes`
- `PlayerTeam.getColor()` returns `Optional<TeamColor>`; `teamColor.textColor().getValue()`
- `Level.isClientSide` is private now
- `CompoundTag.getList` / `contains` signatures changed (ValueIO rework)
- `Minecraft.screen` and `setScreen` moved onto `Minecraft.gui`:
  `mc.gui.screen()` / `mc.gui.setScreen(s)`
- `ClientLevel`'s constructor takes a `LevelExtractor`, not a `LevelRenderer`
- `tryExtractRenderState(be, pt, overlay)` gained `boolean isGloballyRendered`;
  NeoForge adds a further `Frustum` overload that is **not** visible from
  common/multi-loader source sets
- `submitBreakingBlockModel(pose, model, seed, progress)` → takes collected
  `List<BlockStateModelPart>`; collect via `model.collectParts(...)` first

### Collection-shaped block families

Dyed and weathering blocks moved into collections:

```java
Blocks.WHITE_CONCRETE   → Blocks.CONCRETE.pick(DyeColor.WHITE)
Blocks.COPPER_BLOCK     → Blocks.COPPER_BLOCK.weathering().unaffected()
Sheets.cutoutBlockSheet()      → Sheets.cutoutBlockItemSheet()
Sheets.translucentBlockSheet() → Sheets.translucentBlockItemSheet()
```

## Toolchain

- Minecraft 26.2 runs on **Java 25** (`javaVersion.majorVersion = 25`)
- Gradle 8.x rejects JDK 25 with `Unsupported class file major version 69` — use 9.x
- Gradle 9 fails the build on undeclared task dependencies and made some
  `ConfigurationContainer` lookups nullable
