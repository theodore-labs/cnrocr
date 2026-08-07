# cnrocr — model weights

This repository hosts the trained model weights for `cnrocr`, a container
number detection and recognition library (ISO 6346).

Weights are distributed as **release assets**, not as files in the tree.

| Tag | Contents |
|---|---|
| `models-v1` | D-FINE detector + CRNN recognizer (horizontal / vertical), ONNX opset 17 |

## Immutability

Published release assets are **never modified or replaced**. Installed versions
of the library pin each file by SHA-256, so changing an asset in place would
break every existing install.

A new model set is published under a new tag (`models-v2`, …) alongside a new
library version. Older sets remain available indefinitely.

## License

Proprietary. The weights are licensed for evaluation and non-commercial
research use only. Contact the copyright holder for commercial licensing.
