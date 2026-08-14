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
A [local server](#local-server) with a review dashboard ships in the same
package — `pip install "cnrocr[server]"`.

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
pip install cnrocr             # CPU
pip install "cnrocr[gpu]"      # NVIDIA CUDA — see the note below
pip install "cnrocr[server]"   # local REST API + dashboard
pip install "cnrocr[camera]"   # webcams and RTSP streams — see Cameras
```

Python 3.10–3.13 on linux x86_64/aarch64, macOS arm64/x86_64, Windows amd64.

The weights (~113 MB) are **not** bundled in the wheel. They are downloaded on
first use and cached locally:

### About the GPU extra

`onnxruntime-gpu` does not carry the CUDA runtime, so `cnrocr[gpu]` alone is
often not enough. With the runtime missing or a different major version,
onnxruntime prints an error, **carries on using the CPU**, and the only
symptom is that inference is slow.

- Remove the CPU build first. The two share one directory and cannot both own
  it: `pip uninstall -y onnxruntime && pip install "cnrocr[gpu]"`
- Check what you got. `cnrocr check` prints the providers actually in use, and
  `--device cuda` warns when it lands on the CPU anyway.

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

**No container photographs to hand?** `cnrocr samples` fetches three, so there
is something to try before you point this at a camera:

```bash
cnrocr samples --dir photos
cnrocr read photos/*.jpg
```

```
wiki_01.jpg  [  OK  ] MSMU7761306  45G1  conf 0.999
wiki_02.jpg  [REVIEW] TNSU1018530  22G1  conf 0.010
                      -> low confidence (0.010 < 0.7)
wiki_03.jpg  [  OK  ] SUDU1782454  22G1  conf 0.997
```

One of the three goes to review on purpose — see [Sample images](#sample-images).

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
| `cnrocr license --set KEY` | Register a key (or paste it into the dashboard) |
| `cnrocr server start` | Run the local API and dashboard — see [Local server](#local-server) |
| `cnrocr watch --source webcam:0` | Watch a camera and read what a trigger catches — see [Cameras](#cameras) |
| `cnrocr fake-plc --to URL` | Knock on a trigger the way a gate controller would, and name what came back |
| `cnrocr samples --dir ./photos` | Download a few container photographs to try |
| `cnrocr export --out ./dataset` | Write what has been read as a training set — see [Collecting data](#collecting-data) |

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

---

## Local server

A REST API and a browser dashboard, both served from your own machine. Images
never leave it.

```bash
pip install "cnrocr[server]"
cnrocr server start                  # http://127.0.0.1:8000
cnrocr server start --daemon         # background; `cnrocr server stop` to end it
```

The models load once at startup and are shared by every request. Open the
address for the dashboard, or `/docs` for the interactive API.

![dashboard](docs/dashboard.png)

### Linking to a tab

Each tab has an address, so `http://gate-pc:8000/#history` opens there. Useful
when telling somebody which screen to look at, and the browser's back button
now returns to the tab you came from rather than leaving the dashboard.

### The review queue

The dashboard leads with the queue rather than with a file picker, because the
interesting question is not "what did it read" but "which ones should a human
look at".

Results below `--review-confidence` (default `0.7`) are collected instead of
being silently trusted. On our validation set every misread scored below 0.7
while correct reads sat near 1.0, so the threshold catches the errors and sends
only a small fraction of good reads along with them — the run above is 60
images with one flagged, at confidence 0.036.

### Which stage failed

Clicking a photograph opens it with the **detected regions drawn on top** and
a line saying where the reading stopped:

| | |
|---|---|
| **Nothing detected** | no region was found, so the recogniser was never given anything |
| **Detected but not read** | the region was located and came back empty — the detector is not at fault |
| a number and its confidence | read, with the check digit and any merge shown |

Both of the first two look like "no number" on a list, and they are different
faults in different models. Telling them apart is what decides whether a
problem is worth chasing, and — if it ever comes to retraining — which data
would help.

History carries a date range, a search, CSV export, and deletion of selected
rows. Deleting takes the stored photographs with it and cannot be undone.
Clicking an ISO code in the table narrows the list to it.

Confirming a correction stores the corrected value. That record is the only
evidence of which misreadings repeat, and therefore of what is worth fixing.
Corrections are validated against the ISO 6346 check digit too: a typo must not
acquire a "checked by a human" stamp on its way downstream.

**Both numbers survive, and either one finds the row.** History shows the
corrected value with the original beneath it, and searching for either returns
the same reading. That is not a courtesy to the machine: the webhook carried
the original downstream at the moment of the reading, so it is the key that
matches whatever the gate system recorded. Exports lead with `final_number`,
which is the human's answer where there is one.

### At the gate

<table>
<tr>
<td width="330"><img src="docs/dashboard-mobile.png" width="310" alt="the dashboard on a phone"></td>
<td valign="top">

The layout is mobile first. The camera button photographs a container and
uploads it directly, which is enough to work the queue where you stand.

```bash
cnrocr server start --host 0.0.0.0 --token "$(openssl rand -base64 24)"
```

Binding to `0.0.0.0` exposes the server to everyone who can reach the host, so
set a token when you do. The server says so at startup if you forget; it does
not refuse to start, because a closed network is a legitimate setup.

</td>
</tr>
</table>

### API

```bash
curl -X POST -F "file=@gate_cam.jpg" http://127.0.0.1:8000/api/read
```

```json
{
  "detection_id": 41,
  "device": "cuda",
  "containers": [{"number": "TGHU8913889", "iso_type": "22G1",
                  "confidence": 0.9997, "needs_review": false}],
  "elapsed_ms": 38.4
}
```

| Endpoint | Purpose |
|---|---|
| `POST /api/read` | One image (multipart). `/api/read/base64` and `/api/read/binary` take other shapes |
| `POST /api/read/batch` | Several images in one call |
| `POST /api/multiview` | Several views of the **same** container, fused into one answer |
| `GET /api/review` | The queue of results a human should look at |
| `POST /api/review/{id}` | Record the verdict; corrections are check-digit validated |
| `GET /api/history` | Past detections, with filters and CSV/JSON export |
| `GET /api/health` | Liveness, version, and which device is actually in use |
| `GET /api/stats` | Counts, review rate, remaining quota |

`/api/health` reports the device that onnxruntime **actually** chose next to the
one you asked for. When a CUDA major version does not match, onnxruntime warns
and quietly falls back to CPU; without this you would only notice that
inference got slow.

### Configuration

Flags, a YAML file, or the environment — later wins, and flags win over all.

```yaml
server:
  host: 127.0.0.1
  port: 8000
  api_token: ""            # required in practice once host is not loopback
models:
  device: auto             # auto | cpu | cuda
  workers: 4
  review_confidence: 0.7
storage:
  save_images: true        # the review screen needs the photo
  max_history: 5000
```

Every setting also reads from `CNROCR_SERVER_<NAME>`. An unknown key in the
YAML is an error rather than a silent no-op — a typo in `api_token` must not
quietly leave authentication off.

```bash
cnrocr server status        # is it up, on what device, since when
cnrocr server list          # every instance this machine knows about
cnrocr server logs -f       # follow
cnrocr server read img.jpg  # send a file to a running instance
cnrocr server install       # a unit file that starts it at boot
cnrocr server stop
```

### Telling something else

The API is pull-only, which is no use to a barrier or a terminal operating
system. Point the server at a URL and every result is POSTed there as it
happens:

```bash
cnrocr server start --webhook https://gate.internal/cnrocr
```

```json
{"event": "read", "server": "cnrocr", "ts": "...", "data": { ... }}
```

`event` is `read` for a recognition and `review` for a human verdict, so a
receiver can supersede what it was told when the read first came in. Delivery
never blocks or fails a request — a detection that was stored succeeded
whether or not anyone could be told. Failures are retried and then counted in
`/api/stats`, because a webhook that quietly stopped working is otherwise
invisible. If deliveries must not be lost, poll `/api/history` and treat the
webhook as a latency improvement rather than a transport.

### Running unattended

A gate PC reboots. `cnrocr server install` prints a systemd unit, a launchd
plist or a `schtasks` command for the machine it is run on, along with the one
command that installs it. It does not install anything itself — that needs
administrator rights, and both the unit and the command should be read first.
It also lists what will break after the next reboot but works now, such as a
licence key that only exists in the current shell.

Photographs kept for the review screen are capped separately from the history,
since a row costs a few hundred bytes and the picture beside it costs a few
hundred kilobytes:

```yaml
storage:
  save_images: true
  max_history: 5000        # rows
  max_images: 2000         # photographs; they age out first
```

---

## Cameras

The server answers questions; nothing in it goes and looks. `cnrocr watch`
does that — it holds cameras open and reads whatever a trigger catches.

```bash
pip install "cnrocr[camera]"
cnrocr server start --daemon
cnrocr watch --source webcam:0 --trigger key
```

```
watching 1 source(s) -> http://127.0.0.1:8000
  webcam:0
press Enter to read, q to quit

  14:50:04  1 view   MSMU7761306  0.999   219 ms
  14:50:11  1 view   TNSU1018530  0.010?  204 ms
```

A `?` marks a reading that went to the review queue. Everything `watch` sends
appears in the dashboard as it arrives — History, the review queue and the live
log — so the usual way to demonstrate this is with the browser open beside the
terminal.

`watch` and the server are separate processes on purpose: the agent touches
hardware and the server does not, so the server stays something you can move
into a container or onto another machine without thinking about drivers.

### Sources

| `--source` | |
|---|---|
| `webcam:0` | A camera on this machine. `webcam:1` for the second one |
| `rtsp://user:pass@host:554/stream` | A network camera |
| `./photos` | A folder of images, or a single image file — replayed in a loop |

A folder needs nothing installed beyond cnrocr itself, so you can watch the
whole path work before deciding what camera to buy — and `cnrocr samples` will
give you one:

```bash
cnrocr samples --dir photos
cnrocr watch --source photos --trigger key
```

RTSP streams are opened over TCP rather than UDP. Lost packets show up as
smeared blocks across the very characters being read.

### Triggers

| `--trigger` | |
|---|---|
| `key` *(default)* | Press Enter to read. What you want at a desk |
| `interval:5` | Every five seconds. Unattended demonstrations |
| `http` | Opens a small endpoint; something else decides when |

```bash
cnrocr watch --source rtsp://cam/stream --trigger http --trigger-host 0.0.0.0
```

```
POST http://10.0.0.31:8100/ to read, Ctrl-C to quit

trigger  0.0.0.0:8100   hits 47  reads 47  last 3s ago from 10.0.0.7
```

The HTTP trigger is the integration path. Gate controllers, PLCs and the
Ethernet I/O modules that turn a loop detector's dry contact into a network
event all speak HTTP or can be made to, which means no driver has to ship for
any of them.

**It listens on this machine only until you say otherwise.** `--trigger-host
0.0.0.0` opens it to the network, and `--trigger-token` puts a bearer token on
it, which is worth doing at the same time: the endpoint takes a photograph on
request. When you do open it the agent prints **the address a controller can
actually be configured with** rather than `0.0.0.0`, because that is the string
somebody is about to type into a device.

**Knocks are counted, with who sent them.** `hits` against `reads` is the
number that separates the two faults you cannot otherwise tell apart: a trigger
that never arrives, and one that arrives while the camera is down.

**Why not connect when the trigger arrives?** RTSP negotiation plus the wait
for a keyframe costs one to three seconds, and by then the truck has gone.
Frames are decoded continuously and the most recent second is kept, so a
trigger picks from pictures that have already arrived — the sharpest one near
the moment it fired.

### Proving the wiring

A gate has two connections and both are silent when they work. These check
them.

```bash
cnrocr watch --source rtsp://cam/stream --trigger http --check
```

```
sources
  rtsp://cam/stream                     ok      1920x1080
trigger in
  listening 0.0.0.0:8100                ok
    reachable at 10.0.0.31:8100
  waiting 10s for a knock...
    none arrived. Try: cnrocr fake-plc --to http://10.0.0.31:8100
server out
  http://10.0.0.31:8000                 ok      cnrocr 0.5.6
  its webhook                           ok      sent 412  failed 0
```

It exits with a nonzero status if anything is wrong, so an installation script
can use it.

That answers "is the port open" **from the agent's side**. Whether the
controller's machine can reach it is a different question, and only that
machine can answer it:

```bash
cnrocr fake-plc --to http://10.0.0.31:8100 --key    # knock on every Enter
```

```
  knock 1  ok       202  11 ms
  knock 2  FAILED   connection refused. The host is up and nothing is
                    listening there. Is cnrocr watch running with
                    --trigger http?
```

**Refused and timed out are different faults.** Refused means the host was
reached and nothing was listening — a process or port problem. A timeout means
the host was not reached at all — a firewall or the wrong address. They are
fixed by different people, and this says which one you have.

### Several cameras, one answer

A gate lane points four to six cameras at the same container and whichever face
is readable wins. Repeat `--source` and they are treated as one lane:

```bash
cnrocr watch \
  --source rtsp://cam-rear/stream \
  --source rtsp://cam-left/stream \
  --source rtsp://cam-right/stream \
  --trigger http
```

Every camera contributes a frame from the moment the trigger fired; they are
read together and fused into a single answer. **This costs one read, not
three** — adding cameras improves the result without changing the bill. If one
camera is down the others still answer.

### Mounting

Aim the camera as square to the container face as the site allows. Numbers
photographed steeply from above and to one side are the ones that get misread,
and a misreading that happens to satisfy the ISO 6346 check digit will not be
caught by it.

### Leaving it running

A gate PC reboots, and an agent that does not come back takes the lane with
it. `--service` writes a boot unit for the command you just typed:

```bash
cnrocr watch --source rtsp://cam/stream --trigger http --service
```

It prints the unit and the one command that installs it, and installs nothing
itself — registering a service needs administrator rights, and a tool that
quietly takes them is not one to run on a machine that matters. systemd,
launchd and Windows scheduled tasks are all covered; `--trigger key` is
refused with an explanation, because a service has no keyboard.

While it runs, the agent reports to the server every ten seconds with the age
of each camera's last frame, and the System tab shows it:

```
gate-pc-1   last seen 4s ago    rtsp://cam-rear/stream — 0.4s ago
                                rtsp://cam-left/stream — no frames
```

The header carries a shorter version of the same thing on every screen —
`4 cameras`, or `3/4 cameras` when one has stopped delivering, or `cameras ?`
when no agent has reported for a minute. It is hidden entirely if you have
never run `watch`.

Two more chips sit beside it, on the same rule: **shown only once there is
something to report.** `io` follows the gate trigger, and `webhook` follows
what the server is posting onward.

The trigger is the harder one to state honestly. A camera is healthy when
frames arrive, and they arrive constantly. A trigger that has been quiet for
three hours is either a quiet gate or a fallen cable, and no screen can tell
those apart. So the chip says only what it can defend — the port is bound, this
long since the last knock, and **`io 47/46`** when a knock produced no read,
which is the one unambiguous fault. If the agent itself goes quiet the chip
becomes `io ?`, because a green light over a dead agent is the worst thing a
header can show.

**A quiet gate and a dead agent look identical** from a dashboard. The only
thing that separates them is how long it has been since the agent last said
anything, so that is what the screen shows.

### Options

| | |
|---|---|
| `--server URL` | Where to send. Default `http://127.0.0.1:8000` |
| `--token KEY` | If the server requires one. `CNROCR_API_TOKEN` also works |
| `--trigger-port N` | Port for `--trigger http`. Default 8100 |
| `--trigger-host A` | What it listens on. Default is this machine only; `0.0.0.0` opens it to the network |
| `--trigger-token T` | Require a token on the trigger. Worth setting whenever the host is not loopback |
| `--check` | Test the cameras, the trigger and the server, then exit |
| `--service` | Write a boot unit instead of running |

## Collecting data

Every reading the server handles is already kept: the photograph under
`images/`, and beside it the number, the ISO type, the confidence, the review
verdict and **the box coordinates** the detector produced. What was missing
was a way to get at it without opening the SQLite file by hand.

```bash
cnrocr export --out ./dataset --since 2026-08-01
```

```
dataset/
  images/           the photographs
  labels.jsonl      one record per reading
```

```json
{"id": 48, "images": ["00000048_0.jpg", "00000048_1.jpg"],
 "ts": "2026-08-11T08:30:36+00:00", "kind": "multiview",
 "number": "CPMU2984019", "iso_type": "22G1", "confidence": 0.707,
 "needs_review": false,
 "boxes": [{"label": "container_no", "box": [726.5, 286.2, 1059.9, 349.2]},
           {"label": "iso_type", "box": [873.8, 366.6, 978.6, 426.8]}],
 "corrected": "MPMU2984019", "agrees": false}
```

**A multiview stays one record.** Several photographs of one container that
disagree are the interesting case, and splitting them into separate rows would
throw away the only thing that makes them worth keeping.

**A correction sits beside the machine's answer, not on top of it.** When
`agrees` is false you are looking at a labelled error, which is worth more
than either number alone.

`--reviewed-only` keeps just the readings a human confirmed or corrected —
the part with a verified label on it. Reviewed photographs are never removed
by the retention cap either; they are the most valuable pictures on the disk
and used to be the first ones deleted.

The server does not have to be running; this reads the database directly. And
nothing is uploaded anywhere. It writes a folder, and what happens to that
folder is between you and whoever owns the containers.

## Licensing

Licences are priced by **daily volume**: a licence raises the daily image
limit to the tier you are on, from a few hundred a day up to unlimited.
Nothing else changes — the software is the same either way, there is no
account to create, and nothing phones home.

Email **vislab2026@gmail.com** with a rough idea of your daily volume and we
will tell you the tier and the price. Keys are issued by return.

> **Multi-view counts per view.** `read_multiview` with three photographs of
> one container spends three images, not one. Three hundred containers shot
> from three angles is 900 images a day, not 300 — size it from images, not
> from containers.

```bash
cnrocr license      # which tier, and today's usage
```

The counter is per image, not per call, and resets at 00:00 UTC. The library,
the CLI and the server all draw on the same daily budget. A call that would
exceed it processes nothing and exits with code 3 — all or nothing, so a
refused call costs none of your allowance. The server answers **429** rather
than 500, since the request was fine and so is the server.

`cnrocr check` warns for thirty days before a licence expires. A renewal
should not be discovered by a system that stopped working.

## Evaluation

Without a licence, cnrocr processes **30 images per day** — enough to see
whether it reads your photographs, not enough to wire up the API and put load
through it.

Email the same address for a **free 14-day evaluation key with no daily
limit**. Nothing to negotiate, no card. When it expires the key stops applying
and you are back to 30 a day; nothing breaks and there is nothing to
uninstall.

A key looks like this:

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

```bash
cnrocr samples --dir ./photos
```

Three container photographs, so there is something to try before pointing this
at your own cameras. One reads cleanly, one carries an ISO type, and one goes
to the review queue — which is the part worth seeing. A reader that never says
"I am not sure" is not safer, only quieter.

They are not bundled in the wheel and they are **not ours**: they are used
under Creative Commons licences and are not covered by the cnrocr licence.
`cnrocr samples` writes a `CREDITS.md` beside them naming each photographer
and licence. If you republish one, those terms come with it.

The screenshots above use images from the
[Container Number Recognition](https://www.kaggle.com/datasets/johnkhanhnguyen/container-number-recognition)
dataset by johnkhanhnguyen, under the MIT licence.

## License

Proprietary. Evaluation and non-commercial research use only.
Contact the copyright holder for commercial licensing.
