# Architecture Map — turbo-fieldfare

Single-domain inference runtime. One core domain (model inference), supporting
domains (model install, front ends). No network serving exists today.

## Domains

| id | name | classification | maturity | owner | confidence |
|----|------|----------------|----------|-------|------------|
| inference | Model inference runtime (Metal kernels, forward pass, sampling) | core | mature | repo | [VERIFIED] Sources/TurboFieldfare |
| install | Model download/repack/verify into .gturbo | supporting | mature | repo | [VERIFIED] Sources/TurboFieldfareRepack |
| frontend | User-facing entry points (CLI, Mac app, decode IPC) | supporting | mature | repo | [VERIFIED] Sources/TurboFieldfareCLI, TurboFieldfareApp |
| serving | Network API serving | generic | absent | — | [VERIFIED] no HTTP server anywhere in Sources/ |

## Capabilities

| id | domain | name | implemented-by | consumed-by |
|----|--------|------|----------------|-------------|
| cap-load-model | inference | Load .gturbo model + tokenizer + Metal context | Model.load, GFTokenizer.load, MetalContext | CLI Run.swift, Mac app |
| cap-generate | inference | Streaming prefill+decode loop with progress callbacks | runRawCompletion (RawCompletion.swift) | CLI Run.swift, Mac app decode service |
| cap-chat-format | inference | Gemma chat template rendering | GFTokenizer.applyChatTemplate | CLI Run.swift, Mac app |
| cap-sample | inference | temperature/top-k/top-p/repetition/seed sampling | GenerationConfig + Sampler | runRawCompletion |
| cap-install | install | Streaming HF repack into .gturbo + verify | RemoteStreamingRepacker | Mac app, TurboFieldfareRepack CLI |
| cap-decode-ipc | frontend | Unix-socket decode protocol app↔service | TurboFieldfareDecodeProtocol | Mac app |

## Entities

| id | owning-domain | lifecycle | projections | aggregate notes |
|----|---------------|-----------|-------------|-----------------|
| .gturbo model directory | install | installed → verified → loaded | manifest.json, receipt | validated by hashes [VERIFIED] |
| Generation | inference | per-request: prefill → decode → stop | RawDecodeResult, RawDecodeProgress | single-in-flight per scratch [VERIFIED] |
| Chat message | frontend | role+content rows → templated prompt | GFTokenizer.Message | roles: user/model/system [VERIFIED] |

## Use cases

| name | feature/source | capabilities | entities | UX surfaces |
|------|----------------|--------------|----------|-------------|
| Chat via CLI | TurboFieldfareCLI --messages-file | cap-chat-format, cap-generate | Chat message, Generation | terminal |
| Raw completion via CLI | TurboFieldfareCLI --prompt | cap-generate | Generation | terminal |
| Chat via Mac app | TurboFieldfareMac | cap-install, cap-load-model, cap-generate | all | SwiftUI app |
| Install model | TurboFieldfareRepack | cap-install | .gturbo | terminal/app |

## Components

| path | domain | feature | container | capabilities/entities |
|------|--------|---------|-----------|----------------------|
| Sources/TurboFieldfare | inference | runtime library | SPM lib target | cap-load-model, cap-generate, cap-chat-format, cap-sample |
| Sources/TurboFieldfareCLI | frontend | CLI | SPM target+exec | chat/completion use cases |
| Sources/TurboFieldfareApp | frontend | Mac app + decode service | SPM targets | app use cases |
| Sources/TurboFieldfareRepack | install | installer | SPM target+exec | cap-install |
| Sources/TurboFieldfareDecodeProtocol | frontend | IPC protocol | SPM target | cap-decode-ipc |

## UX surfaces

| screen/flow | bound entities | triggered capabilities | nav in/out |
|-------------|----------------|------------------------|------------|
| CLI invocation | Generation | cap-generate | stdout/stderr |
| Mac app window | .gturbo, Generation | cap-install, cap-generate | — |

## Edges

| upstream domain | downstream consumer | capability | contract | transport | archetype |
|-----------------|---------------------|------------|----------|-----------|-----------|
| frontend | inference | cap-generate | runRawCompletion Swift API | in-process | library call |
| frontend | install | cap-install | RemoteStreamingRepacker Swift API | in-process | library call |
| Mac app | decode service | cap-decode-ipc | DecodeProtocol Codable structs | Unix socket | IPC |

## Unknowns

| gap | why it matters | how to verify |
|-----|----------------|---------------|
| Concurrency contract of RealForwardRunner beyond single-in-flight scratch | Server must serialize or isolate generations | RawCompletion.swift docstring: "single-in-flight guard upstream is the contract" — no upstream guard exists in library [VERIFIED doc, INFERRED consequence] |
| HTTP framework tolerance (dependency policy) | Package.swift has exactly one external dep (swift-transformers) | Decide stage |
| Cancellation propagation to in-flight generation | Client disconnect must stop decode (expensive on 8GB) | runRawCompletion checks Task.checkCancellation [VERIFIED] |

## Recommended S5D entry points

- feat.api-server (new serving domain component) — decision-tier: HTTP framework + lifecycle tradeoffs.
