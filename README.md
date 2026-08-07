# cnrocr — model weights

This repository hosts the trained model weights for `cnrocr`, a container
number detection and recognition library (ISO 6346).

```bash
pip install cnrocr
cnrocr read gate_cam.jpg
```

Weights are distributed as **release assets**, not as files in the tree.

| Tag | Contents |
|---|---|
| `models-v2` | region detector + ocr recognizer (horizontal / vertical), ONNX opset 17, encrypted |

The assets are encrypted with AES-256-GCM and are decrypted into memory by the
library; they are not usable on their own. Requires `cnrocr >= 0.3.0`.

## What it looks like

Horizontal markings — the number, the ISO size-type code, and the confidence:

![horizontal](docs/horizontal.jpg)

Vertical markings are read without rotating the crop; the recognizer keeps a
separate graph for each orientation:

![vertical](docs/vertical.jpg)

Fragments split across panels (owner code and serial detected separately) are
merged back into a single number and validated against the ISO 6346 check
digit. Anything that fails the check digit, misses the spec, or falls below a
confidence floor is flagged for human review rather than reported as a result.

## Immutability

Published release assets are **never modified or replaced**. Installed versions
of the library pin each file by SHA-256, so changing an asset in place would
break every existing install.

A new model set is published under a new tag (`models-v3`, …) alongside a new
library version. Older sets remain available indefinitely.

## Sample images

The photographs above come from the
[Container Number Recognition](https://www.kaggle.com/datasets/johnkhanhnguyen/container-number-recognition)
dataset by johnkhanhnguyen, used under the MIT licence. It is a convenient
source of test images if you want to try the library before pointing it at your
own cameras.

## License

Proprietary. The weights are licensed for evaluation and non-commercial
research use only. Contact the copyright holder for commercial licensing.

Without a licence key the library processes 10 images per day. To request an
unrestricted licence, email **vislab2026@gmail.com** with your name,
organisation and intended use.
