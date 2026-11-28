# Findings: wave64 compute programs on 32-lane hosts, DexNom `4ad8692` and Marvisak/VIctory#2633

Not for merge. This compares how DexNom's translator and Marvisak/VIctory#2633 translate the title's wave64 compute programs for hosts whose waves are at most 32 lanes (NVIDIA, Intel: the Experimental NVIDIA mode). Radeon runs these programs as native wave64 and is not affected. Translation only: no game was run for these numbers.

## Builds and inputs

- **DexNom:** `4ad8692` (`main`, 2026-11-27), release build, `VICTORY_HOST_WAVE=32 victory translate --list`, with the local hook below.
- **PR 2633:** Marvisak/VIctory `main` (`63bcca4`) + Marvisak/VIctory#2633 (`00a0234`) + Marvisak/VIctory#2319 and Marvisak/VIctory#2178 (the two NVIDIA patches I run with), `victory translate --list --wave32`.
- **Inputs, the same for both:** `shaders.rpf`, `game_scarlett_master.exe`, `XboxBvhBuilder.dll` and `amd_fidelityfx_upscaler_xbox.dll` from one dump of the title.

The hook does three things, and nothing else changes in the build:
- it writes on each list line which step of `emit::emit` produced a wave64 compute program;
- it marks programs that still declare `WaveSize(64)`, whose pipelines a 32-lane driver refuses (debug layer id 622);
- it lists emitted programs without `--validate`, as the upstream list does.

## Numbers

Compute programs both lists name: 2,734. DexNom's list leaves out 29 programs of `amd_fidelityfx_upscaler_xbox.dll` (section E).

| | DexNom `4ad8692` | main + PR 2633 |
|---|---|---|
| Emitted | 2,731 | 2,703 |
| Of these, still `WaveSize(64)` (no pipeline on a 32-lane host) | 31 | 0 |
| **With a pipeline on a 32-lane host** | **2,700** | **2,703** |
| Translated by an APPROXIMATE local-waves form | 41 | 0 |

DexNom's 1,158 wave64 compute programs, by the form that produced them:

| Form | Programs |
|---|---|
| plain lowering | 926 |
| without skips / without early-outs / without skips, scalars followed | 83 / 28 / 22 |
| local waves for the allocation idiom (exact) | 27 |
| local waves, APPROXIMATE: each 32-lane half gets its own wave-wide result | 41 |
| still `WaveSize(64)` | 31 |

Exact-first with DexNom's forms as the fallback would give a pipeline to 2,729 of the 2,734 programs, with 1 left approximate. That count comes from the two lists, not from a build. The 5 without a pipeline are the water light programs (section D).

## Program lists

### A. Approximate in DexNom, exact with PR 2633 (40 of 41)

- **shadowmask** (7)
  - `CS_GenerateShadowMasks`
  - `CS_GenerateShadowMasks_HairOnly`
  - `CS_GenerateShadowMasks_Raytraced`
  - `CS_RTConstantCostShadowsTileClassificationNormalCones`
  - `CS_TraceShadowRays_Full`
  - `CS_TraceShadowRays_Fullres`
  - `CS_TraceShadowRays_ScreenSpaceOnlyFullRes`
- **cascadeshadows_rendering** (4)
  - `CS_ShadowRenderTiledCascadesAlphaCBPPPC`
  - `CS_ShadowRenderTiledCascadesCBPPPC`
  - `CS_ShadowRenderTiledCascadesPPPC`
  - `CS_ShadowRenderTiledMiscAlphaPPPC`
- **indirect_diffuse_lighting** (4)
  - `CS_DeferredFallback_BVH`
  - `CS_DeferredFallback_Fallback`
  - `CS_DeferredFallback_Voxels`
  - `CS_ResolveVBuffer_Voxels`
- **lens_flares** (4)
  - `CS_ScanMip0`
  - `CS_ScanMip1`
  - `CS_ScanMip2`
  - `CS_ScanMip3`
- **scattering_transmittance_system** (4)
  - `CS_Gather`
  - `CS_GatherDensityAreaForReadBack`
  - `CS_GatherDiffusion`
  - `CS_Gather_Reference`
- **sr_denoising** (4)
  - `CS_TemporalResolve1_Opaque_FrameStatistics`
  - `CS_TemporalResolve2_Opaque_FrameStatistics`
  - `CS_TemporalResolveOpaque2`
  - `CS_TemporalResolveOpaque_FrameStatistics`
- **indirect_light_probes** (3)
  - `CS_IrradianceCache_IntegrateIrradianceSH`
  - `CS_IrradianceCache_Shade`
  - `CS_IrradianceCache_ShadeSecondary`
- **ptx_prerender** (2)
  - `ptfx_CS_lightAtlas_Sprite`
  - `ptfx_CS_lightAtlas_Trail`
- **sr_tracing** (2)
  - `CS_DeferredFallbacksAlpha_SkyProbe`
  - `CS_ResolveVolumetrics`
- **tiled_lighting** (2)
  - `CS_ClusterLightsWorldSpace`
  - `CS_ClusterLightsWorldSpaceIntensityBased`
- **firevbuffer** (1)
  - `CS_EntityFireResolve_Scalarized`
- **shader_binning_utils** (1)
  - `CS_GenerateShaderBinOffsets`
- **vfx_proj_decal_vbuffer** (1)
  - `CS_ProjDecalResolve_Vectorized`
- **water_interaction_foam** (1)
  - `CS_DisturbersContribution`

The one left, refused by PR 2633 for a reason main has too: `vfx_proj_decal_vbuffer:CS_ProjDecalResolve_Scalarized` (a descriptor's address or format read as data).

### B. No pipeline on a 32-lane host in DexNom, translated by PR 2633 (29)

DexNom's outcome after each name.

- **vfx_proj_decal_trail** (9)
  - `CS_BuildProjDecalTrailProjectionBoxInstances`: still `WaveSize(64)`
  - `CS_BuildProjDecalTrailVertexBufferFromNodes`: still `WaveSize(64)`
  - `CS_DefragHeap_ComputeRelocations`: still `WaveSize(64)`
  - `CS_PackProjDecalTrailVertexBuffer`: still `WaveSize(64)`
  - `CS_SpawnLiquidTrail_SampleDecalRaycastTriangleTest`: still `WaveSize(64)`
  - `CS_SpawnLiquidTrail_SampleDecalRaycastTriangleTest_Nodes`: still `WaveSize(64)`
  - `CS_UpdateProjDecalTrailAttachment`: still `WaveSize(64)`
  - `CS_UpdateProjDecalTrailAttachment_Nodes`: still `WaveSize(64)`
  - `CS_UpdateProjDecalTrailMeshIndices`: still `WaveSize(64)`
- **GpuParticleEmit** (7)
  - `CS_EmitCounts_Dynamic_Edge`: still `WaveSize(64)`
  - `CS_Emit_Dynamic_Model`: still `WaveSize(64)`
  - `CS_Emit_Dynamic_Model_AttachMesh`: still `WaveSize(64)`
  - `CS_Emit_Dynamic_Model_AttachMesh_Skinned`: still `WaveSize(64)`
  - `CS_Emit_Dynamic_Model_Edge`: still `WaveSize(64)`
  - `CS_Emit_Dynamic_Model_EdgeHashGrid`: still `WaveSize(64)`
  - `CS_Emit_Dynamic_Model_Skinned`: still `WaveSize(64)`
- **XboxBvhBuilder.dll** (5)
  - `XboxBvhBuilder.dll at 0x25c40`: still `WaveSize(64)`
  - `XboxBvhBuilder.dll at 0x28770`: still `WaveSize(64)`
  - `XboxBvhBuilder.dll at 0x99160`: a first-lane read where not every lane runs
  - `XboxBvhBuilder.dll at 0x9a830`: a first-lane read where not every lane runs
  - `XboxBvhBuilder.dll at 0x9be80`: a first-lane read where not every lane runs
- **vfx_triangleClip** (3)
  - `CS_CollectTriangles`: still `WaveSize(64)`
  - `CS_CollectTriangles_Meshlets`: still `WaveSize(64)`
  - `CS_DefragHeap_ComputeRelocations`: still `WaveSize(64)`
- **shadowmask** (2)
  - `CS_GenerateShadowMasksHairOnly_SampleHairScreenSpace`: still `WaveSize(64)`
  - `CS_GenerateShadowMasks_SampleHairScreenSpace`: still `WaveSize(64)`
- **GpuParticleUpdate** (1)
  - `CS_Visibility`: still `WaveSize(64)`
- **alphaResolve** (1)
  - `CS_AlphTransmissionPrefilterMip`: still `WaveSize(64)`
- **sr_tracing** (1)
  - `CS_Tiling_Generate_Opaque`: still `WaveSize(64)`

### C. Translated by DexNom, refused by PR 2633 (26)

PR 2633's reason after each name; "(not wave64)" marks programs that are not wave64 programs.

- **rtdirectionalshadows** (13)
  - `CS_RTShadowTrace32HQ`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32HQFarOnly`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32HQ_Tiled`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32HQ_TiledFarOnly`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQFarOnly`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_OneOver2RPP`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_OneOver2RPPFarOnly`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_OneOver2RPP_Tiled`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_OneOver8RPP`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_OneOver8RPP_Tiled`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_Tiled`: a lane-mask join of an untracked mask (not wave64)
  - `CS_RTShadowTrace32TemporalHQ_TiledFarOnly`: a lane-mask join of an untracked mask (not wave64)
- **bvh_skinning** (4)
  - `CSComputeSkinningPedBlendShapes`: a lane mask read as data (not wave64)
  - `CSComputeSkinningPedBlendShapesSecMotion`: a lane mask read as data (not wave64)
  - `CSComputeSkinningPedBlendShapesSecMotionVelocity`: a lane mask read as data (not wave64)
  - `CSComputeSkinningPedBlendShapesVelocity`: a lane mask read as data (not wave64)
- **ambient_map** (3)
  - `CS_AmbientMapSky32`: a group whose rows a 32-lane host wave may tile (wave64 on a 32-lane host)
  - `CS_AmbientMapSkyClassified32`: a group whose rows a 32-lane host wave may tile (wave64 on a 32-lane host)
  - `CS_AmbientMapSkyLocal32`: a group whose rows a 32-lane host wave may tile (wave64 on a 32-lane host)
- **GpuParticleEmit** (2)
  - `CS_EmitCounts_WaterSim`: a group whose rows a 32-lane host wave may tile (wave64 on a 32-lane host)
  - `CS_Emit_WaterSim`: a group whose rows a 32-lane host wave may tile (wave64 on a 32-lane host)
- **ShallowWaterSim** (2)
  - `CS_AssembleComputeList`: a lane mask read as data, written where not every thread of the group meets (wave64 on a 32-lane host)
  - `CS_MinTimestepReduce`: a descriptor read only as data
- **terrainshadows** (1)
  - `CS_ComputeTerrainVisibility`: a group whose rows a 32-lane host wave may tile (wave64 on a 32-lane host)
- **vfx_proj_decal_vbuffer** (1)
  - `CS_ProjDecalResolve_Scalarized`: a descriptor's address or format read as data

### D. No pipeline on a 32-lane host in either (5)

Still `WaveSize(64)` in DexNom; PR 2633's reason after each name.

- **water_shading** (5)
  - `CS_TransmissionVolumetrics`: v_cvt_f32_f64
  - `CS_TransmissionVolumetricsSubmerged`: v_cvt_f32_f64
  - `CS_TransmissionVolumetrics_Tiled`: v_cvt_f32_f64
  - `CS_TransmissionVolumetrics_Tiled_LocalLights`: v_cvt_f32_f64
  - `CS_TransmissionVolumetrics_VolumePortalsOnly`: v_cvt_f32_f64

### E. Not in DexNom's list (29)

All in `amd_fidelityfx_upscaler_xbox.dll`. PR 2633 translates 24 of them; it refuses 5 for reasons main has too.

## Reproducing

```powershell
# DexNom, with the hook applied
$env:VICTORY_HOST_WAVE = '32'
victory translate --list shaders.rpf game_scarlett_master.exe XboxBvhBuilder.dll amd_fidelityfx_upscaler_xbox.dll > list-dexnom.txt
# Marvisak/VIctory main + Marvisak/VIctory#2633
victory translate --list --wave32 shaders.rpf game_scarlett_master.exe XboxBvhBuilder.dll amd_fidelityfx_upscaler_xbox.dll > list-2633.txt
```

<details><summary>The local hook on DexNom <code>4ad8692</code></summary>

```diff
diff --git a/apps/victory/src/translate.rs b/apps/victory/src/translate.rs
index 133787a..d3f2d32 100644
--- a/apps/victory/src/translate.rs
+++ b/apps/victory/src/translate.rs
@@ -536,6 +536,11 @@ fn translate_programs(container: &Container, programs: Vec<NativeProgram>, examp
                     *counter(t, p.stage, 0) += 1;
                     match emit::emit(container, &p, &program, &s, &i, &r) {
                         Ok(shader) => {
+                            // LOCAL MEASUREMENT HOOK: the form tag on the list line.
+                            let form = emit::take_form();
+                            let label = || {
+                                if form.is_empty() { format!("{} {:?}", example(), p.stage) } else { format!("{} {:?} [{form}]", example(), p.stage) }
+                            };
                             census_line(&label(), &p, &program, &shader);
                             *counter(t, p.stage, 1) += 1;
                             check(shader, t, checks, label);
@@ -645,6 +650,10 @@ fn check(
         }
     }
     let Some(v) = checks.validator.as_mut() else {
+        // LOCAL MEASUREMENT HOOK: list emitted programs too, as upstream's list does.
+        if t.list && checks.host.is_none() {
+            t.lines.push(format!("{}  emitted", label()));
+        }
         return;
     };
     match v.validate(&mut shader.dxil) {
diff --git a/crates/victory-shader/src/emit/mod.rs b/crates/victory-shader/src/emit/mod.rs
index 2f413de..463d59c 100644
--- a/crates/victory-shader/src/emit/mod.rs
+++ b/crates/victory-shader/src/emit/mod.rs
@@ -408,6 +408,21 @@ fn tiled_ambient(c: &Container, native: &NativeProgram) -> bool {
         })
 }
 
+// LOCAL MEASUREMENT HOOK: the form that produced the last program.
+thread_local! {
+    static FORM: std::cell::RefCell<String> = const { std::cell::RefCell::new(String::new()) };
+}
+fn set_form(s: &str) {
+    FORM.with(|f| *f.borrow_mut() = s.to_owned());
+}
+/// The form tag of the program just emitted on this thread (measurement only).
+pub fn take_form() -> String {
+    FORM.with(|f| std::mem::take(&mut *f.borrow_mut()))
+}
+fn host32_w64(native: &NativeProgram) -> bool {
+    native.stage == HwStage::Cs && native.wave64 && lower::host_max_wave() == 32
+}
+
 /// Emits a program whose earlier stages already ran.
 pub fn emit(
     c: &Container,
@@ -419,7 +434,11 @@ pub fn emit(
 ) -> Result<HostShader, ShaderError> {
     // (what an earlier program's lowering on this thread left behind)
     lower::take_needs();
+    set_form("");
     let original = emit_program(c, native, p, st, io, res);
+    if host32_w64(native) {
+        set_form("w64 plain");
+    }
     // A console wave64 compute program on a host of 32-lane waves that
     // still needs 64 lanes, or whose cross-lane code sits where not every
     // lane runs: once more with its early-outs left out
@@ -436,6 +455,9 @@ pub fn emit(
     if stuck && native.stage == HwStage::Cs && native.wave64 && lower::host_max_wave() == 32 {
         let why = std::env::var_os("VICTORY_WAVE_WHY").is_some();
         let report = |what: &str, r: &Result<(HostShader, Option<u32>), ShaderError>| {
+            if matches!(r, Ok((_, wave)) if *wave != Some(64)) {
+                set_form(&format!("w64 {what}"));
+            }
             if why {
                 match r {
                     Ok((_, wave)) => eprintln!("WAVE64-RETRY {what}: wave {wave:?}"),
@@ -584,12 +606,18 @@ pub fn emit(
         {
             let again = widen(c, native, p, std::env::var_os("VICTORY_WAVE_WHY").is_some());
             match again {
-                Ok(_) => again,
+                Ok(_) => {
+                    set_form(if host32_w64(native) { "w64 wave-wide (lane mask)" } else { "wave-wide (lane mask)" });
+                    again
+                }
                 Err(_) => emit_program(c, native, p, st, io, res),
             }
         }
         other => other,
     };
+    if host32_w64(native) && matches!(&original, Ok((_, Some(64)))) {
+        set_form("w64 STILL WAVE64");
+    }
     let original = original.map(|(shader, _)| shader);
     // Retry unsupported programs only. Threading can separate an unknown
     // register value from its apparent use on a mutually exclusive path.
@@ -604,7 +632,15 @@ pub fn emit(
     let st = structure::structure(&p)?;
     let io = io::raise(&p, native)?;
     let res = resource::track(&p, &native.layout, native.stage, Some(&st))?;
-    emit_program(c, native, &p, &st, &io, &res).map(|(shader, _)| shader)
+    let last = emit_program(c, native, &p, &st, &io, &res);
+    {
+        let f = take_form();
+        set_form(&format!("{f} +threaded edges"));
+    }
+    if host32_w64(native) && matches!(&last, Ok((_, Some(64)))) {
+        set_form("w64 STILL WAVE64 +threaded edges");
+    }
+    last.map(|(shader, _)| shader)
 }
 
 /// `p` (a console wave64 compute program on a host of 32-lane waves) with the per-lane
```

</details>
