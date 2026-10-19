<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Configure the Cosmos3 Certified NIM

This page lists launch-time environment variables intended for users and
operators. Start with the model-selection variables and keep automatic defaults
unless the workload requires an override. See [Deployment](deployment.md) for
complete Docker commands.

## Essential configuration

### Authentication and cache

| Name | Default | Use |
| --- | --- | --- |
| `NGC_API_KEY` | Empty | Download required model and runtime artifacts when they are not already cached |
| `NIM_CACHE_PATH` | `/opt/nim/.cache` | Set the writable in-container artifact cache |

### Choose the runtime and model

| Name | Default | Use |
| --- | --- | --- |
| `NIM_MODEL_TYPE` | `generator` | Select `generator` or `reasoner` |
| `NIM_MODEL_VARIANT` | `nano` preference | Reasoner: `edge`, `nano`, or `super`. Generator: `nano`, `super`, `edge-droid`, `nano-droid`, `super-t2i`, `super-t2i-4step`, `super-i2v`, or `super-i2v-4step` |
| `NIM_PRECISION` | Generator: FP8 preference; Reasoner: GPU-derived preference | Optionally pin a precision available in the selected image; when omitted, Reasoner prefers BF16 on compute capability 8.0 through 8.8, FP8 on 8.9 through 9.x, and NVFP4 on 10.0 or newer when compatible |
| `NIM_PERF_PROFILE` | `latency` | Generator only: choose `latency` or `throughput` |

`NIM_MODEL_VARIANT` selects the checkpoint contract for either runtime.
Reasoner accepts `edge`, `nano`, or `super` and does not use `NIM_PERF_PROFILE`.
Edge Reasoner has BF16 and FP8 profiles; Edge-DROID and Nano-DROID have BF16
profiles only. General-purpose Edge generation is unsupported.

The NIM chooses the best compatible profile for these settings and the visible
GPUs. A normal deployment does not need a profile ID.

### Bring your own checkpoint

| Name | Default | Use |
| --- | --- | --- |
| `NIM_MODEL_PATH` | Empty | Absolute local directory or `hf://owner/repository[:revision]` for either runtime |
| `NIM_DFLASH_MODEL_PATH` | Empty | Nano or Super Reasoner: independently override the DFlash draft with an absolute local directory |
| `NIM_DISABLE_MODEL_DOWNLOAD` | `false` | Disable profile download for a completely local Reasoner override; incompatible with Reasoner `hf://` and rejected for Generator |
| `HF_TOKEN` | Empty | Authenticate to a private Hugging Face repository named by `NIM_MODEL_PATH` |

See [Bring your own checkpoint](bring-your-own-checkpoint.md) for layouts,
mounts, and compatibility checks.

### Server and logging

| Name | Default | Use |
| --- | --- | --- |
| `NIM_HTTP_API_PORT` | `8000` | Set the container HTTP port |
| `NIM_LOG_LEVEL` | `INFO` | Set the service logging threshold |
| `NIM_LOGGING_JSONL` | `false` | Emit JSON-line logs |

## Advanced profile controls

Use these only when automatic model selection is not sufficient:

| Name | Default | Use |
| --- | --- | --- |
| `NIM_OFFLOAD_MODE` | Automatic compatible preference | Generator only: request `none`, `model`, or `layer` when the selected model provides that mode |
| `NIM_TEXT_GUARD_OFFLOAD` | Automatic; `resident` preferred | Generator only: select profiles whose text guardrail is `resident` or `offloaded` during diffusion |
| `NIM_VIDEO_GUARD_OFFLOAD` | Automatic; `resident` preferred | Generator only: select profiles whose output guardrails are `resident` or `offloaded` during diffusion |
| `NIM_NUM_GPUS` | Automatic | Select profiles that use exactly this many GPUs (`n_gpus` tag) |
| `NIM_TENSOR_PARALLEL_SIZE` | Automatic | Select profiles with this tensor-parallel size (`nim_tp` tag); Reasoner Super BF16 offers TP1 and TP2 |
| `NIM_GUIDANCE_PARALLEL_SIZE` | Automatic | Generator only: select profiles with this classifier-free-guidance parallel size (`nim_gp` tag) |
| `NIM_CONTEXT_PARALLEL_SIZE` | Automatic | Generator only: select profiles with this sequence (Ulysses) parallel size (`nim_up` tag) |
| `NIM_DATA_PARALLEL_SIZE` | Automatic | Generator only: select profiles with this many data-parallel engine replicas (`nim_dp` tag) |
| `NIM_GPU_MEMORY_HEADROOM_GIB` | `4` | Reasoner only: reserve this many binary GiB of free memory per discrete GPU during profile selection and runtime sizing |
| `NIM_TAGS_SELECTOR` | Empty | Filter by comma-separated exact manifest tags |
| `NIM_MODEL_PROFILE` | Empty | Pin an exact profile ID from the current image |

The GPU-count and parallelism variables are shorthand for the matching manifest
tags. They filter the image's existing profiles and cannot create a layout the
manifest does not contain. A selection that matches no profile fails at startup.

Without these selectors, the Generator uses as many visible GPUs as a
compatible profile supports (1, 2, 4, or 8). A multi-GPU `latency` profile
splits one request across GPUs with guidance parallelism 2 and sequence
parallelism `n_gpus / 2`. A multi-GPU `throughput` profile runs one
single-GPU engine replica per GPU (`nim_dp = n_gpus`). There is no one-GPU
`throughput` profile, and model or layer offload is available only on one GPU.
To keep the Generator on fewer GPUs, limit the visible devices with Docker
`--gpus` or set `NIM_NUM_GPUS`.

Do not combine shorthand variables with conflicting values in
`NIM_TAGS_SELECTOR`. Use the `model_variant` tag when filtering by model. Exact
tags and profile IDs are tied to an image release and are less portable than
automatic selection.

On an integrated GPU with unified host/device memory, such as DGX Spark (GB10)
or Jetson AGX Thor, the NIM treats 70% of the shared pool as GPU memory. It
uses that share for profile floors and as the default Reasoner
`NIM_GPU_MEMORY_UTILIZATION`. Earlier images used
`NIM_UNIFIED_MEMORY_HOST_RESERVE_GIB`; this image ignores that variable.

## Generator configuration

### Input and output

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ALLOW_URL_INPUT` | `true` | Allow HTTP(S) image, video, and Transfer inputs; set `false` to require encoded input |
| `NIM_VIDEO_SAVE_QUALITY` | `7` | Set VP9 output quality from 1 through 9; this affects encoding, not diffusion quality |
| `NIM_TRITON_REQUEST_TIMEOUT` | 30 minutes (`1800000000` microseconds) | Set the queue-plus-execution timeout in microseconds |

Client HTTP timeouts do not change this backend ceiling. The Transfer example
uses a 60-minute client timeout; to allow the backend the same ceiling, add
`-e NIM_TRITON_REQUEST_TIMEOUT=3600000000` to the Generator Docker launch
command and restart the container. See the [Transfer hardware and timeout
guidance](transfer.md#run-the-examples).

### Startup and execution

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_WARMUP` | `false` | Run synthetic inference before readiness |
| `NIM_ENABLE_TORCH_COMPILE` | `true` | Enable the Generator compilation path |
| `NIM_CACHE_BACKEND` | Edge-DROID: `none`; other Generator variants: `sea_cache` | Generator diffusion cache: `sea_cache`, `cache_dit`, or `none` |
| `NIM_CACHE_CONFIG` | Empty object | Override individual diffusion-cache settings as a JSON object; rejected when the backend is `none` |
| `NIM_LINEAR_BACKEND` | Quantized profiles: `cutlass`; BF16: unset | Select `auto`, `cutlass`, `flashinfer_cutlass`, `flashinfer_cutedsl`, or `torch` for quantized Generator DiT linear layers |
| `DIFFUSION_ATTENTION_BACKEND` | Compute capability 8.0 through 9.x without sequence parallelism: `SAGE_ATTN`; otherwise automatic | Override the Generator diffusion attention backend |
| `NIM_VAE_FAST_PATH` | `channels_last` | Select the VAE decode path: `channels_last`, `lossless`, or `off` |
| `NIM_ENABLE_VAE_PARALLELISM` | `true` when one engine replica spans more than one GPU | Shard VAE decode spatially across the replica's GPUs |
| `NIM_VAE_SPATIAL_TEMPORAL_CHUNK_SIZE` | Automatic | Set a positive number of latent frames per sharded VAE decode chunk |
| `NIM_TRITON_LOG_VERBOSE` | `0` | Increase Generator backend logging during diagnosis |
| `NIM_MAX_SEQUENCE_LENGTH` | `5120` | Set the startup prompt-token sequence length |

Leave the quantized linear backend at its default unless the exact
hardware/precision pairing has been validated. An incompatible backend fails
during model load rather than silently falling back.

#### Diffusion cache (SeaCache)

SeaCache reuses selected transformer results across denoising steps to reduce
generation time. It is enabled by default for every Generator variant except
`edge-droid`. It does not change profile selection or checkpoint precision, and
the Reasoner does not use it. Set `NIM_CACHE_BACKEND=sea_cache` to enable it
for `edge-droid`, or `NIM_CACHE_BACKEND=none` to disable it for uncached
comparisons.

The SeaCache defaults are `sea_threshold=0.25`, `sea_residual_order=1`,
`sea_max_consecutive_cached=2`, and `sea_power_exp=3.0`. Override individual
fields with JSON. For example:

```bash
-e NIM_CACHE_CONFIG='{"sea_threshold": 0.20, "sea_max_consecutive_cached": 1}'
```

`sea_max_consecutive_cached=0` means no limit; it does not disable caching.
Startup logs the effective backend and settings. The defaults are not validated
as quality-equivalent to uncached output for every model and task. Compare
representative outputs with `NIM_CACHE_BACKEND=none` before relying on cached
output for a quality-sensitive workload.

#### Attention and VAE decode

On GPUs with compute capability 8.0 through 9.x (Ampere, Ada, and Hopper), the
Generator uses SageAttention 2.2 by default unless the profile uses sequence
parallelism. Blackwell GPUs and sequence-parallel profiles use automatic
backend selection. An explicit `DIFFUSION_ATTENTION_BACKEND` overrides that
default.

`NIM_VAE_FAST_PATH=channels_last` is the fastest decode path, but its output
can differ slightly from the reference decoder. `lossless` keeps the
reference output with the same sharding and chunking, and `off` uses the
reference decoder. These are startup settings, not request fields. Setting
`NIM_ENABLE_VAE_PARALLELISM=0` on a multi-GPU replica can increase per-GPU
memory. `NIM_VAE_SPATIAL_TEMPORAL_CHUNK_SIZE` affects only sharded decode;
larger chunks raise peak memory and can change rounding.

### Guardrails

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_TEXT_GUARDRAILS` | `true` | Enable input-prompt policy checks |
| `NIM_ENABLE_VIDEO_GUARDRAILS` | `true` | Enable output image/video face and visual guardrails |
| `NIM_ENABLE_SIGLIP_GUARDRAILS` | `true` | Enable the output-frame safety classifier |
| `NIM_QWEN3GUARD_MAX_INPUT_TOKENS` | `8192` | Set the text guardrail's maximum input tokens; minimum 512 |

The selected profile owns guardrail residency. To choose profiles whose
guardrails sleep on the CPU during diffusion, use the
`NIM_TEXT_GUARD_OFFLOAD` and `NIM_VIDEO_GUARD_OFFLOAD` profile selectors under
[Advanced profile controls](#advanced-profile-controls). Earlier images used
`NIM_OFFLOAD_TEXT_GUARDRAIL` and `NIM_OFFLOAD_VIDEO_GUARDRAIL`; this image
ignores them.

### Transfer diagnostic override

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ALLOW_UNSAFE_TRANSFER` | `false` | Bypass Transfer's VRAM admission check for an approved diagnostic; the request can fail with OOM |

### Prompt upsampling

Prompt upsampling is optional and applies only to Generator T2I, T2V, and I2V.
It sends the request to an operator-provided OpenAI-compatible endpoint.

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_PROMPT_UPSAMPLING` | `false` | Enable prompt upsampling |
| `NIM_PROMPT_UPSAMPLING_ENDPOINT_URL` | Empty | Set the OpenAI-compatible endpoint base or Chat Completions route |
| `NIM_PROMPT_UPSAMPLING_MODEL` | Empty | Set the external model name |
| `NIM_PROMPT_UPSAMPLING_API_KEY` | Empty | Set the external Bearer credential |
| `NIM_PROMPT_UPSAMPLING_TEMPLATE_STYLE` | `external_api` | Select `external_api` or `reasoner` templates |
| `NIM_PROMPT_UPSAMPLING_TIMEOUT_S` | `120` | Set the external call timeout in seconds |
| `NIM_PROMPT_UPSAMPLING_MAX_TOKENS` | `8192` | Set the requested output-token limit |
| `NIM_PROMPT_UPSAMPLING_TEMPERATURE` | Omitted | Optionally set external sampling temperature |
| `NIM_PROMPT_UPSAMPLING_TOP_P` | Omitted | Optionally set external nucleus sampling |
| `NIM_PROMPT_UPSAMPLING_TOP_K` | Omitted | Optionally set top-k when the provider accepts it |
| `NIM_PROMPT_UPSAMPLING_EXTRA_BODY` | Empty object | Merge additional JSON into the external request |

Endpoint, model, and key are required when the feature is enabled. A request-time
external failure logs a warning and continues with the original prompt. Keep
the external key separate from `NGC_API_KEY`.

## Reasoner configuration

Most of these are advanced workload controls. The unified-memory utilization
setting is required for Reasoner on DGX Spark and Jetson AGX Thor. Change other
controls one at a time and validate memory, latency, quality, and correctness.

### Optimization presets

| Name | Default | Use |
| --- | --- | --- |
| `NIM_OPTIMIZATION_PRESET` | Unset | Reasoner only: apply the `TTFT` or `TPOT` starting configuration; names are case-insensitive |

The presets are experimental starting points for time-to-first-token (`TTFT`) or
time-per-output-token (`TPOT`) latency. They are not performance-qualified
across hardware and workloads. Generator startup rejects the variable.

| Preset | `NIM_USE_DFLASH` | `NIM_MAX_NUM_BATCHED_TOKENS` | `NIM_DISABLE_CHUNKED_MM_INPUT` |
| --- | --- | --- | --- |
| `TTFT` | `0` | `32768` with at least 70 GiB of GPU memory; otherwise `16384` | Unchanged (`true`) |
| `TPOT` | `1` | `4096` | `0` |

On profiles with tensor parallelism greater than 1, both presets also set
`NIM_MM_ENCODER_TP_MODE=data`. This replicates the vision encoder and splits
media across the TP ranks. An explicitly set variable takes precedence over the
preset value. `TPOT` enables DFlash, so set `NIM_USE_DFLASH=0` with it on Edge,
which has no DFlash draft. Measure latency, throughput, and memory on the
exact image before adopting a preset.

### Speculative decoding

| Name | Default | Use |
| --- | --- | --- |
| `NIM_USE_DFLASH` | `true` for Nano/Super Reasoner; `false` for Edge | Use the bundled Nano or Super DFlash draft; set `false` to run the target model without speculative decoding. Edge rejects `true` |
| `NIM_DFLASH_MODEL_PATH` | Empty | Use an independent absolute local DFlash directory containing `config.json` and `model.safetensors` |
| `NIM_DFLASH_BF16_KV_CACHE` | `false` on Hopper; `true` otherwise | Override whether DFlash uses a BF16 KV cache instead of the profile-derived cache dtype |
| `NIM_DFLASH_CONFIG` | Empty object | Add or override advanced vLLM DFlash speculative-configuration fields as JSON |

DFlash does not change the Reasoner request API. Nano and Super Reasoner
profiles include variant-specific drafts and enable them by default; Generator
rejects an explicit DFlash enable. Edge does not support DFlash in this version;
set `NIM_USE_DFLASH=0` for Edge. Confirm that the selected image contains the
draft artifact. `NIM_DFLASH_MODEL_PATH` accepts only an absolute local path,
not an `hf://` source. `NIM_DFLASH_CONFIG` requires DFlash to remain enabled and
cannot set the reserved `method` or `model` keys.

The BF16 KV-cache default is disabled on Hopper and enabled on every other
architecture. BF16 can improve DFlash compatibility and acceptance length, but
it uses more KV-cache memory. Treat the independent draft path, KV-cache
override, and `NIM_DFLASH_CONFIG` as advanced controls; measure memory,
correctness, latency, and throughput on the exact image before production use.

### Context and scheduling

| Name | Default | Use |
| --- | --- | --- |
| `NIM_MAX_MODEL_LEN` | `-1` (auto) | Let the runtime choose a context length bounded by the model |
| `NIM_MAX_NUM_BATCHED_TOKENS` | `16384` effective with the default unchunked multimodal input; otherwise `8192` | Set the scheduler token budget; unchunked multimodal input enforces a minimum of `16384` |
| `NIM_MAX_NUM_SEQS` | `256` | Set maximum scheduled sequences |
| `NIM_GPU_MEMORY_UTILIZATION` | Discrete GPU: `0.93`, capped to leave `NIM_GPU_MEMORY_HEADROOM_GIB` free; unified memory: `0.70` | Set the Reasoner GPU-memory target in `(0,1]`; startup warns when the target exceeds current free memory but does not reduce it |

On DGX Spark and Jetson AGX Thor, this target is a share of the unified
host/device memory pool. The NIM defaults to `0.70` on these systems, so the
variable is not required. Set a lower value only when host measurements show
that media processing or other host workloads need more memory. See
[Set the Reasoner memory share on unified-memory
systems](deployment.md#set-the-reasoner-memory-share-on-unified-memory-systems).

### Caching and multimodal processing

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_KV_CACHE_REUSE` | `true` | Enable prefix/KV-cache reuse |
| `NIM_ENABLE_CHUNKED_PREFILL` | `true` | Enable chunked prefill |
| `NIM_DISABLE_CHUNKED_MM_INPUT` | `true` | Require one complete multimodal item to fit in a scheduler iteration; this raises `NIM_MAX_NUM_BATCHED_TOKENS` to at least `16384` |
| `NIM_DISABLE_MM_PREPROCESSOR_CACHE` | `false` | Disable the multimodal preprocessor cache |
| `NIM_MAX_IMAGES_PER_PROMPT` | Unset | Optionally set a nonnegative image limit; when unset, do not override the runtime limit |
| `NIM_MAX_VIDEOS_PER_PROMPT` | Unset | Optionally set a nonnegative video limit; when unset, do not override the runtime limit |
| `NIM_MEDIA_IO_KWARGS` | Video backend `pynvvc` | Replace the complete operator-level media I/O object with a JSON object |
| `NIM_MM_PROCESSOR_KWARGS` | Unset | Set operator-level multimodal processor options as a JSON object |
| `NIM_VIDEO_PRUNING_RATE` | `0.6` | Set video-token pruning from 0 through 1; `0`, `none`, `null`, or an empty value disable pruning |
| `NIM_VIDEO_PRUNING_METHOD` | `vidcom2` | Select `vidcom2` or `evs` when pruning is enabled |

Prefer request-level `media_io_kwargs` for one workload rather than changing the
operator-wide media object. Leave `NIM_MM_PROCESSOR_KWARGS` unset unless the
selected runtime has been validated with the complete processor-options object.
Both operator-level options require JSON objects. Disabling multimodal-input
chunking is the default and can increase peak GPU memory use because one
complete item must fit in a scheduler iteration; set
`NIM_DISABLE_CHUNKED_MM_INPUT=0` only after validating memory, latency, and
throughput. Leave Reasoner GPU-memory headroom at its default unless system
measurements establish another safe reserve; reducing it increases startup and
runtime OOM risk.

### Streaming video sessions

Set `NIM_MODEL_TYPE=reasoner` and `NIM_ENABLE_STREAMING=true` to enable the
WebSocket and REST session APIs in standalone NIM. In [Dynamo mode](#dynamo-streaming),
only REST sessions are supported. Changing these settings requires restarting
the service with the new environment. Ordinary Chat Completions token streaming
uses the request field `stream=true` and does not require this enable flag.

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_STREAMING` | `false` | Enable stateful video-frame sessions and apply the required engine settings |
| `NIM_STREAMING_MAX_SESSIONS` | `2` | Maximum concurrent sessions shared by WebSocket and REST; integer, minimum 1 |
| `NIM_STREAMING_MAX_VIDEO_SEGMENTS` | `8` | Default retained frame window per session; integer, minimum 1 |
| `NIM_STREAMING_IDLE_TIMEOUT_S` | `900` | Idle timeout for WebSocket sessions, in seconds; must be greater than 0 |
| `NIM_STREAMING_REST_IDLE_TIMEOUT_S` | `120` | Idle timeout for REST sessions, in seconds; must be greater than 0 |

Streaming forces the multimodal processor cache off and disables asynchronous
scheduling. It raises the engine's image budget to at least
`2 × NIM_STREAMING_MAX_SESSIONS × NIM_STREAMING_MAX_VIDEO_SEGMENTS`, or 32 image
slots with the defaults. This floor can override a lower
`NIM_MAX_IMAGES_PER_PROMPT`. Increasing either streaming limit increases
encoder-cache memory requirements; reduce the limits if startup lacks memory
and validate representative requests before increasing concurrency. These
engine changes also affect ordinary requests on the same container.

Accepted frames refresh session activity; a frame in flight is not reaped.
WebSocket disconnects release the session, while REST clients should explicitly
delete it rather than rely on the shorter idle timeout. Increase the relevant
timeout for feeds with long pauses. Per-session `retention` and `sampling`
belong in the creation request; retention overrides remain subject to shared
capacity and model-context admission checks.

#### Advanced streaming controls

Leave these at their defaults unless diagnosing or tuning a measured workload:

| Name | Default | Use |
| --- | --- | --- |
| `NIM_STREAMING_SPEC_METHOD` | `dflash` when DFlash is enabled; `none` otherwise | Select `dflash`, GPU n-gram speculation (`ngram`), or `none`. Set `none` for Edge |
| `NIM_STREAMING_SPEC_NUM_TOKENS` | `3` | Number of proposed tokens for the n-gram replacement; values below 1 become 1, and non-integer values fall back to 3 |
| `NIM_STREAMING_FORCE_V1_RUNNER` | Unset | Set `1` to force the vLLM V1 model runner; otherwise leave runner selection to vLLM |

The speculation controls apply when streaming is enabled. An explicit
`NIM_STREAMING_SPEC_METHOD` takes precedence over `NIM_USE_DFLASH`. With the
method unset, `NIM_USE_DFLASH=0` disables speculation in updated images; set
`NIM_STREAMING_SPEC_METHOD=none` explicitly for Edge to also cover older images
that default streaming to DFlash. DFlash is available only for Nano and Super.
Both V1 and V2 model runners support the
streaming path, so pinning V1 is not required for normal operation.

Inspect `GET /v1/streaming/config` for the served model and default retention
and sampling policy. Standalone NIM and the Dynamo streaming adapter expose
this route. See [Streaming](streaming.md) for session requests and error handling,
and [Dynamo REST streaming](dynamo/dynamo_with_streaming.md#stream-video-frames-over-rest)
for the client's shared-frontend checks, which require no worker URLs or metadata.

### API behavior

| Name | Default | Use |
| --- | --- | --- |
| `NIM_GUIDED_DECODING_BACKEND` | `xgrammar` | Select the structured-output backend |
| `NIM_DISABLE_LOG_REQUESTS` | `true` | Keep Reasoner request bodies out of logs |
| `NIM_DISABLE_RESPONSES_ROUTE` | `false` | Remove Responses routes when set to `true` |

## Dynamo configuration

### Run NIM as a Dynamo worker

Use the available NIM image `<nim-image-build>` and start
`/opt/nim/start_server.sh` with `NIM_MODEL_TYPE=reasoner` and
`NIM_DYNAMO_WORKER=true`. Generator profiles do not support Dynamo worker mode.
The NIM selects and loads the model, launches a Dynamo vLLM worker and a private
frontend, and exposes the NIM HTTP API. Keep the usual model-selection,
cache, and authentication settings from [Essential configuration](#essential-configuration).
The shared frontend is a separate service; it does not need worker enablement
or a GPU.

| Name | Default | Use |
| --- | --- | --- |
| `NIM_DYNAMO_WORKER` | `false` | Launch the Reasoner as a Dynamo worker instead of the standalone Reasoner runtime |
| `NIM_DISCOVERY_BACKEND` | `file` | Set worker and private-frontend discovery; use `kubernetes` for the supplied Kubernetes deployments. Also accepts the legacy JSON argument object shown below |
| `DYN_NAMESPACE_PREFIX` | From the `nvidia.com/dynamo-namespace` pod label in the supplied YAML | Let the private frontend discover operator-suffixed worker namespaces; takes precedence over the exact `DYN_NAMESPACE` filter |
| `NIM_DYNAMO_ASYNC_SCHEDULING` | `true` | Control worker asynchronous scheduling; streaming forces this off |
| `NIM_DYNAMO_ENABLE_MULTIMODAL` | `true` | Enable Dynamo multimodal handling for image/video requests |
| `NIM_DYNAMO_ARGS` | Empty object | Advanced Dynamo worker CLI overrides as a JSON object; use underscore keys such as `discovery_backend`, `namespace`, and `endpoint` |
| `NIM_DYNAMO_PROXY_PORT` | `8080` | Private frontend's loopback HTTP port; integer from 1 through 65535, different from `NIM_HTTP_API_PORT` (default `8000`) |
| `NIM_DYNAMO_PROXY_REQUEST_TIMEOUT_SECS` | `120` in the NIM image | NIM proxy request timeout in seconds; persistent session operations use the separate streaming timeout below |
| `NIM_DYNAMO_PROXY_FORWARDED_REQUEST_HEADERS_LIST` | Empty additions | Comma- or whitespace-separated extra request headers to forward through the NIM proxy; built-in tracing, affinity, worker-routing, and tenant headers remain enabled |

Argument precedence is defaults, then `NIM_DISCOVERY_BACKEND` (including its
legacy JSON fields), then the individual scheduling/multimodal variables, then
`NIM_DYNAMO_ARGS`. Streaming engine constraints are applied last and cannot be
overridden by these arguments. The supplied YAML files use the legacy JSON form:

```text
# Regular Dynamo
NIM_DISCOVERY_BACKEND={"async_scheduling":true,"discovery_backend":"kubernetes","enable_multimodal":true}
# Dynamo with REST streaming
NIM_DISCOVERY_BACKEND={"async_scheduling":false,"discovery_backend":"kubernetes","enable_multimodal":true}
```

### Dynamo streaming

Set `NIM_ENABLE_STREAMING=1` alongside `NIM_DYNAMO_WORKER=true` on workers.
Use the same `<nim-image-build>` image for the shared frontend, running
`/opt/nim/.venv/bin/python -m streaming.dynamo.frontend` from `/opt/nim`.
The stock `python3 -m dynamo.frontend` command does not serve video-session routes.
The NIM worker selects its private streaming frontend automatically.

| Name | Default | Use |
| --- | --- | --- |
| `NIM_DYNAMO_STREAMING_ENDPOINT` | Derived `<namespace>.<component>.streaming`; standalone adapter fallback `dynamo.backend.streaming` | Set the same explicit RPC endpoint on the shared frontend and workers. The YAML uses `cosmos3-reasoner.backend.streaming`; use a distinct endpoint for each model pool |
| `NIM_DYNAMO_STREAMING_REQUEST_TIMEOUT_S` | `180` | Bound each REST session create, frame, or delete operation, including upload and inference, in seconds; positive finite number. Set on the shared frontend and workers |
| `DYN_HTTP_HOST`, `DYN_HTTP_PORT` | `0.0.0.0`, `8000` | Streaming adapter listener defaults; explicit `--http-host` and `--http-port` arguments take precedence |
| `DYN_TLS_CERT_PATH`, `DYN_TLS_KEY_PATH` | Unset | Optional certificate and key paths for TLS on the streaming adapter; set both together, or use `--tls-cert-path` and `--tls-key-path` |

The adapter's `--streaming-endpoint` flag overrides its environment endpoint;
keep it aligned with the workers. Use aggregated workers with a local encoder;
disaggregated prefill/decode and remote encoders are unsupported. The
[streaming capacity, retention, idle-timeout, and engine controls](#streaming-video-sessions)
also apply. Streaming forces synchronous scheduling and disables the multimodal
processor cache. The request timeout above is separate from
`NIM_STREAMING_REST_IDLE_TIMEOUT_S`, which expires inactive sessions. REST session
ownership does not depend on the Chat Completions affinity TTL.

**WebSocket (`/v1/streaming/ws`) is unsupported in Dynamo mode**, on both shared
frontend and worker NIM endpoints. The streaming adapter and enabled worker NIM
proxy expose `GET /v1/streaming/config` and the REST session routes. The stock
`dynamo.frontend` launcher does not attach these streaming routes.
See [Dynamo with streaming](dynamo/dynamo_with_streaming.md) for the manifest
and REST request examples.

## Secret handling

Keep `NGC_API_KEY`, `HF_TOKEN`, and `NIM_PROMPT_UPSAMPLING_API_KEY` separate.
Inject them through the deployment environment. Do not put them in source
control, model-source URIs, image layers, requests, notebooks, or logs.
