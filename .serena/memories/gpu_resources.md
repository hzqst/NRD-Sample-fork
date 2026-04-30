# gpu_resources

## Overview
`Source/NRDSample.cpp` 中显式创建的 GPU resources 主要分为持久资源和初始化临时资源。持久资源由 `m_AccelerationStructures`、`m_Buffers`、`m_Textures` 管理，支撑路径追踪、NRD 降噪、SHARC、DLSS/TAA/NIS 后处理以及最终拷贝到 swapchain。

本 memory 的“引用者”以已确认的 C++ 函数、descriptor set、GPU pass 和 shader 文件为准；descriptor、pipeline、fence、command buffer 这类执行/绑定/同步对象不作为 GPU data resource 清单主体，仅在 Notes 中说明边界。

## Responsibilities
- 记录 `NRDSample.cpp` 中通过 `CreateAccelerationStructures`、`CreateResourcesAndDescriptors`、`CreateTexture`、`CreateBuffer`、`CreateSwapChain` 显式创建或取得的 GPU data resources。
- 说明每类 acceleration structure、buffer、texture 的用途、主要写入者和主要读取/引用者。
- 标注初始化阶段用于 BLAS build/compaction 的临时 GPU/host-visible 资源，避免把它们误认为每帧持久资源。
- 说明资源状态与 descriptor 绑定的关键关系，便于后续修改 pass、资源格式或 descriptor layout 时定位影响面。

## Involved Files (no line numbers)
- Source/NRDSample.cpp
- Shaders/MorphMeshUpdateVertices.cs.hlsl
- Shaders/MorphMeshUpdatePrimitives.cs.hlsl
- Shaders/SharcUpdate.cs.hlsl
- Shaders/SharcResolve.cs.hlsl
- Shaders/ConfidenceBlur.cs.hlsl
- Shaders/TraceOpaque.cs.hlsl
- Shaders/Composition.cs.hlsl
- Shaders/TraceTransparent.cs.hlsl
- Shaders/Taa.cs.hlsl
- Shaders/DlssBefore.cs.hlsl
- Shaders/DlssAfter.cs.hlsl
- Shaders/Final.cs.hlsl
- Shaders/Include/RaytracingShared.hlsli
- Shaders/Include/Shared.hlsli

## Architecture
初始化创建链路：`Sample::Initialize` 先按场景大小 resize `m_Buffers`、`m_Textures`、`m_TextureStates`、`m_AccelerationStructures`，然后调用 `CreateSwapChain`、`CreateAccelerationStructures`、`CreateResourcesAndDescriptors`、`CreateDescriptorSets`、`UploadStaticData`。

资源创建 helper：
- `CreateTexture`：在 `NriDeviceHeap` 中创建 placed 2D texture；所有非 read-only texture 都创建 SRV 和 UAV，read-only texture 只创建 SRV；如果提供 `initialAccess`，同时写入 `m_TextureStates` 的初始状态。
- `CreateBuffer`：在 `NriDeviceHeap` 中创建 placed buffer；按 usage 创建 structured buffer SRV 和/或 storage structured buffer UAV。
- `CreateAccelerationStructures`：创建 `TLAS_World` / `TLAS_Emissive`，创建静态/动态 BLAS，随后用 compacted BLAS 替换临时 BLAS；初始化阶段的 upload/scratch/readback/query 资源在函数末尾销毁。
- `CreateSwapChain`：创建 swapchain，取得 swapchain textures，为每个 back buffer 创建 color attachment view 和 acquire/release semaphore，并设置 debug name `Texture::SwapChain#N`。
- `UploadStaticData`：上传场景 read-only texture、初始化 texture state，并上传 `PrimitiveData`、`MorphMeshIndices`、`MorphMeshVertices`。

每帧引用链路：
```text
PrepareFrame
  -> GatherInstanceData
       -> Stream InstanceData
       -> Stream TLAS_World / TLAS_Emissive instance data
RenderFrame
  -> Streamer copy
  -> optional Morph buffer update and Morph BLAS build/update
  -> TLAS_World / TLAS_Emissive build
  -> RestoreBindings root descriptors and global descriptor sets
  -> SHARC update/resolve/confidence blur
  -> TraceOpaque
  -> NRD SIGMA / REBLUR / RELAX / REFERENCE denoise
  -> Composition
  -> TraceTransparent
  -> DLSS or TAA
  -> NIS
  -> Final
  -> copy Texture::Final to swapchain texture
```

### Acceleration Structures
| 资源 | 作用 | 确认引用者 |
| --- | --- | --- |
| `TLAS_World` | 每帧重建的主场景 top-level AS，包含 static opaque/transparent 与 dynamic objects，用于主路径追踪和 SHARC ray query。 | 创建：`CreateAccelerationStructures`；descriptor：`CreateResourcesAndDescriptors`；构建：`RenderFrame` 的 `TLAS` pass；绑定：`RestoreBindings` root `gWorldTlas`；读取：`Shaders/Include/RaytracingShared.hlsli`、`TraceOpaque.cs.hlsl`、`TraceTransparent.cs.hlsl`、`SharcUpdate.cs.hlsl`。 |
| `TLAS_Emissive` | 每帧重建的 emissive-only top-level AS，用于发光体采样、候选光线可见性等逻辑。 | 创建/descriptor/构建同 `TLAS_World`；绑定：`RestoreBindings` root `gLightTlas`；读取：`RaytracingShared.hlsli` 中 emissive/light sampling 相关函数，以及引用该 include 的 trace/SHARC pass。 |
| `BLAS_MergedOpaque` | 合并后的静态非透明几何 BLAS，作为 `TLAS_World` 的一个或一组 instance 来源。 | 创建并 compact：`CreateAccelerationStructures`；引用：`GatherInstanceData` 将其 handle 写入 `m_WorldTlasData`；读取：通过 `TLAS_World` 被 ray tracing shader 间接访问。 |
| `BLAS_MergedTransparent` | 合并后的静态透明几何 BLAS，供透明路径追踪使用。 | 创建并 compact：`CreateAccelerationStructures`；引用：`GatherInstanceData` 写入 `m_WorldTlasData`，mask 为 transparent；读取：通过 `TLAS_World` 被 `TraceTransparent` / ray query 间接访问。 |
| `BLAS_MergedEmissive` | 合并后的静态 emissive 几何 BLAS，供 `TLAS_Emissive` 使用。 | 创建并 compact：`CreateAccelerationStructures`；引用：`GatherInstanceData` 写入 `m_LightTlasData`；读取：通过 `TLAS_Emissive` 被 light sampling/visibility 查询间接访问。 |
| `BLAS_Other` 及之后追加的 dynamic BLAS | 动态或可更新 mesh 的独立 BLAS；morph mesh 会在每帧根据变形后的 vertex buffer build/update。`BLAS_Other` 是动态 BLAS 区间的起始枚举，不是单个固定 BLAS。 | 创建并 compact：`CreateAccelerationStructures`；引用：`GatherInstanceData` 使用 `meshInstance.blasIndex` 写入 TLAS instance；更新：`RenderFrame` 的 `Morph mesh: BLAS` pass；读取：通过 `TLAS_World`/`TLAS_Emissive` 间接访问。 |

### Buffers
| 资源 | 作用 | 确认引用者 |
| --- | --- | --- |
| `MorphMeshIndices` | morph mesh 的 compact index buffer，既用于 primitive update shader，也用于 morph BLAS build/update 的 index input。 | 创建：`CreateResourcesAndDescriptors`；上传：`UploadStaticData`；绑定：`DescriptorSet::MorphTargetUpdatePrimitives`；读取：`MorphMeshUpdatePrimitives.cs.hlsl`；BLAS 引用：`RenderFrame` 的 `Morph mesh: BLAS`。 |
| `MorphMeshVertices` | morph base pose 和 target vertex 数据，供 morph vertex compute pass 生成当前帧位置/属性。 | 创建：`CreateResourcesAndDescriptors`；上传：`UploadStaticData`；绑定：`DescriptorSet::MorphTargetPose`；读取：`MorphMeshUpdateVertices.cs.hlsl`。 |
| `MorphPositions` | 当前/上一帧 morph 后的位置 buffer，按 `MAX_ANIMATION_HISTORY_FRAME_NUM` 存储历史；也是 morph BLAS 的 vertex input。 | 创建：`CreateResourcesAndDescriptors`；写入：`MorphMeshUpdateVertices.cs.hlsl`；读取：`MorphMeshUpdatePrimitives.cs.hlsl`；BLAS 引用：`RenderFrame` 的 `Morph mesh: BLAS`。 |
| `MorphAttributes` | morph 后的 normal/tangent 等属性，供 primitive data 更新使用。 | 创建：`CreateResourcesAndDescriptors`；写入：`MorphMeshUpdateVertices.cs.hlsl`；读取：`MorphMeshUpdatePrimitives.cs.hlsl`。 |
| `MorphPrimitivePositions` | morph primitive 的上一帧三角形顶点位置，用于 ray tracing shader 计算 previous state / motion 相关信息。 | 创建：`CreateResourcesAndDescriptors`；写入：`MorphMeshUpdatePrimitives.cs.hlsl`；绑定：`RestoreBindings` root `gIn_MorphPrimitivePositionsPrev`；读取：`RaytracingShared.hlsli`。 |
| `MorphMeshScratch` | morph BLAS build/update scratch buffer。 | 创建：`CreateResourcesAndDescriptors`，大小来自 `CreateAccelerationStructures` 汇总的 `m_MorphMeshScratchSize`；引用：`RenderFrame` 的 `Morph mesh: BLAS`。 |
| `InstanceData` | 每帧 instance 级材质索引、flags、transform/previous transform、primitive offset、morph offset 等 shader 数据。 | 创建：`CreateResourcesAndDescriptors`；写入：`GatherInstanceData` 通过 streamer 上传，`RenderFrame` 执行 copy；绑定：`RestoreBindings` root `gIn_InstanceData`；读取：`RaytracingShared.hlsli`、trace/SHARC shader。 |
| `PrimitiveData` | 每个 primitive 的 uv、normal、tangent、面积等数据；静态部分初始化上传，morph primitive 可被每帧改写。 | 创建：`CreateResourcesAndDescriptors`；初始化：`UploadStaticData`；写入：`MorphMeshUpdatePrimitives.cs.hlsl`；绑定：`RestoreBindings` root `gIn_PrimitiveData`；读取：`RaytracingShared.hlsli`。 |
| `SharcHashEntries` | SHARC hash table storage buffer。 | 创建：`CreateResourcesAndDescriptors`；绑定：`DescriptorSet::Sharc`；读写：`SharcUpdate.cs.hlsl`、`SharcResolve.cs.hlsl`、`TraceOpaque.cs.hlsl`、`TraceTransparent.cs.hlsl` 经 `RaytracingShared.hlsli` 访问。 |
| `SharcAccumulated` | SHARC accumulated radiance/cache storage buffer，首帧会清零。 | 创建：`CreateResourcesAndDescriptors`；清零：`RenderFrame` 首帧 `CmdZeroBuffer`；绑定：`DescriptorSet::Sharc`；读写：SHARC update/resolve 与 trace pass。 |
| `SharcResolved` | SHARC resolved cache storage buffer，供后续 ray tracing pass 读取 cache 结果。 | 创建：`CreateResourcesAndDescriptors`；绑定：`DescriptorSet::Sharc`；写入：`SharcResolve.cs.hlsl`；读取/引用：trace pass 和 SHARC shader。 |
| `WorldScratch` | `TLAS_World` build scratch buffer。 | 创建：`CreateResourcesAndDescriptors`，大小来自 `GetAccelerationStructureBuildScratchBufferSize(TLAS_World)`；引用：`RenderFrame` 的 `TLAS` pass。 |
| `LightScratch` | `TLAS_Emissive` build scratch buffer。 | 创建：`CreateResourcesAndDescriptors`，大小来自 `GetAccelerationStructureBuildScratchBufferSize(TLAS_Emissive)`；引用：`RenderFrame` 的 `TLAS` pass。 |

### Textures
| 资源 | 作用 | 确认引用者 |
| --- | --- | --- |
| `ViewZ` | render-resolution view depth / linear viewZ，NRD、Composition、DLSS guide 都依赖它；DLSS before pass 会在 SR 路径中把它改写为 upscaler 需要的 depth 表示。 | 写入：`TraceOpaque.cs.hlsl`，`DlssBefore.cs.hlsl` 可改写；读取：`Denoise` 的 `IN_VIEWZ`、`Composition.cs.hlsl`、DLSS/DLRR dispatch。 |
| `Mv` | motion vector 与 TAA mask/辅助值。 | 写入：`TraceOpaque.cs.hlsl`、`TraceTransparent.cs.hlsl`；读取：`Denoise` 的 `IN_MV`、`Taa.cs.hlsl`、DLSS/DLRR dispatch。 |
| `Normal_Roughness` | packed normal/roughness/material id，供 NRD、合成和 DLSS guide 使用。 | 写入：`TraceOpaque.cs.hlsl`、`TraceTransparent.cs.hlsl`；读取：`Denoise` 的 `IN_NORMAL_ROUGHNESS`、`Composition.cs.hlsl`、`DlssBefore.cs.hlsl`、DLSS/DLRR dispatch。 |
| `PsrThroughput` | path-space regularization throughput，用于后续合成修正直接光。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Composition.cs.hlsl`。 |
| `BaseColor_Metalness` | 当前像素的 base color 与 metalness。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Composition.cs.hlsl`、`DlssBefore.cs.hlsl`。 |
| `DirectLighting` | path tracing 输出的直接光。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Composition.cs.hlsl`。 |
| `DirectEmission` | path tracing 输出的直接自发光。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Composition.cs.hlsl`。 |
| `Shadow` | SIGMA 输出的降噪 shadow/translucency 结果。 | 写入：`Denoise` 的 `OUT_SHADOW_TRANSLUCENCY`；读取：`Composition.cs.hlsl`。 |
| `Diff` | NRD 输出的降噪 diffuse radiance / hit distance / occlusion 数据。 | 写入：`Denoise` 的 diffuse output；读取：`Composition.cs.hlsl`。 |
| `Spec` | NRD 输出的降噪 specular radiance / hit distance / occlusion 数据。 | 写入：`Denoise` 的 specular output；读取：`Composition.cs.hlsl`。 |
| `Unfiltered_Penumbra` | `TraceOpaque` 输出的 raw penumbra/shadow signal，供 SIGMA 使用。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Denoise` 的 `IN_PENUMBRA`。 |
| `Unfiltered_Diff` | `TraceOpaque` 输出的 raw diffuse signal；在不同 NRD mode 下可作为 diffuse radiance/hitdist、directional occlusion 或 SH0 输入。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Denoise` 的 diffuse input。 |
| `Unfiltered_Spec` | `TraceOpaque` 输出的 raw specular signal/hit distance。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Denoise` 的 specular input、`DlssBefore.cs.hlsl` 的 spec hit distance guide。 |
| `Unfiltered_Translucency` | `TraceOpaque` 输出的 raw translucency signal。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Denoise` 的 `IN_TRANSLUCENCY`。 |
| `Validation` | NRD validation overlay 输出。 | 写入：`Denoise` 的 `OUT_VALIDATION`；读取：`Final.cs.hlsl`。 |
| `Composed` | 透明 pass 后的 render-resolution 合成色，也是 Reference denoiser 的 in/out signal，随后进入 DLSS/TAA/Final。 | 写入：`TraceTransparent.cs.hlsl`、`Denoise` 的 `REFERENCE` in-place；读取：`Taa.cs.hlsl`、DLSS dispatch、`Final.cs.hlsl`。 |
| `Gradient_StoredPing` | SHARC radiance/history ping-pong 之一。 | 读写：`SharcUpdate.cs.hlsl`，通过 `DescriptorSet::SharcUpdatePing/Pong` 在奇偶帧间作为 previous/current 切换。 |
| `Gradient_StoredPong` | SHARC radiance/history ping-pong 之一。 | 读写：`SharcUpdate.cs.hlsl`，与 `Gradient_StoredPing` 成对切换。 |
| `Gradient_Ping` | SHARC gradient/confidence 中间 texture；最终 blur 轮次后也可能作为 ping-pong 输入。 | 写入：`SharcUpdate.cs.hlsl`；读写：`ConfidenceBlur.cs.hlsl`；参与 `BuildOptimizedTransitions` 的 storage/read 状态切换。 |
| `Gradient_Pong` | confidence blur ping-pong texture；最终作为 NRD diffuse/spec confidence 输入。 | 写入/读取：`ConfidenceBlur.cs.hlsl`；读取：`Denoise` 的 `IN_DIFF_CONFIDENCE` 与 `IN_SPEC_CONFIDENCE`。 |
| `ComposedDiff` | opaque composition 的 diffuse history/intermediate；下一帧 `TraceOpaque` 用它做 reprojection，当前帧 transparent pass 读取它作为 opaque base。 | 写入：`Composition.cs.hlsl`；读取：`TraceOpaque.cs.hlsl`、`TraceTransparent.cs.hlsl`。 |
| `ComposedSpec_ViewZ` | opaque composition 的 specular + previous viewZ history/intermediate。 | 写入：`Composition.cs.hlsl`；读取：`TraceOpaque.cs.hlsl`、`TraceTransparent.cs.hlsl`。 |
| `TaaHistoryPing` | TAA history ping-pong 之一。 | 读写：`Taa.cs.hlsl` 通过 `DescriptorSet::TaaPing/Pong` 切换；读取：NIS dispatch 在 DLSS disabled 路径中作为输入。 |
| `TaaHistoryPong` | TAA history ping-pong 之一。 | 读写：`Taa.cs.hlsl`；读取：NIS dispatch 在 DLSS disabled 路径中作为输入。 |
| `DlssOutput` | DLSS/DLRR output-resolution 输出 texture，`DlssAfter` 会原地处理。 | 写入：`CmdDispatchUpscale` 的 DLSR/DLRR output、`DlssAfter.cs.hlsl`；读取：NIS dispatch。 |
| `PreFinal` | NIS upscaler/sharpener 输出，供最终 pass 使用。 | 写入：NIS `CmdDispatchUpscale`；读取：`Final.cs.hlsl`。 |
| `Final` | swapchain format 的最终 image，先由 `Final.cs.hlsl` 写入，再 copy 到当前 back buffer。 | 写入：`Final.cs.hlsl`；读取/复制：`RenderFrame` 的 `Copy to back-buffer`。 |
| `Unfiltered_DiffSh` | `NRD_MODE == SH` 时的 raw diffuse SH1 输入。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Denoise` 的 SH diffuse input。 |
| `Unfiltered_SpecSh` | `NRD_MODE == SH` 时的 raw specular SH1 输入。 | 写入：`TraceOpaque.cs.hlsl`；读取：`Denoise` 的 SH specular input。 |
| `DiffSh` | `NRD_MODE == SH` 时的 diffuse SH1 denoised output。 | 写入：`Denoise`；读取：`Composition.cs.hlsl`。 |
| `SpecSh` | `NRD_MODE == SH` 时的 specular SH1 denoised output。 | 写入：`Denoise`；读取：`Composition.cs.hlsl`。 |
| `RRGuide_DiffAlbedo` | Ray Reconstruction / DLSS denoiser guide 的 diffuse albedo。 | 写入：`DlssBefore.cs.hlsl`；读取：DLRR dispatch guide。 |
| `RRGuide_SpecAlbedo` | Ray Reconstruction / DLSS denoiser guide 的 specular albedo。 | 写入：`DlssBefore.cs.hlsl`；读取：DLRR dispatch guide。 |
| `RRGuide_SpecHitDistance` | Ray Reconstruction / DLSS denoiser guide 的 specular hit distance。 | 写入：`DlssBefore.cs.hlsl`；读取：DLRR dispatch guide 的 `specularMvOrHitT`。 |
| `RRGuide_Normal_Roughness` | Ray Reconstruction / DLSS denoiser guide 的 normal/roughness，代码注释说明只支持 `RGBA16f` encoding。 | 写入：`DlssBefore.cs.hlsl`；读取：DLRR dispatch guide。 |
| `BaseReadOnlyTexture + i` | 场景只读 texture 数组，包含 static blue-noise/sampling texture 与材质 baseColor、roughnessMetalness、normal、emissive texture；每个 `m_Scene.textures[i]` 都创建一个 read-only SRV。 | 创建：`CreateResourcesAndDescriptors` 循环；上传：`UploadStaticData`；绑定：`DescriptorSet::RayTracing` 的 static textures 与 material bindless array；读取：`RaytracingShared.hlsli`、`TraceOpaque.cs.hlsl` 等 ray tracing shader。 |

### Swapchain And Presentation Resources
| 资源 | 作用 | 确认引用者 |
| --- | --- | --- |
| `SwapChainTexture::texture` | swapchain back buffer，由 `CreateSwapChain` 通过 `NRI.GetSwapChainTextures` 取得，不在 `Texture` enum 中。 | 写入：`RenderFrame` 先把 `Texture::Final` copy 到当前 back buffer，再作为 UI color attachment；呈现：`QueuePresent`。 |
| `SwapChainTexture::colorAttachment` | back buffer 的 color attachment descriptor，不是 texture 本体。 | 创建：`CreateSwapChain`；引用：`RenderFrame` 的 UI pass attachment。 |

### Initialization-Time Temporary Resources
| 资源 | 作用 | 生命周期 |
| --- | --- | --- |
| `uploadBuffer` | HOST_UPLOAD buffer，临时存放 BLAS build 所需的 vertices、indices、static transforms。 | `CreateAccelerationStructures` 内创建、map、填充，BLAS build/compaction 完成后 unmap 并 destroy。 |
| `scratchBuffer` | 初始化阶段批量 BLAS build scratch buffer。 | `CreateAccelerationStructures` 内创建，`CmdBuildBottomLevelAccelerationStructures` 使用后 destroy。 |
| `readbackBuffer` | HOST_READBACK buffer，用于读取每个 BLAS 的 compacted size。 | `CmdCopyQueries` 写入后 CPU map，compaction 完成后 destroy。 |
| `queryPool` | acceleration-structure compacted-size query pool。 | 用于 `CmdWriteAccelerationStructuresSizes`，读取后 destroy。 |
| 临时 committed BLAS | 初始 build 的未压缩 BLAS。 | 创建后参与 build 和 compact query；compacted placed BLAS 创建并 copy 完成后，临时 BLAS 被 destroy，`m_AccelerationStructures` 中的指针被替换为 compacted BLAS。 |

## Dependencies
- NRI Core：placed/committed texture、buffer、acceleration structure、descriptor view、barrier、copy、command submission。
- NRI RayTracing：TLAS/BLAS 创建、build/update、scratch size 查询、compaction query、AS descriptor/root binding。
- NRI Streamer：`InstanceData` 每帧上传、TLAS instance data 上传、dynamic constant buffer。
- NRD Integration：`Denoise` 通过 `nrd::ResourceSnapshot` 读取/写入 `Mv`、`Normal_Roughness`、`ViewZ`、`Validation`、unfiltered signals、`Diff`、`Spec`、`Shadow`、confidence gradient、SH/occlusion/reference resources。
- NRI Upscaler：DLSR/DLRR/NIS dispatch 读取 DLSS/TAA inputs 和 guide textures，写 `DlssOutput`、`PreFinal`。
- HLSL passes：MorphMeshUpdate、SHARC、ConfidenceBlur、TraceOpaque、Composition、TraceTransparent、TAA、DLSS helper、Final。
- Scene data：`m_Scene.textures`、materials、instances、meshInstances、morph vertices/indices/primitives 决定 resource size、bindless texture 数量和 BLAS/TLAS instance 内容。

## Notes
- `m_TextureStates` 只覆盖非 read-only textures；scene read-only textures 初始化上传后保持 shader resource 状态，不参与每帧 `BuildOptimizedTransitions`。
- `CreateTexture` 对所有非 read-only texture 都创建 storage view，因此即使初始状态是 SRV，后续 pass 仍可通过 UAV 写入；状态正确性依赖 `RenderFrame` 中的 barrier 和 `Denoise` 后的 state 回写。
- `ComposedDiff` 与 `ComposedSpec_ViewZ` 既是当前帧 composition output，也是下一帧 opaque trace 的 history input；修改它们的格式或清理逻辑会影响 reprojection 和透明 pass。
- `Gradient_Pong` 是 NRD confidence 的最终输入；confidence blur 循环次数目前为奇数，保证输出落到 `Gradient_Pong`。
- `Texture::ViewZ` 在 DLSS SR 路径中会被 `DlssBefore` 改写为 upscaler 需要的 depth 表示；同一帧后续 DLSS dispatch 读取的是改写后的状态。
- `MorphPositions` 同时是 shader storage/read buffer 和 AS build input；改动 usage 或 state transition 时要同步检查 morph compute 与 BLAS update。
- `PrimitiveData` 同时有静态上传数据和 morph pass 动态覆盖数据；shader 端依赖 `InstanceData.primitiveOffset` 与 BLAS geometry layout 顺序匹配。
- `BLAS_Other` 是动态 BLAS 区间起点，不等于一个固定资源；动态 BLAS 数量由 scene 中 `allowUpdate` mesh 决定。
- `Descriptor::Constant_Buffer` 指向 streamer-owned dynamic constant buffer，不在 `Buffer` enum 中；它是每个 compute pass 和 morph pass root constants/constant data 的依赖。
- 本清单不把 pipeline、pipeline layout、descriptor pool、descriptor set、fence、command allocator/buffer、upscaler object 计入 GPU data resource；它们是执行、绑定或同步对象。

## Callers
- `Sample::Initialize` 调用 `CreateSwapChain`、`CreateAccelerationStructures`、`CreateResourcesAndDescriptors`、`CreateDescriptorSets`、`UploadStaticData` 完成资源初始化。
- `Sample::PrepareFrame` 调用 `GatherInstanceData`，准备 `InstanceData`、`TLAS_World` 和 `TLAS_Emissive` 的每帧输入数据。
- `Sample::RenderFrame` 是所有持久 GPU resources 的主要每帧引用者，负责 streamer copy、morph update、TLAS build、SHARC、trace、NRD、composition、upscaler、final copy 和 UI pass。
- `Sample::~Sample` 统一销毁 `m_Textures`、`m_Buffers`、`m_AccelerationStructures`、swapchain descriptors/fences 以及相关 NRI 对象。