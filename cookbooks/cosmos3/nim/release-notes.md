<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Cosmos3 Certified NIM release notes

## 2.1.0

Image: `nvcr.io/nim/nvidia/cosmos3:2.1.0`.

### New models and features

- **Edge Reasoner** (`NIM_MODEL_VARIANT=edge`): a smaller Reasoner with BF16 and
  FP8 profiles and a 12-GiB per-device floor. Edge has no DFlash draft. See
  [Edge verified GPUs](support-matrix.md#edge-verified-gpus).
- **Edge-DROID policy** (`NIM_MODEL_VARIANT=edge-droid`, experimental): a BF16
  Generator policy checkpoint with action-only output. This capability is
  experimental while NVIDIA verifies it with benchmarks. See
  [Action](action.md#edge-droid-and-nano-droid-policy).
- **Streaming video sessions** for the Reasoner over WebSocket and REST, enabled
  with `NIM_ENABLE_STREAMING=true`. See [Streaming](streaming.md).
- **Dynamo deployment** for the Reasoner, including REST streaming. See
  [Dynamo deployment](dynamo/dynamo_deployment.md).
- **Reasoner optimization presets** (`NIM_OPTIMIZATION_PRESET=TTFT` or `TPOT`),
  experimental. See [Configuration](configuration.md#optimization-presets).
- **Generator Hugging Face BYOC**: `NIM_MODEL_PATH` accepts
  `hf://owner/repository[:revision]` for the Generator as well as the Reasoner.
- **Transfer control weights**: each control object accepts `control_weight`;
  weights are normalized across enabled controls. See
  [Transfer](transfer.md#advanced-multiple-controls).
- **AVI video input** for both runtimes.

### Behavior changes

- **Transfer guidance**: when control and text guidance are both active, the
  control term is no longer also multiplied by `guidance_scale`. Combined-guidance
  Transfer outputs differ from 2.0.0. See
  [Control guidance behavior](transfer.md#control-guidance-behavior).
- **Generator performance defaults**: SeaCache diffusion caching is on by default
  for every Generator variant except Edge-DROID (`NIM_CACHE_BACKEND`,
  `NIM_CACHE_CONFIG`). SageAttention 2.2 is the default attention backend on
  compute capability 8.0 through 9.x without sequence parallelism. The
  channels-last VAE decode path is the default (`NIM_VAE_FAST_PATH`). Set
  `NIM_CACHE_BACKEND=none` or `NIM_VAE_FAST_PATH=lossless` when you need
  output closer to the 2.0.0 reference path.
- **Multi-GPU Generator layouts**: latency profiles use guidance and sequence
  parallelism; throughput profiles run one engine replica per GPU. New
  selectors: `NIM_NUM_GPUS`, `NIM_TENSOR_PARALLEL_SIZE`,
  `NIM_GUIDANCE_PARALLEL_SIZE`, `NIM_CONTEXT_PARALLEL_SIZE`, and
  `NIM_DATA_PARALLEL_SIZE`.
- **System-memory floors are enforced**: each profile now carries its measured
  system-memory floor instead of a 16-GiB admission tag, and FP8 offload floors
  are lower. See the [Support matrix](support-matrix.md#generator-configurations).
- **Unified memory**: on DGX Spark and Jetson AGX Thor, 70% of the shared pool
  counts toward profile floors, and the Reasoner defaults to
  `NIM_GPU_MEMORY_UTILIZATION=0.70`. Super Generator profiles no longer fit
  these systems.
- **Reasoner headroom**: `NIM_GPU_MEMORY_HEADROOM_GIB` defaults to 4 GiB on
  discrete GPUs.
- **Validation**: `flow_shift` must be greater than 0.

### Renamed or removed variables

| 2.0.0 | 2.1.0 |
| --- | --- |
| `NIM_OFFLOAD_TEXT_GUARDRAIL` | `NIM_TEXT_GUARD_OFFLOAD` profile selector (`resident` or `offloaded`) |
| `NIM_OFFLOAD_VIDEO_GUARDRAIL` | `NIM_VIDEO_GUARD_OFFLOAD` profile selector (`resident` or `offloaded`) |
| `NIM_UNIFIED_MEMORY_HOST_RESERVE_GIB` | Removed; unified-memory systems use a fixed 70% share |
| `NIM_QWEN3GUARD_MAX_GPU_MEMORY_GB` | Removed |

## 2.0.0

This is the first release of the unified Cosmos3 Certified NIM. One container
image includes both runtime choices:

- **Generator** serves image and video generation, Action, and Transfer through
  `POST /v1/infer`.
- **Reasoner** serves OpenAI-compatible image and video understanding through
  Chat Completions and the Responses API.

Each container starts one selected runtime. See [Deployment](deployment.md) for
the current image and launch instructions.
