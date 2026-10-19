<!-- SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: OpenMDW-1.1 -->

# Cosmos3 Reasoner with Dynamo

See [Dynamo configuration](../configuration.md#dynamo-configuration) for worker
enablement, discovery, proxy settings, and
[streaming controls](../configuration.md#dynamo-streaming).

Choose a deployment:

| Guide | Manifest | Frontend and behavior |
| --- | --- | --- |
| [Regular Dynamo](dynamo_deployment.md) | [reasoner.yaml](reasoner.yaml) | Pinned Dynamo frontend running `dynamo.frontend`; Chat Completions, no persistent video sessions. |
| [Dynamo with streaming](dynamo_with_streaming.md) | [reasoner_with_streaming.yaml](reasoner_with_streaming.yaml) | Same NIM image for frontend and workers; frontend runs `streaming.dynamo.frontend` for Chat Completions plus REST video sessions. |

Both use [model-cache.yaml](model-cache.yaml), the
[model-download Job](model-download.yaml), and a Dynamo operator serving the
`nvidia.com/v1alpha1` `spec.services` schema. Both use the same NIM image for
the download Job and workers. The regular frontend uses the pinned Dynamo
image; the streaming frontend uses the NIM image. The regular manifest follows
the supplied working deployment; streaming requires separate session validation.
Use the available Cosmos3 NIM image address for each `<nim-image-build>`
placeholder in your chosen variant and the download Job.
Replace `<angle-bracket>` placeholders before applying. These are alternative
manifests for the same `cosmos3-reasoner` deployment; apply only one. Switching
variants replaces pods and loses active streaming sessions.

Here, **streaming** means persistent video sessions, not token streaming in a
Chat Completions response. The Dynamo video-session path uses REST only.
