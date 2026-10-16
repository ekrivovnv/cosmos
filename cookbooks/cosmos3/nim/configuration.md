<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Configure the Cosmos3 NIM

This page lists launch-time environment variables. Start with the
model-selection variables and keep automatic defaults unless the workload
requires an override. See [Deployment](deployment.md) for complete Docker
commands.

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
| `NIM_MODEL_VARIANT` | `nano` preference | Select `nano` or `super` for either runtime; Generator also accepts `nano-droid`, `super-t2i`, `super-t2i-4step`, `super-i2v`, and `super-i2v-4step` |
| `NIM_PRECISION` | Generator: FP8 preference; Reasoner: GPU-derived preference | Optionally pin a precision available in the image; see [Precision selection](deployment.md#precision-selection) |
| `NIM_PERF_PROFILE` | `latency` | Generator only: choose `latency` or `throughput` |

Reasoner rejects `NIM_PERF_PROFILE`, including `profile=` in
`NIM_TAGS_SELECTOR`, with a startup error. Nano-DROID has BF16 profiles only.
The NIM chooses the best compatible profile for these settings and the visible
GPUs; a normal deployment does not need a profile ID.

### GPU selection

The GPUs exposed by Docker `--gpus` are the visible GPUs that automatic
selection considers. List host GPUs with their indexes and UUIDs:

```bash
nvidia-smi -L
```

Expose specific GPUs instead of `--gpus all`, for example
`--gpus '"device=0,1"'` or `--gpus '"device=GPU-<uuid>"'`. Expose a GPU count
listed in the [Generator](support-matrix.md#generator-configurations) or
[Reasoner](support-matrix.md#reasoner-configurations) configurations, all of a
single model ([mixed GPUs are not
supported](support-matrix.md#gpu-architecture-and-topology)); extra GPUs sit
idle.

Do not pass `-e CUDA_VISIBLE_DEVICES`: the NIM sets it for its worker
processes, so a passed value can place work outside the intended GPUs.

### Bring your own checkpoint

| Name | Default | Use |
| --- | --- | --- |
| `NIM_MODEL_PATH` | Empty | Generator: absolute local directory. Reasoner: absolute local directory or `hf://owner/repository[:revision]` |
| `NIM_DISABLE_MODEL_DOWNLOAD` | `false` | Disable profile download for a completely local Reasoner override; incompatible with Reasoner `hf://` and rejected for Generator |
| `HF_TOKEN` | Empty | Authenticate to a private Reasoner Hugging Face repository |

To override the DFlash draft, see `NIM_DFLASH_MODEL_PATH` under [Speculative
decoding](#speculative-decoding). See [Bring your own
checkpoint](bring-your-own-checkpoint.md) for layouts, mounts, and
compatibility checks.

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
| `NIM_TAGS_SELECTOR` | Empty | Filter by comma-separated exact manifest tags as `key=value` pairs; see the valid keys below |

`NIM_TAGS_SELECTOR` accepts only these keys; any other key is an error. Values
are lowercased.

| Key | Meaning | Runtime |
| --- | --- | --- |
| `model_type`, `model_variant`, `precision` | Runtime, model, and precision | Both |
| `profile` | `latency` or `throughput` | Generator |
| `offload`, `text_guard_offload`, `video_guard_offload` | Offload modes | Generator |
| `n_gpus` | GPUs used by one NIM; the product of `nim_dp`, `nim_gp`, `nim_up`, and `nim_tp` | Both |
| `nim_dp` | Data-parallel model replicas; throughput profiles use `n_gpus` | Generator |
| `nim_gp` | CFG (guidance) parallel factor; latency profiles on 2 or more GPUs use `2` | Generator |
| `nim_up` | Ulysses sequence-parallel factor; latency profiles use `n_gpus / 2` | Generator |
| `nim_tp` | Tensor-parallel factor; Super BF16 also has a 2-GPU `nim_tp=2` layout | Reasoner |

`gp`, `cp`, and `dp` are accepted as aliases for `nim_gp`, `nim_up`, and
`nim_dp`.

A shorthand variable that conflicts with `NIM_TAGS_SELECTOR` is a startup
error; use the `model_variant` tag to filter by model. Exact tags and profile
IDs are tied to an image release. For the offload, memory-reserve, and
profile-pin variables, see [Advanced settings](#advanced-settings).

## Generator configuration

### Input and output

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ALLOW_URL_INPUT` | `true` | Allow HTTP(S) image, video, and Transfer inputs; set `false` to require encoded input |
| `NIM_TRITON_REQUEST_TIMEOUT` | 30 minutes (`1800000000` microseconds) | Set the queue-plus-execution timeout in microseconds |

Client HTTP timeouts do not change this backend ceiling; see [Long-running
requests](operations.md#long-running-requests).

### Startup and execution

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_WARMUP` | `false` | Run synthetic inference before readiness at the `256_1_1`, `480_16_9`, and `720_16_9` resolutions; Nano-DROID uses only `480_16_9`, and 4-step variants use their fixed step count |

### Guardrails

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_TEXT_GUARDRAILS` | `true` | Enable input-prompt policy checks |
| `NIM_ENABLE_VIDEO_GUARDRAILS` | `true` | Enable output image/video face and visual guardrails |
| `NIM_ENABLE_SIGLIP_GUARDRAILS` | `true` | Enable the output-frame safety classifier |

See [Guardrails](operations.md#guardrails) for what each layer does.

### Prompt upsampling

Prompt upsampling is optional and applies only to Generator T2I, T2V, and I2V.
It sends the request to an operator-provided OpenAI-compatible endpoint.

| Name | Default | Use |
| --- | --- | --- |
| `NIM_ENABLE_PROMPT_UPSAMPLING` | `false` | Enable prompt upsampling |
| `NIM_PROMPT_UPSAMPLING_ENDPOINT_URL` | Empty | Set the OpenAI-compatible endpoint base or Chat Completions route |
| `NIM_PROMPT_UPSAMPLING_MODEL` | Empty | Set the external model name |
| `NIM_PROMPT_UPSAMPLING_API_KEY` | Empty | Set the external Bearer credential |
| `NIM_PROMPT_UPSAMPLING_TIMEOUT_S` | `120` | Set the external call timeout in seconds |
| `NIM_PROMPT_UPSAMPLING_TEMPLATE_STYLE`, `NIM_PROMPT_UPSAMPLING_MAX_TOKENS`, `NIM_PROMPT_UPSAMPLING_TEMPERATURE`, `NIM_PROMPT_UPSAMPLING_TOP_P`, `NIM_PROMPT_UPSAMPLING_TOP_K`, `NIM_PROMPT_UPSAMPLING_EXTRA_BODY` | `external_api`, `8192`, omitted, omitted, omitted, empty object | Select `external_api` or `reasoner` templates; set the output-token limit, sampling temperature, nucleus sampling, and top-k (when the provider accepts it); merge additional JSON into the external request |

Endpoint, model, and key are required when the feature is enabled. A request-time
external failure logs a warning and continues with the original prompt.

## Reasoner configuration

On DGX Spark and Jetson AGX Thor, `NIM_GPU_MEMORY_UTILIZATION` is required;
change other controls one at a time and measure the effect.

### Speculative decoding

| Name | Default | Use |
| --- | --- | --- |
| `NIM_USE_DFLASH` | `true` for Reasoner | Use the bundled Nano or Super DFlash draft; set `false` to run the target model without speculative decoding |
| `NIM_DFLASH_MODEL_PATH` | Empty | Nano or Super Reasoner: use an independent absolute local DFlash directory containing `config.json` and `model.safetensors`; see [Local directory](bring-your-own-checkpoint.md#local-directory) for the mount |

DFlash is on by default for Reasoner and does not change the request API; set
`NIM_USE_DFLASH=0` to disable it (Generator rejects an explicit enable).
`NIM_DFLASH_MODEL_PATH` accepts only an absolute local path, not `hf://`, and
`NIM_DFLASH_CONFIG` requires DFlash enabled and cannot set the `method` or
`model` keys.

### Context and scheduling

| Name | Default | Use |
| --- | --- | --- |
| `NIM_MAX_MODEL_LEN` | `-1` (auto) | Let the runtime choose a context length, bounded by the checkpoint limit of 262,144 tokens |
| `NIM_GPU_MEMORY_UTILIZATION` | `0.93` | Set the Reasoner GPU-memory target in `(0,1]`; startup warns when it exceeds current free memory but does not reduce it. On DGX Spark/GB10 and Jetson AGX Thor, set `0.80` for image-only or `0.70` for video or mixed-media workloads from the first preflight on; see [Set the Reasoner memory share](deployment.md#set-the-reasoner-memory-share-on-unified-memory-systems) |

### Caching and multimodal processing

| Name | Default | Use |
| --- | --- | --- |
| `NIM_MAX_IMAGES_PER_PROMPT` | Unset | Optionally set a nonnegative image limit; when unset, do not override the runtime limit |
| `NIM_MAX_VIDEOS_PER_PROMPT` | Unset | Optionally set a nonnegative video limit; when unset, do not override the runtime limit |

Prefer request-level `media_io_kwargs` over the operator-wide
`NIM_MEDIA_IO_KWARGS`. Unchunked multimodal input, the default, can raise peak
GPU memory; set `NIM_DISABLE_CHUNKED_MM_INPUT=0` only after measuring memory,
latency, and throughput.

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
| `NIM_STREAMING_SPEC_METHOD` | `ngram` | Replace configured DFlash speculation with GPU n-gram speculation; `none` disables speculation. The `dflash` override is for diagnostics only and is unsupported for streaming |
| `NIM_STREAMING_SPEC_NUM_TOKENS` | `3` | Number of proposed tokens for the n-gram replacement; values below 1 become 1, and non-integer values fall back to 3 |
| `NIM_STREAMING_FORCE_V1_RUNNER` | Unset | Set `1` to force the vLLM V1 model runner; otherwise leave runner selection to vLLM |

The speculation controls apply only when streaming is enabled and the engine
has a DFlash speculative configuration. They do not independently enable
speculation when `NIM_USE_DFLASH=false`. N-gram replaces DFlash because DFlash
is currently not supported for streaming. Both V1 and V2 model runners support the
streaming path, so pinning V1 is not required for normal operation.

Inspect `GET /v1/streaming/config` for the served model and default retention
and sampling policy. Standalone NIM and the Dynamo streaming adapter expose
this route. See [Streaming](streaming.md) for session requests and error handling,
and [Dynamo REST streaming](dynamo/dynamo_with_streaming.md#stream-video-frames-over-rest)
for the client's shared-frontend checks, which require no worker URLs or metadata.

### API behavior

| Name | Default | Use |
| --- | --- | --- |
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

## Advanced settings

Leave these at their defaults unless you have measured a need.

| Name | Runtime | Default | Use |
| --- | --- | --- | --- |
| `NIM_OFFLOAD_MODE` | Generator | Automatic | Request `none`, `model`, or `layer` when the model provides it |
| `NIM_UNIFIED_MEMORY_HOST_RESERVE_GIB` | Both | `16` | Binary GiB reserved for the host before profile selection on integrated GPUs; no effect on discrete GPUs |
| `NIM_GPU_MEMORY_HEADROOM_GIB` | Reasoner | `2` | Free binary GiB reserved per discrete GPU for selection and sizing; lowering it raises OOM risk |
| `NIM_MODEL_PROFILE` | Both | Empty | Pin an exact profile ID from the current image |
| `NIM_VIDEO_SAVE_QUALITY` | Generator | `7` | VP9 output quality 1-9; affects encoding, not diffusion |
| `NIM_ENABLE_TORCH_COMPILE` | Generator | `true` | Enable the compilation path |
| `NIM_LINEAR_BACKEND` | Generator | Quantized: `cutlass`; BF16: unset | `auto`, `cutlass`, `flashinfer_cutlass`, `flashinfer_cutedsl`, or `torch` for quantized DiT linear layers; an incompatible backend fails at model load |
| `NIM_TRITON_LOG_VERBOSE` | Generator | `0` | Backend log verbosity for diagnosis |
| `NIM_MAX_SEQUENCE_LENGTH` | Generator | `5120` | Startup prompt-token sequence length |
| `NIM_OFFLOAD_TEXT_GUARDRAIL` | Generator | Profile policy | Override whether the text guard sleeps on CPU during diffusion |
| `NIM_OFFLOAD_VIDEO_GUARDRAIL` | Generator | Profile policy | Override whether output guardrails sleep during diffusion |
| `NIM_ALLOW_UNSAFE_TRANSFER` | Generator | `false` | Bypass Transfer's VRAM admission check for an approved diagnostic; can fail with OOM |
| `NIM_DFLASH_BF16_KV_CACHE` | Reasoner | `false` on Hopper; `true` otherwise | BF16 DFlash KV cache; can improve acceptance but uses more memory |
| `NIM_DFLASH_CONFIG` | Reasoner | Empty object | Extra vLLM DFlash speculative-config fields as JSON |
| `NIM_MAX_NUM_BATCHED_TOKENS` | Reasoner | `16384` with unchunked multimodal input; otherwise `8192` | Scheduler token budget; unchunked multimodal input enforces at least `16384` |
| `NIM_MAX_NUM_SEQS` | Reasoner | `256` | Maximum scheduled sequences |
| `NIM_ENABLE_KV_CACHE_REUSE` | Reasoner | `true` | Prefix/KV-cache reuse |
| `NIM_ENABLE_CHUNKED_PREFILL` | Reasoner | `true` | Chunked prefill |
| `NIM_DISABLE_CHUNKED_MM_INPUT` | Reasoner | `true` | Require one whole multimodal item per scheduler iteration |
| `NIM_DISABLE_MM_PREPROCESSOR_CACHE` | Reasoner | `false` | Disable the multimodal preprocessor cache |
| `NIM_MEDIA_IO_KWARGS` | Reasoner | Video backend `pynvvc` | Replace the whole operator-level media I/O JSON object |
| `NIM_MM_PROCESSOR_KWARGS` | Reasoner | Unset | Operator-level multimodal processor options as a JSON object |
| `NIM_VIDEO_PRUNING_RATE` | Reasoner | `0.6` | Video-token pruning from 0 through 1; `0`, `none`, `null`, or an empty value disable pruning |
| `NIM_VIDEO_PRUNING_METHOD` | Reasoner | `vidcom2` | `vidcom2` or `evs` when pruning is enabled |
| `NIM_GUIDED_DECODING_BACKEND` | Reasoner | `xgrammar` | Structured-output backend |

## Secret handling

Inject `NGC_API_KEY`, `HF_TOKEN`, and `NIM_PROMPT_UPSAMPLING_API_KEY` as
separate secrets through the deployment environment, never through source
control, model-source URIs, image layers, requests, notebooks, or logs.
