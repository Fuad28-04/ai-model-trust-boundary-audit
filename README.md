# AI Model Trust Boundary Audit

**Master of IT Security – Artificial Intelligence, Ontario Tech University — Portfolio Project**
**By Md. Fuad Alam**

An audit of 2,809 real Hugging Face repositories, asking a question the ecosystem has not measured: how much of a model artifact is nobody checking?

---

## The Problem

A "model" on Hugging Face is not one file. It is a bundle of eight to twenty: some weights, some inert documentation, and several that are **executable configuration** — Jinja2 chat templates, tokenizer rules, custom Python modules. These are interpreted at runtime and can change what the model does, but every scanner in the pipeline treats them as trusted settings.

In 2026, researchers demonstrated a poisoned Jinja2 chat template passing Hugging Face's entire security pipeline undetected — malware signature matching, unsafe deserialisation checks, third-party format analysis — because each layer classified it as configuration rather than code. Tokenizer-level remapping attacks were shown by NVIDIA in 2024.

Those papers proved the attacks are *possible*. **Nobody had measured how wide that surface actually is.** This project does.

## Results (n = 2,809 repositories)

### Finding 1 — Most of the artifact is executable configuration

| | Share of repositories |
|---|---|
| Ships at least one executable-config file | **72.2%** |
| Ships pickle-based weights (can execute on load) | 27.6% |
| Ships custom `.py` code | 17.0% |
| Declares remote code via `auto_map` | 7.0% |
| Ships a chat template | 31.0% |
| Template disables Jinja2 escaping (`|safe`) | 10.1% |

Median 4 executable-config files per repository; the largest ships 2,110.

### Finding 2 — The coverage gap

Hugging Face runs its own security pipeline and publishes the verdict. Of the repositories with a completed scan, **94.7% are marked completely clean**. Among those clean repositories:

| Still ships | Share |
|---|---|
| **At least one executable-config file** | **64.7%** |
| Pickle-based weights | 21.1% |
| Custom `.py` code | 19.3% |
| Template with escaping disabled | 13.3% |
| Remote code declaration | 7.5% |

**Roughly two in three repositories certified clean carry executable configuration that no scanner inspects.** The unit that gets scanned and the unit that gets deployed are not the same thing.

A single example makes it concrete. `gpt2` has 14.3 million downloads and a clean security scan reporting zero flagged files — while shipping 7 pickle-format weight files and 9 executable-config files.

### Finding 3 — Chat templates are programs 

872 templates analysed:

| | |
|---|---|
| Median length | 4,614 characters |
| Longest | **29,157 characters** |
| Median control-flow operations | 37 |
| Most | **269** |
| Templates with more than 50 control-flow operations | **43.7%** |

Nearly half of all chat templates contain more than fifty branches and loops. That is a program, not a setting. It executes on every single inference call, before the model sees any text, and nothing reads it.

### Finding 4 — Popularity does not predict safety 

| Quartile | Median downloads | Pickle | Custom `.py` | Remote code | `|safe` |
|---|---|---|---|---|---|
| Q1 most downloaded | 1,928,846 | **45%** | 13% | 6% | 10% |
| Q2 | 447,978 | 21% | 15% | 12% | 8% |
| Q3 | 111,216 | **13%** | 15% | 12% | 11% |
| Q4 least downloaded | 1,044 | 37% | **35%** | 9% | **15%** |

There is no monotonic relationship, and that is the point. Pickle exposure is **U-shaped** — highest at both extremes (45% and 37%), lowest in the middle (13%), because the most-downloaded repositories are old enough to predate safetensors and keep legacy `.bin` files for compatibility. Meanwhile custom code and permissive templates rise steadily toward the long tail, from 13% to 35%.

**No tier is safe.** Choosing a popular model trades one class of exposure for another rather than removing it.

### A usable output — trust score and CI/CD gate

Measurement is only half of it. The final section turns the audit into a deployment check: a trust score with severity-ranked findings and an ALLOW / REVIEW / BLOCK verdict.

Running it on `gpt2` returns **70/100, verdict REVIEW** — flagging the pickle weights as HIGH severity, and noting that nine executable-config files sit outside the upstream scan's scope. `Qwen/Qwen2.5-0.5B-Instruct` returns 100/100 and ALLOW, with the same note attached.

The scoring weights are a judgement call rather than a derived quantity, and they are written plainly in the code so anyone can disagree with them.

## How to read these numbers

**Attack surface is not compromise.** Nothing in this audit shows that any repository is malicious. A `|safe` filter in a chat template is frequently legitimate; long templates usually reflect genuine tool-calling logic, not an attack. What is measured is **exposure and scanner coverage**, not exploitation.

Other limits worth stating plainly:

- **Metadata only.** No model was downloaded or executed, so behavioural backdoors are entirely out of scope. Recent work argues detection of weight-space backdoors is fundamentally limited anyway, which is part of why provenance matters more than scanning.
- **The trust-score weights are a judgement call**, not a derived quantity. They are stated in the code so anyone can disagree.
- **4 of 2,813 repositories failed** (0.1%), all because they were deleted or renamed between listing and auditing. An earlier anonymous run lost 64% of the sample to rate limiting, and that run's popularity figures were noise; using an authenticated token fixed it. The lesson is worth stating rather than hiding.

## Method

The core contribution is the **trust-tier classifier**. Every file in a repository is placed in one of five tiers:

| Tier | Meaning |
|---|---|
| `CODE` | Arbitrary Python. Everyone agrees this is dangerous. |
| `EXEC_CONFIG` | **Not code by file type, but interpreted at runtime and able to change model behaviour.** This is the tier nothing scans. |
| `WEIGHTS_UNSAFE` | Tensors in a pickle-backed format that can execute on load. |
| `WEIGHTS_SAFE` | Tensors in a format that cannot execute on load (`safetensors`, `gguf`, `onnx`). |
| `INERT` | Documentation, licences, images. |

The interesting claim is the second row. A `tokenizer_config.json` looks like settings but can carry a Jinja2 template that runs on every inference. A `config.json` looks like settings but `auto_map` inside it points at Python that gets imported.

Chat templates are then analysed on two separate axes — **danger constructs** (things that let a template reach outside itself) and **control flow** (ordinary `if`/`for`/`set`, harmless in itself but evidence that "configuration" is really logic). These must not be conflated. An early version of the pattern set matched the bare word `system`, which appears as a role name in nearly every template; that flagged the whole ecosystem for nothing, and the patterns are now anchored to call syntax instead.

Sampling is stratified across six strata — top downloads, text-generation, recently created, trending, most liked, recently updated — so the sample describes more than the front page.

**Stack:** Python, `huggingface_hub`, pandas, matplotlib.

## Running It

1. Open `AI_Model_Trust_Boundary_Audit.ipynb` in [Google Colab](https://colab.research.google.com)
2. **Strongly recommended:** create a free read-only token at huggingface.co/settings/tokens and set `USE_TOKEN = True`. Without it, anonymous requests are throttled and most of the sample is lost.
3. **Runtime → Run all** — about 20 minutes. No GPU. No model is downloaded or executed.

Results are cached to disk as the audit runs, so an interrupted run resumes rather than restarts.

## Future Work

- **Cryptographic signing and verifiable provenance**, so integrity does not depend on scanning at all. If detection is fundamentally limited, provenance is the more durable answer.
- Extend the classifier to **LoRA adapter stacks**, where components that are individually clean can compose into something that is not.
- **Track the same repositories over time** — does executable-config surface grow as tool-calling gets more elaborate?

## Reference

The chat-template attack surface was demonstrated in 2026 work showing a poisoned Jinja2 template passing Hugging Face's scanning pipeline undetected, because every layer treats it as trusted configuration. Tokenizer remapping attacks were shown by NVIDIA in 2024. This project does not repeat those demonstrations — it measures how wide that surface actually is.

## Repository Structure

- `AI_Model_Trust_Boundary_Audit.ipynb` — full audit pipeline, runnable end-to-end in Colab
- `README.md` — this file

## Author

**Md. Fuad Alam**
