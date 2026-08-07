# cnrocr

Container number detection and recognition — **ISO 6346** end-to-end, ONNX only.

[![PyPI](https://img.shields.io/pypi/v/cnrocr.svg)](https://pypi.org/project/cnrocr/)
[![Python](https://img.shields.io/pypi/pyversions/cnrocr.svg)](https://pypi.org/project/cnrocr/)
[![Platform](https://img.shields.io/badge/platform-linux%20%7C%20macOS%20%7C%20Windows-lightgrey.svg)](https://pypi.org/project/cnrocr/#files)
[![License](https://img.shields.io/badge/license-Proprietary-red.svg)](#license)

A region detector locates the number, ISO type code, owner code and serial on
the container; an OCR recognizer reads each crop with a spec-constrained beam
search. Fragments split across panels are merged back into a single number and
validated against the ISO 6346 check digit.

**No PyTorch required.** The runtime is `onnxruntime` + `numpy` + `pillow`.

This repository also hosts the model weights as release assets — see
[Model weights](#model-weights).

---

## What it looks like

Horizontal markings — the number, the ISO size-type code, and the confidence:

![horizontal](docs/horizontal.jpg)

Vertical markings are read without rotating the crop; the recognizer keeps a
separate graph for each orientation:

![vertical](docs/vertical.jpg)

---

## Install

```bash
pip install cnrocr           # CPU
pip install cnrocr[gpu]      # NVIDIA CUDA via onnxruntime-gpu
```

Python 3.10–3.13 on linux x86_64/aarch64, macOS arm64, Windows amd64.

The weights (~113 MB) are **not** bundled in the wheel. They are downloaded on
first use and cached locally:

```bash
cnrocr models download       # optional — happens automatically otherwise
cnrocr check                 # verify the install, providers and cache
```

## Quick start

```bash
cnrocr read gate_cam.jpg
```

```
gate_cam.jpg  (172.1 ms)
  [  OK  ] TEMU3108252  22G1  conf 1.000
```

Anything the pipeline is not confident about is marked instead of being
reported as a result:

```
  [REVIEW] MSCU1234566  45R1  conf 0.612
           -> low confidence (0.612 < 0.7)
```

## Command line

| Command | What it does |
|---|---|
| `cnrocr read a.jpg b.jpg` | Read one or more images |
| `cnrocr read *.jpg --json` | Emit JSON, including box coordinates |
| `cnrocr multiview c1.jpg c2.jpg c3.jpg` | Fuse several views of **one** container |
| `cnrocr models download` | Fetch the weights ahead of time |
| `cnrocr models status` | Show what is cached and where |
| `cnrocr check` | Diagnose install, execution providers, cache |
| `cnrocr license` | Show the licence and today's usage |

Useful flags: `--device auto|cpu|cuda`, `--threshold 0.5` (detection score
floor), `--review-confidence 0.7`, `--registry bic_codes.txt`, and
`--fail-on-review`, which exits with code 2 when anything needs review — handy
in batch pipelines.

## Python API

```python
from cnrocr import ContainerOCR

ocr = ContainerOCR()                      # weights resolved from cache
result = ocr.read("gate_cam.jpg")

for c in result.containers:
    print(c.number, c.iso_type, c.confidence, c.needs_review)
# TEMU3108252 22G1 0.9997 False
```

Several unrelated images at once — N images in, N results out:

```python
results = ocr.read_many(["a.jpg", "b.jpg", "c.jpg"], batch_size=8)
```

### Multi-view fusion

When several cameras photograph the **same** container, fuse them into one
answer instead of voting on strings:

```python
mv = ocr.read_multiview(["cam1.jpg", "cam2.jpg", "cam3.jpg"])
print(mv.number, mv.agreement, mv.mode)
# TCNU9523625 1.0 candidate_sum
```

Beam-search candidates from every view are summed in log space, so a character
one view is unsure about can be settled by the others. If the views turn out to
be looking at *different* containers, they are not fused — `mv.consensus`
becomes `False` rather than producing a confident wrong answer.

### Review triage

A check digit alone is not enough. The constrained decoder only emits
spec-conforming candidates, so when the true string is absent from the beam it
will confidently output a *plausible* wrong number that still passes the check
digit. `needs_review` combines the check digit, the spec flag and a confidence
floor:

```python
if c.needs_review:
    print(c.review_reason)     # "low confidence (0.612 < 0.7)"
```

### Owner code registry (optional)

Real-world owner codes are registered with the BIC. Supplying the list filters
out invented codes that would otherwise pass both the format check and the
check digit:

```python
from cnrocr import OwnerCodeRegistry
ocr = ContainerOCR(registry=OwnerCodeRegistry.from_file("bic_codes.txt"))
```

The list is not shipped with the package.

## Evaluation limit

Without a licence, cnrocr processes **10 images per day**. The counter is per
image, not per call, and resets at 00:00 UTC.

```bash
cnrocr license      # licence status and today's usage
```

A call that would exceed the quota processes nothing and exits with code 3 —
all or nothing, so a refused call costs no quota.

To request an unrestricted licence, email **vislab2026@gmail.com** with your
name, organisation and intended use. You will receive a key:

```bash
# Windows
setx CNROCR_LICENSE "eyJlbWFpbCI6..."

# macOS / Linux
export CNROCR_LICENSE='eyJlbWFpbCI6...'
```

Verify with `cnrocr check`.

---

## Model weights

Weights are distributed as **release assets**, not as files in the tree.

| Tag | Contents |
|---|---|
| `models-v2` | region detector + ocr recognizer (horizontal / vertical), ONNX opset 17, encrypted |

The assets are encrypted with AES-256-GCM and are decrypted into memory by the
library; they are not usable on their own. Requires `cnrocr >= 0.3.0`.

### Cache

| | |
|---|---|
| Windows | `%LOCALAPPDATA%\cnrocr\Cache\models\<set>` |
| macOS | `~/Library/Caches/cnrocr/models/<set>` |
| Linux | `~/.cache/cnrocr/models/<set>` |

Every file is verified against a SHA-256 recorded in the wheel. The cache holds
ciphertext only; no plaintext model is written to disk.

| Environment variable | Effect |
|---|---|
| `CNROCR_LICENSE` | Licence key, or a path to a file holding it |
| `CNROCR_MODEL_DIR` | Use this directory as-is; never download |
| `CNROCR_CACHE_DIR` | Relocate the cache root |

### Immutability

Published release assets are **never modified or replaced**. Installed versions
of the library pin each file by SHA-256, so changing an asset in place would
break every existing install.

A new model set is published under a new tag (`models-v3`, …) alongside a new
library version. Older sets remain available indefinitely.

---

## Sample images

The photographs above come from the
[Container Number Recognition](https://www.kaggle.com/datasets/johnkhanhnguyen/container-number-recognition)
dataset by johnkhanhnguyen, used under the MIT licence. It is a convenient
source of test images if you want to try the library before pointing it at your
own cameras.

## License

Proprietary. Evaluation and non-commercial research use only.
Contact the copyright holder for commercial licensing.
