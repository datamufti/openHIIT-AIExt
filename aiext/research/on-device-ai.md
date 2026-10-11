# Running AI on the phone from Flutter

Research for [issue #2](https://github.com/datamufti/openHIIT-AIExt/issues/2). Checked 2026-10-11.

**Question.** Which on-device language models and runtimes can a Flutter app use to turn a typed description (for example "20 min tabata, burpees/squats/push-ups, 40 on 20 off") into a structured **Timer**: an ordered list of **Intervals** (work, rest and so on) with durations, exercise names and a repeat count? How does a cloud API compare?

**Short answer.**

- **Best on-device route: Gemma 4 E2B** (Apache 2.0, about 2.4–2.6 GB download) on **LiteRT-LM**.
- **From Flutter:** use the community `flutter_edge_ai` package (formerly `flutter_gemma`). Make the Timer the arguments of a single `create_timer` tool call, then validate it in Dart.
- **Strictest on-device output:** call LiteRT-LM's own Kotlin API through a small platform channel. That API offers full JSON-Schema constrained decoding (`ResponseFormat.json`) on Android.
- **Built-in models** (Gemini Nano on Pixel 9+, Apple Foundation Models) need no download. They reach only a subset of devices, and from Flutter today only through prompt-based parsing.
- **Easiest first step:** call the Gemini API through `firebase_ai`, using a response schema.

## 1. Gemma 4: what exists

The [Gemma 4 model card](https://ai.google.dev/gemma/docs/core/model_card_4) (last updated 2026-07-30) lists five sizes. Each comes as a pre-trained and an instruction-tuned variant:

| Model | Params | Context | Input | Target |
|---|---|---|---|---|
| E2B | 2.3B effective (5.1B with embeddings) | 128K | text, image, audio | phones, laptops |
| E4B | 4.5B effective (8B with embeddings) | 128K | text, image, audio | phones, laptops |
| 12B Unified | 11.95B | 256K | text, image, audio | GPUs, workstations |
| 26B A4B (MoE) | 25.2B total, 3.8B active | 256K | text, image | GPUs, workstations |
| 31B | 30.7B | 256K | text, image | GPUs, workstations |

- **Licence:** Apache 2.0 ([model card](https://ai.google.dev/gemma/docs/core/model_card_4); also on the [LiteRT-LM Gemma 4 page](https://developers.google.com/edge/litert-lm/models/gemma-4)).
- **Tool use:** the model card claims native function calling and a `system` role. It does not mention JSON-schema output as a model feature.
- **On-device variants:**
  - Only E2B and E4B run in LiteRT-LM; 26B A4B and 31B are "planned" ([LiteRT-LM Gemma 4](https://developers.google.com/edge/litert-lm/models/gemma-4)).
  - `flutter_edge_ai` reports that the 12B `.litertlm` is GPU-only ([source](https://github.com/DenisovAV/flutter_edge_ai/blob/main/packages/flutter_edge_ai_litertlm/lib/src/ffi/litert_lm_client.dart)).
- **Files:**
  - Download size is 2.58 GB for E2B and 3.65 GB for E4B ([LiteRT-LM](https://developers.google.com/edge/litert-lm/models/gemma-4)). The `flutter_edge_ai` catalogue lists 2.4 GB and 4.3 GB ([models doc](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/models.md)).
  - Hosted on Hugging Face at `litert-community/gemma-4-E2B-it-litert-lm` and `litert-community/gemma-4-E4B-it-litert-lm`.
  - The Hugging Face card says text-only weights for E2B can be as low as 0.8 GB, with the embeddings memory-mapped ([HF card](https://huggingface.co/litert-community/gemma-4-E2B-it-litert-lm)).
  - `flutter_edge_ai` says no Hugging Face token is needed for these files ([builtin-ai doc](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/builtin-ai.md)).
- **Measured speed and memory:** from [LiteRT-LM Gemma 4](https://developers.google.com/edge/litert-lm/models/gemma-4). Decode is in tokens per second; memory is peak CPU memory.

| Model, device | Backend | Decode (tok/s) | Time to first token | Memory |
|---|---|---|---|---|
| E2B, Galaxy S26 Ultra | GPU | 52 | 0.3 s | 676 MB |
| E2B, Galaxy S26 Ultra | CPU | 47 | 1.8 s | 1733 MB |
| E4B, Galaxy S26 Ultra | GPU | 22 | 0.8 s | 710 MB |
| E2B, iPhone 17 Pro | GPU | 56 | 0.3 s | 1450 MB |

  A Timer as JSON is roughly 150–300 tokens, which comes to a few seconds of generation on E2B. That figure is my estimate.

- **Pixel tiers:** Google publishes no Pixel benchmark or minimum RAM for Gemma 4. All official numbers are from Samsung, Apple and desktop hardware.
  - E2B uses about 0.7–1.7 GB of memory at runtime. My inference is that 8 GB phones (Pixel 8 and later) are comfortable and 6 GB phones are marginal. Measure this on a real Pixel before committing.
  - Android floor for LiteRT-backed models in `flutter_edge_ai`: `minSdk 30`, `arm64-v8a` only ([README](https://github.com/DenisovAV/flutter_edge_ai/blob/main/packages/flutter_edge_ai/README.md)).
- **Cloud access:** Gemma 4 is also served on the Gemini API, on the free tier only ([pricing](https://ai.google.dev/gemini-api/docs/pricing)). The same model can therefore be prompt-tuned in the cloud before it ships on the device.

## 2. Other on-device models

All of these run in LiteRT-LM and in `flutter_edge_ai` ([LiteRT-LM overview](https://developers.google.com/edge/litert-lm/overview); [flutter_edge_ai models](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/models.md)).

| Model | Download | Function calling in flutter_edge_ai | Notes |
|---|---|---|---|
| Gemma 4 E4B | 3.65–4.3 GB | yes | Better quality, about half E2B's decode speed |
| Gemma 3n E2B / E4B | 3.1 / 6.5 GB | yes (E4B) | Previous generation; superseded by Gemma 4 |
| Phi-4 Mini | 3.9 GB | yes | |
| Qwen3 0.6B | 586 MB (int8), 329 MB (int4) | yes | Apache 2.0. 69 tok/s on a Galaxy S25-class GPU with a 585 MB footprint ([HF card](https://huggingface.co/litert-community/Qwen3-0.6B)) |
| Qwen 2.5 0.5B / 1.5B | 0.5 / 1.6 GB | yes | |
| Gemma 3 1B | 0.5 GB | no | Hugging Face repo is gated (licence acceptance) ([README](https://github.com/DenisovAV/flutter_edge_ai/blob/main/packages/flutter_edge_ai/README.md)) |
| FunctionGemma 270M | 284 MB | yes | Built for function calling; likely needs fine-tuning for our schema (inference) |

My judgement, untested: models under 1B parameters will often get tabata arithmetic and exercise rotation wrong. If the download must stay small, a fine-tuned FunctionGemma 270M or Qwen3 0.6B is the realistic fallback. Gemma 4 E2B is the smallest general model likely to work from a prompt alone.

## 3. Runtimes

### LiteRT-LM (Google; the current runtime)

- **What it is:** Google's production runtime for on-device LLMs ([overview](https://developers.google.com/edge/litert-lm/overview)).
  - Apache 2.0; v0.18.0 released 2026-10-06 ([GitHub](https://github.com/google-ai-edge/LiteRT-LM)).
  - Platforms: Android, iOS, web, macOS, Windows, Linux and IoT.
  - Hardware: CPU everywhere, GPU on mobile and desktop, NPU on Android.
  - APIs: stable for Kotlin, C++ and Python; early preview for Swift and JS; Flutter is "community" and points to `flutter_gemma`.
- **Constrained decoding** ([docs](https://github.com/google-ai-edge/LiteRT-LM/blob/main/docs/api/cpp/constrained-decoding.md)). There are two modes, and a conversation uses one or the other:
  1. Tool-call constraint (`SetEnableConstrainedDecoding`). It guarantees valid function-call syntax.
     - The grammar is built from the tool declarations ([source](https://github.com/google-ai-edge/LiteRT-LM/blob/main/runtime/components/constrained_decoding/llg_fc_tool_calls.cc)).
     - It enforces the tool name and each top-level parameter's type and `enum`.
     - Nested arrays and objects fall back to generic JSON. A `intervals: [{...}]` parameter is therefore *not* checked item by item.
  2. LLGuidance (`LlGuidanceConfig`). It handles regex, **JSON Schema** and Lark grammars on each request.
     - In Kotlin this is `ConversationConfig(enableResponseFormat = true)` plus `sendMessage(..., responseFormat = ResponseFormat.json(schema))` ([ResponseFormat.kt](https://github.com/google-ai-edge/LiteRT-LM/blob/main/kotlin/java/com/google/ai/edge/litertlm/ResponseFormat.kt), [Config.kt](https://github.com/google-ai-edge/LiteRT-LM/blob/main/kotlin/java/com/google/ai/edge/litertlm/Config.kt), [Conversation.kt](https://github.com/google-ai-edge/LiteRT-LM/blob/main/kotlin/java/com/google/ai/edge/litertlm/Conversation.kt)).
     - Maven artifact: `com.google.ai.edge.litertlm:litertlm-android` ([Kotlin guide](https://github.com/google-ai-edge/LiteRT-LM/blob/main/docs/api/kotlin/getting_started.md)).
     - This is the strongest on-device guarantee found: output can only be JSON that matches the Timer schema.

### MediaPipe LLM Inference API

- Now in **maintenance-only mode**. Google recommends migrating to LiteRT-LM ([docs](https://developers.google.com/edge/mediapipe/solutions/genai/llm_inference)).
- Uses the `.task` format on Android, iOS and web. Do not start new work on it.

### flutter_edge_ai (formerly flutter_gemma), the main Flutter option

- **Package:** `flutter_edge_ai` 2.1.3, MIT, verified publisher sashadenisov.dev, 1.66k downloads ([pub.dev](https://pub.dev/packages/flutter_edge_ai)). `flutter_gemma` 1.11.4 is the final release under that name and is marked discontinued ([pub.dev](https://pub.dev/packages/flutter_gemma)).
- **Engines** are separate packages ([README](https://github.com/DenisovAV/flutter_edge_ai/blob/main/packages/flutter_edge_ai/README.md)):
  - LiteRT-LM via FFI: Android, iOS, macOS, Windows, Linux; web is a text-only preview.
  - MediaPipe: mobile and web.
  - ONNX Runtime: all six platforms.
  - Built-in OS models: Gemini Nano, Apple Foundation Models, Phi Silica, Chrome Prompt API.
- **Model delivery:** installs from the network, Hugging Face, app assets, bundled files or a local path, with progress reporting and retries. Installed models survive restarts (README).
- **Structured output:**
  - *Function calling.* For Gemma 4 and FunctionGemma on `.litertlm`, tool declarations go through LiteRT-LM's native tool path, which is the grammar-constrained mode 1 above. Other models get tools woven into the prompt and parsed back out ([function-calling doc](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/function-calling.md)).
  - *No Dart JSON-Schema option.* I found no Dart API for `ResponseFormat`/LLGuidance JSON-Schema decoding; a source search found no such option.
  - The package's own Genkit docs state that "on-device Gemma has no native schema-constrained decoder" in this package. They put the schema in the prompt and parse the result ([genkit doc](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/genkit.md)).
  - On web, constrained decoding is switched off because of an upstream bug ([LiteRT-LM#2434](https://github.com/google-ai-edge/LiteRT-LM/issues/2434)).
- **Hybrid routing:** `genkit_hybrid` routes between on-device and cloud: on-device first, then cloud when a check such as "parses as JSON" fails (genkit doc).

### Android AICore / Gemini Nano via ML Kit GenAI Prompt API

- **Status:** Prompt API is beta; its structured output is **alpha** ([ML Kit GenAI](https://developers.google.com/ml-kit/genai)). Artifact `com.google.mlkit:genai-prompt:1.0.0-beta4`, `minSdk 26`, input under 4,000 tokens ([get started](https://developers.google.com/ml-kit/genai/prompt/android/get-started)).
- **Devices for the Prompt API** ([ML Kit GenAI](https://developers.google.com/ml-kit/genai)):
  - nano-v3: Pixel 9 and 10.
  - nano-v4: Pixel 11.
  - Plus selected Samsung, OnePlus, Xiaomi and other models.
  - No Pixel 8 or older. Not available with an unlocked bootloader.
- **Limits:** foreground only, with per-app quotas including a battery quota. Different Nano versions may answer the same prompt differently ([ML Kit GenAI](https://developers.google.com/ml-kit/genai)).
- **Download:** AICore downloads and owns the model, so the app ships nothing (`checkStatus()` / `download()`).
- **Structured output:** Kotlin classes annotated with `@Generable`/`@Guide`, compiled by the KSP `genai-schema-compiler`. It supports nested classes, `List<T>`, int min/max and string enums. It is Kotlin-only and alpha, and the docs do not say whether decoding is constrained ([structured output](https://developers.google.com/ml-kit/genai/prompt/android/structured-output)).
- **From Flutter:** `flutter_edge_ai_builtin_ai` reaches Nano, but tool calling there is **prompt-based**. Using `@Generable` needs our own Kotlin platform channel ([builtin-ai doc](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/builtin-ai.md)).
- **Firebase AI Logic hybrid:**
  - On Android, Firebase AI Logic wraps the same model with `PREFER_ON_DEVICE` and cloud fallback. It is experimental, Kotlin/Java only, single-turn, has no tools, and is validated only for English and Korean ([hybrid Android](https://firebase.google.com/docs/ai-logic/hybrid/android/get-started)).
  - The Flutter `firebase_ai` 4.0.0 page does not mention on-device inference ([pub.dev](https://pub.dev/packages/firebase_ai)).

### Apple Foundation Models (iOS nice-to-have)

- Runs on iOS/macOS 26+ on an iPhone 15 Pro or later with Apple Intelligence enabled, according to `flutter_edge_ai`'s docs ([builtin-ai doc](https://github.com/DenisovAV/flutter_edge_ai/blob/main/website/content/docs/builtin-ai.md)).
- Apple's guided generation (`@Generable`) produces typed Swift values ([Apple docs](https://developer.apple.com/documentation/foundationmodels/generating-swift-data-structures-with-guided-generation)). I could not read the page body, so the decoding mechanism is not verified here.
- In `flutter_edge_ai` it is reachable only through an `@experimental` API; the stable path is prompt-based.

### llama.cpp wrappers (GGUF models, GBNF grammars)

- **`llama_cpp_dart`** 0.2.2: Android, iOS and desktop; you compile llama.cpp yourself; the page does not mention grammar ([pub.dev](https://pub.dev/packages/llama_cpp_dart)).
- **`fllamer`** 0.2.2: has experimental GBNF and JSON-Schema support, but 0 likes and 34 downloads, and it builds from source with CMake and the NDK ([pub.dev](https://pub.dev/packages/fllamer)).
- **Verdict:** grammar support is real in llama.cpp, but the Flutter wrappers are immature. Hold this route in reserve.

## 4. Cloud API (fallback, or the first development step)

- **Gemini API structured output:** pass a JSON Schema with `application/json`. It supports enums, min/max, `items`, `minItems`, nesting and `anyOf`. Output is "syntactically correct JSON", but the docs still say to validate values ([docs](https://ai.google.dev/gemini-api/docs/structured-output)).
- **From Flutter:** `firebase_ai` 4.0.0 covers Android, iOS, macOS and web ([pub.dev](https://pub.dev/packages/firebase_ai)). Its Dart SDK supports `responseSchema` with `Schema.object/array/enumString` ([Firebase docs](https://firebase.google.com/docs/ai-logic/generate-structured-output)).
  - Firebase App Check becomes **required** from 2026-11-02 (same page). Using Firebase means no API key is shipped in the app.
- **Cost** per 1M tokens, input/output ([pricing](https://ai.google.dev/gemini-api/docs/pricing)):
  - Gemini 2.5 Flash-Lite: $0.10 / $0.40.
  - Gemini 3.1 Flash-Lite: $0.25 / $1.50.
  - At about 500 tokens in and 300 out per Timer, that is roughly $0.0002 per request on 2.5 Flash-Lite (my arithmetic).
  - The free tier exists, but free-tier content is "used to improve our products". Paid-tier content is not.
- **Latency:** no official figure. It is a network round trip plus generation; measure it.
- **Privacy:** the workout description leaves the device. That is low sensitivity for this app, but it should be disclosed.
- **Platforms:** every platform with a network connection, including web and desktop.
- **Why start here:** best schema reliability and the least integration work. The prompt and schema built here carry over unchanged to Gemma 4 on the device. Gemma 4 is even on the Gemini API free tier.

## 5. Recommendation

| Option | Platforms | Hardware | Structured-output reliability | Effort |
|---|---|---|---|---|
| **Gemini API via `firebase_ai`** (step 1) | Android, iOS, web, macOS | any, online | High: server-side JSON Schema; still validate | Low |
| **Gemma 4 E2B + `flutter_edge_ai_litertlm`, tool-call mode** (main on-device) | Android, iOS, macOS, Windows, Linux (web preview, unconstrained) | arm64, Android 11+ (`minSdk 30`); about 0.7–1.7 GB memory; 2.4–2.6 GB download; Pixel 8+ likely (unverified) | Medium-high: tool name and top-level types constrained, nested items not. Validate and retry | Medium |
| **Gemma 4 E2B + LiteRT-LM Kotlin via platform channel, `ResponseFormat.json`** | Android only (iOS Swift API is early preview) | as above | High: full JSON-Schema constrained decoding (LLGuidance) | Medium-high (Kotlin bridge, own model download) |
| Qwen3 0.6B / FunctionGemma 270M (fine-tuned) | as `flutter_edge_ai_litertlm` | about 0.3–0.6 GB download and RAM; most phones | Syntax constrained; semantic accuracy low without fine-tuning (inference) | High (fine-tune and evaluate) |
| Gemini Nano via ML Kit Prompt API | Android: Pixel 9+ and some flagships | AICore-managed, no download by us | Medium: `@Generable` is alpha and Kotlin-only; prompt-based from Flutter | Medium (platform channel), small reach |
| Apple Foundation Models | iOS/macOS 26+, iPhone 15 Pro+ | built in | High with `@Generable` (Swift; experimental in Flutter) | Medium, iOS only |
| llama.cpp wrappers (GBNF) | Android, iOS, desktop | GGUF size dependent | High in principle (grammar) | High (immature packages, native builds) |

**Suggested path:**

1. Define the Timer JSON Schema once. Prototype parsing on the Gemini API through `firebase_ai`.
2. Add Gemma 4 E2B on the device through `flutter_edge_ai_litertlm`, with a flat `create_timer` tool. Example: top-level `rounds:int`, `work_s:int`, `rest_s:int`, `exercises:string[]`, plus an optional explicit `intervals` list. Validate in Dart and retry once on failure.
3. Only if validation failures show up in testing, swap in a Kotlin bridge to `ResponseFormat.json` for full schema enforcement on Android.
4. Keep the cloud API as the fallback for unsupported devices, web, or before the model download finishes.
