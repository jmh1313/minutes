# Minutes engine + compliance evaluation (2026-06-23)

> Technical direction note. What to evaluate next for transcription, diarization,
> local summarization, the live path, and compliance, grounded in June 2026 SOTA.
> Companion to the IA plan (`minutes-ia-agent-ux-2026-06-02.md`). Effort sizes are
> S/M/L. Every nontrivial claim is dated and sourced. "Confirmed" vs "uncertain"
> is called out so each item can be validated before commitment.

## Thesis

The differentiation that still holds in mid-2026 is not "local" and not "exposes
meetings to agents" on their own. Local-first meeting tools and native OS
transcription (Apple Intelligence on macOS 26 / iOS 26) are now common. The
durable position is the intersection of:

1. 100% on-device processing,
2. a live, speaker-attributed transcript feeding the append-only event bus for
   mid-meeting agent loops (not just post-meeting notes),
3. structured, queryable decisions/actions, and
4. compliance-grade controls (consent capture, access/audit log, retention) that
   no server ever touches.

The engine and compliance work below is sequenced to make that intersection real.

## 1. Transcription (ASR)

### Current direction is correct (confirmed)
The move to sherpa-onnx as a native no-Python Cargo feature loading `.onnx`
directly, with default `parakeet-tdt-0.6b-v3-int8` (25 EU languages, auto-detect,
CC-BY-4.0), is the right call. sherpa-onnx remains the leading no-Python,
cross-platform, ONNX on-device ASR runtime in 2026 with safe Rust bindings and the
broadest model coverage. Keep `whisper.cpp` + Whisper large-v3-turbo (MIT, 99
languages) as the long-tail multilingual fallback and the one mature Metal path.

Two validations before locking the default (effort S):
- Validate the int8 build against fp16/fp32. sherpa-onnx issue
  [#2605](https://github.com/k2-fsa/sherpa-onnx/issues/2605) reports dropped words
  with `nemo-parakeet-tdt-0.6b-v3-int8`. Confirm accuracy on real audio.
- Test PT-BR / LatAm-ES accents specifically. They are within the language set but
  per-accent WER is not published (uncertain).

Apple Silicon note: there is no clean Metal GPU acceleration for Parakeet via ONNX
today. CPU is the realistic target (and on M3 CPU already beats Whisper's Metal
path per the `parakeet-rs` notes). Bake CPU into expectations.

### Highest-value new model: streaming ASR for the live path (effort M)
**NVIDIA Nemotron-3.5-ASR-streaming-0.6b** (released 2026-06-04,
[HF card](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b)) is
purpose-built cache-aware streaming, 40 locales, configurable 80ms to 1.12s chunks,
license OpenMDW-1.1 (commercial OK). It is already merged into sherpa-onnx
streaming and into the `parakeet-rs` v0.3.6 Rust crate (MIT/Apache-2.0, via the
`ort` ONNX Runtime crate). This is far better for live coaching than running batch
Parakeet in a sliding window, and it is the same ecosystem as the existing
parakeet path.

Caveat to test: French streaming WER is mediocre (~9% at 1.12s chunk) vs ES/IT
(~4%). Validate French live quality before promising it.

### Watch, do not adopt yet
Cohere Transcribe (03-2026, Apache-2.0, leaderboard #1) and Qwen3-ASR-1.7B
(Apache-2.0, 52 langs) top the HuggingFace Open ASR Leaderboard with the cleanest
licenses, but neither has a no-Python on-device runtime today (transformers/vLLM
oriented). Revisit if a sherpa-onnx or Rust ONNX path lands. Moonshine v2 (MIT,
107ms latency) is the latency king but English-only, so it is disqualified for the
multilingual live path until non-English variants ship.

Licensing reminders: Parakeet/Canary are CC-BY-4.0 (attribution; add a NOTICE
line). Nemotron-3.5 is OpenMDW-1.1 (newer; read terms). Whisper/Moonshine/Qwen3-ASR/
Cohere are MIT or Apache-2.0.

## 2. Diarization + speaker ID

### Live diarization is the gap; streaming Sortformer closes it (effort M)
Live mode currently transcribes but barely diarizes. **Streaming Sortformer**
(NVIDIA) is 2026 SOTA for low-latency online diarization, and `parakeet-rs` v0.3.6
(2026-06-04) already runs streaming Sortformer v2/v2.1 in pure Rust via the `ort`
crate, no Python, with `diarize_chunk()` carrying state across calls
([lib.rs/parakeet-rs](https://lib.rs/crates/parakeet-rs)). Model license CC-BY-4.0,
code MIT/Apache-2.0. This is the single highest-value capability upgrade and it is
de-risked (same ecosystem as the parakeet path).

Constraints to validate: 4-speaker ceiling (degrades hard at 5+ speakers); CoreML
is reported unstable for this model on Apple Silicon (use WebGPU/Metal or CPU);
real-world DER on our audio is uncertain until tested. NVIDIA's official ONNX export
of the *streaming* variant is still broken as of Mar 2026 ([NeMo
#15536](https://github.com/NVIDIA-NeMo/NeMo/issues/15536)), so use the parakeet-rs /
community ONNX path, not a self-export.

### Batch diarization: resolve a licensing landmine (effort M)
The modern pyannote open model (community-1, v4.x) is CC-BY-4.0 but **gated** (HF
agreement + email + token to download) ([HF model
card](https://huggingface.co/pyannote/speaker-diarization-community-1)). That is
friction for shipping weights in a privacy-first installer. Current pyannote-rs
builds on 3.1, which is fine, but evaluate moving the batch path to the sherpa-onnx
Apache-2.0 ONNX pipeline (segmentation + WeSpeaker/3D-Speaker/TitaNet embedding +
clustering), which is ungated and reuses one runtime. Spot-check DER against current
pyannote-rs before switching (sherpa's pyannote port had a documented result
mismatch, [issue #1708](https://github.com/k2-fsa/sherpa-onnx/issues/1708)).

### Enrollment embeddings upgrade (effort S)
For `voices.db` cosine-similarity ID, 3D-Speaker (ERes2NetV2/CAM++) or WeSpeaker
ship as off-the-shelf Apache-2.0 ONNX in sherpa-onnx and beat older ECAPA-class
models. ReDimNet is the accuracy ceiling (0.287% EER Vox1-O) but needs self-export
and weights-license validation.

## 3. Local summarization / extraction

### Reframe: JSON reliability is a runtime property, not a model property
Ollama (and llama.cpp/LM Studio) constrain output to a schema at the token level
(`format` to GBNF), so valid JSON is structurally enforced regardless of model
([Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs)).
The local-LLM work is therefore mostly a sane default + a schema prompt, not a model
hunt.

### Recommended local engines via the existing Ollama path (effort S)
Keep Claude-via-MCP as the default no-key conversational path. Position local models
as the offline / private / compliance engine. Recommended:
- Qwen3.5/3.6 (Apache-2.0): best all-around local structured extraction in 2026.
- Gemma 4 (140+ languages, 256K context): best for long French-medical / pt-BR / ES
  transcripts. Note the `enable_thinking=false` gotcha so JSON lands in output.
- Mistral / Ministral 3 (Apache-2.0): strongest native French/European, small.
- Granite 4 (Apache-2.0, ISO-42001 certified) as an enterprise-compliance option.

### GLM 5.2: not viable on-device (confirmed)
GLM 5.2 (2026-06-13, MIT) is 744B total / ~40B active MoE. The smallest usable quant
is ~239GB and needs 256GB+ unified memory ([Unsloth
GLM-5.2](https://unsloth.ai/docs/models/glm-5.2)); Ollama offers only `glm-5.2:cloud`.
It cannot run on a typical Mac, and the hosted API carries a data-residency concern
that is wrong for the medical/regulated audience. Do not add local GLM 5.2. The only
locally sane GLM is GLM-4.7-Flash (30B MoE, MIT, ~18GB at 4-bit) but it is
coding-tuned with no multilingual claim, so it is dominated by the picks above.

### Apple Foundation Models (macOS 26): defer
The on-device ~3B model is usable by third-party apps via Swift, with guided
generation into structs, good for terse extraction and fully private. But it needs a
Swift/Tauri bridge (not the Rust `ureq` path) and is too small for nuanced
multilingual summaries. Worth it only as a macOS-only zero-download "instant private"
tier. Defer unless that tier becomes a priority.

## 4. The live path / "loops"

With streaming ASR (1.1) + streaming diarization (2.1) landing, the live path can
emit a live, speaker-attributed transcript on-device. That is the substrate the
append-only event bus (RFC #194) needs for mid-meeting agent loops: an agent reacting
to a live, local, speaker-labeled stream with zero cloud. This is currently deferred
past Phase 4; the streaming stack is the enabler that makes pulling it forward
reasonable.

Compliance constraint to design in from the start: the EU AI Act prohibits AI emotion
recognition from voice in the workplace (in force since 2025-02-02, fines up to E35M;
[FPF](https://fpf.org/blog/red-lines-under-eu-ai-act-unpacking-the-prohibition-of-emotion-recognition-in-the-workplace-and-education-institutions/)).
Live coaching (and `/minutes-mirror`) must stay behavioral (talk-time, filler,
monologue length) and never biometric-emotional. Medical use has an exemption.

## 5. Compliance features

Local processing gives structural advantages: no GDPR cross-border transfer, no HIPAA
BAA requirement (PHI never leaves the device), and it sidesteps the cloud-bot
recording theories now being litigated. To convert that from "architecturally true"
to "procurement-checkable," ship (effort S to M, some already in flight via the
consent layer):
- Consent capture + a recording disclosure record.
- An access/audit log rendered over the event bus.
- Configurable retention / auto-delete.

Timely context: EU AI Act Article 50 transparency obligations apply from 2026-08-02
([AI Act timeline](https://artificialintelligenceact.eu/implementation-timeline/));
France HDS v2.0 is fully effective as of 2026-05-16. HDS is a *hosting* certification,
so a tool that processes entirely on-device and never hosts data off-device may fall
outside the HDS trigger. That specific claim must be validated with French
health-data counsel before it is published anywhere. The `medical-fr` template family
(RFC 0001, issue #143) is the design-partner path for this.

## Suggested sequencing

1. Live streaming stack: wire `parakeet-rs` streaming Nemotron-3.5 ASR + streaming
   Sortformer diarization into the live path (M). Spike first to verify the
   "de-risked" claim on real audio and on Apple Silicon.
2. Loops: make a live, local, speaker-attributed agent-coaching flow first-class and
   demoable, behavioral-only (M).
3. Compliance features: consent disclosure record + audit log + retention controls,
   plus the medical-fr / HDS narrative validated with counsel (S to M).
4. Validate sherpa int8 vs fp16 and PT-BR/ES accents; keep whisper large-v3-turbo
   fallback (S).
5. Resolve diarization licensing: evaluate sherpa-onnx Apache-2.0 batch path; upgrade
   enrollment embeddings to 3D-Speaker/WeSpeaker (M).
6. Standardize the local-LLM story: Qwen3.5 / Gemma 4 / Mistral via Ollama with a
   schema prompt; document it; do not add GLM 5.2 (S).

## Sources

ASR: [HF Open ASR blog](https://huggingface.co/blog/open-asr-leaderboard),
[Parakeet-TDT-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3),
[Nemotron-3.5 streaming](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b),
[sherpa-onnx](https://k2-fsa.github.io/sherpa/onnx/index.html),
[parakeet-rs](https://lib.rs/crates/parakeet-rs).
Diarization: [streaming Sortformer](https://huggingface.co/nvidia/diar_streaming_sortformer_4spk-v2),
[pyannote community-1](https://huggingface.co/pyannote/speaker-diarization-community-1),
[sherpa-onnx diarization](https://k2-fsa.github.io/sherpa/onnx/speaker-diarization/index.html).
Local LLM: [Unsloth GLM-5.2](https://unsloth.ai/docs/models/glm-5.2),
[Ollama structured outputs](https://docs.ollama.com/capabilities/structured-outputs),
[Apple Foundation Models](https://www.apple.com/newsroom/2025/09/apples-foundation-models-framework-unlocks-new-intelligent-app-experiences/).
Compliance: [EU AI Act timeline](https://artificialintelligenceact.eu/implementation-timeline/),
[emotion-recognition prohibition (FPF)](https://fpf.org/blog/red-lines-under-eu-ai-act-unpacking-the-prohibition-of-emotion-recognition-in-the-workplace-and-education-institutions/),
[France HDS v2.0](https://www.privacyworld.blog/2026/05/v2-0-certification-of-french-health-data-hosting-service-providers-hds-now-fully-effective/).
