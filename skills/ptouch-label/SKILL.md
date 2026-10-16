---
name: ptouch-label
type: procedure
description: "Print tape labels on a Brother P-touch network label printer (PT-P750W or PT-P950NW) with the ptouch-label CLI: probe the loaded tape width first, size the label to the object, preview, then print at 360 dpi over the raster protocol with half-cuts between labels. Auto-invoke when the user wants to print or create a label, or mentions P-touch, TZe tape, label tape, ptouch-label, PT-P750W or PT-P950NW."
tags: [printing, labels, brother, p-touch, tze]
freshness:
  last_refreshed_at: 2026-10-15
  auto_refresh_cadence: quarterly
---

# P-touch label printing

**One tool, two printer models.** `ptouch-label` drives the Brother PT-P750W and PT-P950NW
over the same Brother raster protocol (TCP port 9100) or, as a fallback, AirPrint through
CUPS. Rendering is identical across models; only the print-head dimensions differ.

| | PT-P750W | PT-P950NW |
|---|---|---|
| Print pins | 128 | 560 |
| Native dpi | 180 (360 in hi-res) | 360 (720 in hi-res) |
| Tape widths (laminated TZe) | 6, 9, 12, 18, 24 mm | 3.5, 6, 9, 12, 18, 24, 36 mm |
| Typical connection | Wi-Fi | wired Ethernet |

## Before you print (read this screen, then print)

1. **What is it for?** Name the object's form factor and pass it: `--for mini-pc` (known kinds:
   cable, peripheral, power-brick, phone, tablet, mini-pc, tower, nas, rack, other). The CLI caps
   the tape width for that kind and refuses wider tape. A label that *fits* can still be far too
   big: a 27 mm bold name on 24 mm tape fit a NUC lid and was wrong. Know the clear area and pass
   `--fit LxH` (mm) when you do. An agent cannot print without `--for`.
2. **Several labels are ONE job.** `--label A --label B`, or `--image a.png --image b.png`.
   That is the only way to get half-cuts between them. **`--chain` is not it:** it leaves NO cut
   and refuses unless you add `--no-cut-ok`.
3. **Say only what the `Cuts:` line says.** Every job prints one (`Cuts: half-cut between the 3
   labels, full cut at the end`). Quote it to the user. Never describe a cut you did not see there.
4. **Probe, size, preview, print.** The probe says what tape is loaded, not what the object needs.
   If the loaded tape is wider than the form factor allows, ask for a narrower cassette.

## Install and configure (once per Mac)

```bash
pipx install git+https://github.com/johntrandall/ptouch-label     # or: uv tool install git+https://github.com/johntrandall/ptouch-label
```

Per-printer settings live in `~/.config/ptouch-label/config.toml`, not in the code. Name your
printers there (CUPS queue names, hosts, the default model, a measured vertical offset):

```toml
default_model = "PT-P950NW"

[printers.PT-P750W]
cups_queue = "Brother_PT_P750W"      # see `lpstat -p`
host = "brw0123456789ab.local"       # or an IP; optional, the IPP probe can discover it

[printers.PT-P950NW]
cups_queue = "Brother_PT_P950NW"
host = "label-printer.example.lan"
vertical_offset_mm = 0               # set after --calibrate-vertical, see below
```

`PTOUCH_HOST`, `PTOUCH_MODEL` and `PTOUCH_PRINTER` override the file for one shell;
`PTOUCH_LABEL_CONFIG` points at a different file. If the user has no config yet, ask which model
and queue they have (`lpstat -p` lists queues) and write the file before the first print.

## Model selection

The CLI defaults to the PT-P750W unless `default_model` says otherwise. Precedence, highest first:

1. `--model {PT-P750W,PT-P950NW}`
2. `--printer NAME` (a CUPS queue; a queue that maps to a known model switches the model too)
3. `PTOUCH_MODEL`, then `PTOUCH_PRINTER`
4. `default_model` in the config file, else PT-P750W

```bash
ptouch-label "Hello"                          # default model
ptouch-label --model PT-P950NW "Hello"        # explicit
export PTOUCH_MODEL=PT-P950NW; ptouch-label "Hello"   # sticky for a session
```

The resolved printer host is cached per model under `~/.cache/ptouch-label/` so probes do not collide.

## The rules that save tape

1. **Probe before every job.** By default the CLI asks the printer over IPP which tape is
   loaded and sizes the label to it. The probe is the only trusted source of tape width: not
   what the user said earlier, not a notification, not what was loaded yesterday. If the probe
   fails, wake the printer and probe again (`ptouch-label --probe`); do not bypass.
2. **Size the label to the object, not the text.** The probe says what tape is loaded, not what
   the object needs. If the loaded tape is wider than the item warrants (a short name on a cable,
   a peripheral), ask the user to load a narrower cassette rather than printing big. Look up the
   object's real dimensions first (a datasheet, a retailer spec table, or ask the user to measure
   the label-bearing face), then check the CLI's reported `NN mm long` against them. Stack text
   onto more lines when length is the constraint: one line of text on 12 mm tape can come out
   longer than the power brick it is for, where two lines fit easily.
3. **Preview first.** `--preview` writes a PNG; show it and confirm before printing, unless the
   user said to print directly. Tape is expensive.
4. **Batch, and half-cut between labels.** N separate jobs cost N leader feeds (about 23 mm of
   blank tape each, cut off as waste). One job with `--copies N`, or one job with several
   `--label` or several `--image` arguments, pays the leader once and leaves a strip with
   half-cuts between labels, which peel apart by hand and stay in order. The job's `Cuts:` line
   states the result; report that, not what you intended.
5. **Hi-res is the default.** Every call prints at 360 dpi horizontal over the raster protocol.
   AirPrint at 180 dpi (`--no-high-resolution`) is visibly fuzzier on tape; use it only for a
   large batch where the roughly 2x speed matters.

## Usage

```bash
ptouch-label "Hello World"                        # one label, auto-sized to the loaded tape
ptouch-label "Server Room" "Rack A-1"             # two lines on one label
ptouch-label --bold "DANGER"
ptouch-label --preview "Check this first"         # PNG at /tmp/ptouch-preview.png, nothing printed
ptouch-label --probe                              # report the loaded tape width and exit (also wakes the radio)
ptouch-label --tape 12 "Small label"              # override the probe; warns if it disagrees with the loaded tape
ptouch-label --length 80 "Longer label"           # fixed length in mm
ptouch-label --font-size 32 "Custom"
ptouch-label --copies 12 --bold "PWR ONLY"        # twelve identical labels, one feed, half-cuts between
ptouch-label --label "NAS" "DS1821+" --label "Scanner" "iX500"   # different labels, one job, half-cuts between
ptouch-label --image a.png --image b.png          # the same, from pre-rendered images (previews)
ptouch-label --for mini-pc --fit 60x15 "quirrell" # cap the tape width by object kind; refuse if it does not fit
ptouch-label --mirror "Reverse me"                # for transparent or iron-on tape
ptouch-label --dry-run "Debug"                    # show the command, print nothing
```

## Flags

| Flag | Short | Default | Description |
|---|---|---|---|
| `--tape` | `-t` | auto-probe | Tape width in mm (use `4` for 3.5 mm tape). Without it the printer is probed; the CLI refuses to print (exit 2) if the probe fails. |
| `--no-probe` | | off | Skip the probe. Requires `--tape N`; only for a printer that is genuinely offline and whose cassette the user has read by eye. |
| `--probe` | | off | Probe and exit. |
| `--label` | | | Repeatable: one physical label per `--label`, its arguments are the lines. Several `--label`s print as one job with half-cuts between. Mutually exclusive with positional lines. |
| `--image` | | | Repeatable: a pre-rendered image per `--image`, scaled to the tape, white trimmed (`--no-trim` keeps it). Several print as one job with half-cuts between. |
| `--for` | | | The object's form factor (cable, peripheral, power-brick, phone, tablet, mini-pc, tower, nas, rack, other). Caps the tape width by label kind; wider tape is refused. Required for a real print when an agent runs the tool. Override the table under `[form_factors]` in the config. |
| `--kind` | | inferred | `name` (one line, read at a glance) or `stats` (several lines). Inferred: one line of text is a name; more lines or an image is stats. |
| `--fit` | | | `LxH` in mm, the clear area. Refuses a label longer than L or tape wider than H. |
| `--length` | `-l` | auto | Label length in mm. |
| `--font-size` | `-s` | auto | Font size in px (auto targets 70 % of the printable height per line). |
| `--bold` | `-b` | off | Bold font. |
| `--padding` | | 4.0 | Horizontal padding in mm. |
| `--preview` | `-p` | off | Write a PNG instead of printing. `--preview-path` sets where. |
| `--dry-run` | `-n` | off | Show the command only. |
| `--copies` | `-c` | 1 | Identical copies in one feed (1 to 99), half-cuts between. |
| `--half-cut` | | off | Half-cut between pages. Implied by `--copies` and `--label`. |
| `--chain` | | off | Leave the tape uncut after the job: NO cut between it and the next job (scissors). Refuses without `--no-cut-ok`. Not how labels are joined; see below. |
| `--mirror` | | off | Mirrored print, for transparent or iron-on tape. |
| `--special-tape` | | off | No-cut mode for non-laminated decorative tape (untested on real special tape). |
| `--high-resolution` / `--no-high-resolution` | | on | 360 dpi raster over port 9100, or 180 dpi AirPrint. |
| `--calibrate-vertical`, `--vertical-offset-mm` | | | See Vertical calibration. |

## Two backends

- **Raster protocol (default):** TCP port 9100, through the `nbuchwitz/ptouch` library. This is
  what Brother's own P-touch Editor uses. It is the only path that supports `--copies`,
  `--half-cut`, `--chain`, `--mirror`, `--special-tape` and hi-res.
- **AirPrint through CUPS (`--no-high-resolution`):** `lpr` to the queue. 180 dpi, no half-cut,
  chain or mirror (the IPP `finishings` enum cannot express them).

## Cuts, chains and the hardware behind them

The full cutter sits about 23 mm downstream of the print head, so a full cut at any position
means feeding that much tape past it. That one fact explains the rest:

- **Every job starts with a leader feed** of about 23 mm, cut off as waste. Batch to pay it once.
- **Between pages of one job** the printer cannot reach the boundary with the full cutter, so it
  half-cuts there (`--copies` and `--label` both get half-cuts whether or not `--half-cut` is
  passed; it is not yet verified whether the flag can turn them off).
- **24.5 mm is the minimum label length** (Raster Command Reference v1.02, page 18). Shorter
  content is padded, and the padding lands inconsistently at the front or back of each copy.
- **`--chain` produces no cut at all** between jobs: it suppresses the end-of-job feed and cut,
  so consecutive chained jobs butt together and need scissors. It refuses unless `--no-cut-ok` is
  also passed. Every job in a chain except the last must pass `--chain` (each invocation opens a
  new connection, which resets the mode bits). For separate labels on one strip, use `--label` or
  `--image` several times: one job, half-cuts between. An uncut seam is remembered
  (`~/.cache/ptouch-label/chain-open-<model>.txt`) and the next job's `Cuts:` line says it JOINS
  the previous job.
- **Every job prints a `Cuts:` line** stating the physical result (half-cut between N labels;
  full cut, NO cut or none at the end). It is the only thing to report about cuts.

Raster mode bits, for the curious (ESC i K, reference page 33): bit 2 half cut, bit 3 *no*
chain (so `--chain` clears it), bit 4 special tape no-cut, bit 6 high resolution. Mirror is a
separate command (ESC i M, bit 7).

## What to print for each kind of object

The object decides the tape (`--for`), the line count and the length. Defaults below; the
owner's own labeling convention and intake procedure, if there are any, win. Order for new gear:
photograph the stickers, record in the inventory, then print.

| Object | Labels | Tape | Notes |
|---|---|---|---|
| **New machine** (mini PC, desktop, NAS) | 1. name (one line); 2. facts (3 lines: name + role, model + CPU + RAM + disk, address + access); 3. power brick (`<name> power brick`, one line) | `--for mini-pc`: name ≤ 12 mm, facts ≤ 18 mm; `--for tower`/`nas`: ≤ 18 / 24 mm; brick: `--for power-brick`, 9–12 mm | Measure the clear face and pass `--fit`. A one-word name on wide tape is the classic mistake. |
| **Internal SSD or boot disk** (M.2, 2.5") | one line: `<host> boot disk · <OS>` | 9 mm (an M.2 2280 is 22 mm wide); `--for peripheral` as the cap | Beside the maker's sticker, never over the serial or the part number. The disk carries the host's identity across board swaps, so it gets the host's name. |
| **Retired or parts unit** | one 3-line label: `ex-<name> · <model> · PARTS ONLY` / the fault / what still works + retired date | the unit's form factor | Remove or strike the old name label so nobody plugs it in as the live machine. |
| **RAM module** | none | | The module's own sticker (part, size, voltage, serial) is the record; slots are tight and some stickers void the warranty if removed. Photograph it and record the module from the host (`dmidecode` / `system_profiler`) in the inventory. |
| **External drive, cable, power adapter** | see `drive-labeling`, `cable-labeling`, and the owner's convention | | |

## Printing from any macOS print dialog (optional)

The CLI repo ships an IPP Everywhere front end (`contrib/print-dialog/`): a CUPS queue that
accepts any app's print job, probes the tape, rasterises at 360 dpi and prints through this CLI,
trimming and scaling the content down to the loaded cassette. For a person who wants layout
(several type sizes, logos, barcodes) that or Brother's P-touch Editor is the better route; this
CLI is the scripted and agent route. Setup is in the CLI README. If that queue reports "paused",
`cupsenable <queue>`.

## Tape specifications

PT-P750W (180 dpi native; values deliberately conservative against Brother's spec):

| Tape | Printable | Pixels | Lines |
|---|---|---|---|
| 24 mm | 18.0 mm | 128 px | 3 to 4 |
| 18 mm | 12.0 mm | 85 px | 2 to 3 |
| 12 mm | 8.0 mm | 57 px | 1 to 2 |
| 9 mm | 6.9 mm | 49 px | 1 |
| 6 mm | 3.9 mm | 28 px | 1 |

PT-P950NW (360 dpi native; straight from Brother's pin tables):

| Tape | Printable | Pixels | Lines |
|---|---|---|---|
| 36 mm | 32.0 mm | 454 px | 5 to 7 |
| 24 mm | 22.6 mm | 320 px | 4 to 5 |
| 18 mm | 16.5 mm | 234 px | 3 to 4 |
| 12 mm | 10.6 mm | 150 px | 2 to 3 |
| 9 mm | 7.5 mm | 106 px | 1 to 2 |
| 6 mm | 4.5 mm | 64 px | 1 |
| 3.5 mm (`--tape 4`) | 3.4 mm | 48 px | 1 |

Brother sells "3.5 mm" tape; the library calls it `Tape3_5mm` with `width_mm = 4`, so the CLI
takes `--tape 4`.

## How the tape probe works

1. Reads `device-uri` from `lpoptions -p <queue>` to get the printer's Bonjour name, resolves it
   with `dns-sd -L`, and caches the host under `~/.cache/ptouch-label/`.
2. Runs `ipptool` against the printer's `/ipp/print` endpoint and reads
   `media-col-ready` `x-dimension` (hundredths of a mm; `900` is 9 mm), snapping to the nearest
   supported width.

Use `media-ready` / `media-col-ready` (what is loaded), never `media-default` (on the PT-P750W
it always reports the maximum, 24 mm). Query the printer directly; the local CUPS proxy returns
letter-paper defaults. Cold probe about 8 s, warm about 5 s. The PT-P750W's Wi-Fi radio sleeps
deeply: the first probe after sleep often times out and wakes it, the second succeeds, so the
CLI retries once.

**Fail closed.** If the probe still fails, the CLI exits 2 and prints "printer is off or
unreachable", even when `--tape` was given, because a job queued to an off printer prints later
out of context. Recovery: power the printer on (it boots in about 5 s), run `--probe` once or
twice, then print normally. Two real failures behind this rule: three labels printed at 24 mm on
9 mm tape because `--no-probe` was used while the printer was off; and about fourteen labels
ruined because an operator's "it's 9 mm" from earlier in the session was trusted all afternoon
while 24 mm was loaded.

**Jobs spool in CUPS.** If jobs were submitted while the printer was off they still queue
(`lpstat -W not-completed -o <queue>` shows "Looking for printer") and print when it comes on.
Do not resend; `cancel -a <queue>` only if a job is wrong or wedged.

## Vertical calibration (labels off-centre)

The CLI centres content in the printable band exactly; what sits off-centre is the band on the
tape, which is per printer (its tape path), not per width. `ptouch-label --model <M>
--calibrate-vertical` prints a 45 mm ruler with a centre line, band edges and labelled mm ticks;
the tick on the tape's physical middle is the number to store as `vertical_offset_mm` for that
model (positive = toward higher head pins). Measure at offset 0, verify the centre line at the
stored offset, and check at least one other width. One PT-P950NW measured +1.45 mm; expect
values of that order.

## Troubleshooting

**"No route to host" from Python while `curl` and `ping` reach the printer (macOS).** macOS
Local Network privacy blocks non-platform binaries from the Mac's directly attached subnet
unless the responsible app has the Local Network grant; Apple's own `curl`, `nc`, `ping` and
`cupsd` are exempt, and routed subnets are exempt. Fixes, in order: grant Local Network to the
terminal or agent app that runs the CLI; put the printer on a routed VLAN; or set a `proxy_host`
in the config to a small TCP forwarder (socat) on a routed host. The PT-P950NW does not answer
the tape-width query over port 9100 at all, so the CLI scrapes its web status page through
`curl` instead; expect "detected tape width: N mm (EWS scrape via curl)".

**PT-P950NW unreachable, no ARP entry, probe fails: check the Wi-Fi lamp.** On this model
turning Wi-Fi on disables the wired Ethernet link; the two are mutually exclusive, and the Wi-Fi
button sits next to the power button. A sleeping printer still answers ping and keeps an ARP
entry; a missing entry means it is off the wire. Press the Wi-Fi button until the lamp is dark.
A cable reseat does not help. Another Brother printer on the same subnet makes a good control.

**Descenders clipped (g, p, y, J).** Check the generated PNG for asymmetric vertical margins:

```sh
python3 -c "from PIL import Image; im=Image.open('/tmp/ptouch-preview.png').convert('L'); H=im.height; \
  rows=[y for y in range(H) if min(im.getpixel((x,y)) for x in range(im.width))<100]; \
  print(f'top={rows[0]}, bottom={H-1-rows[-1]}')"
```

Symmetric within a pixel or two means the maths is right. The CLI subtracts the first line's
`textbbox` top offset when centring (Pillow's default anchor puts the ascender line at `y`, and
bold Helvetica Neue reports a 9 px offset at size 34); do not "simplify" that away.

**Raster connect or timeout errors.** Run `--probe` first (port 9100 does not wake the radio the
way IPP does); check `nc -z -G 2 <host> 9100`; if the printer changed address, `--probe` refreshes
the cache. `ModuleNotFoundError: usb` means the `pyusb` extra is missing: reinstall with it.

**Half-cut comes out as a full cut.** The half-cut blade is separate and dulls faster; try a
fresh cartridge, then service.

**`--chain` but the next job still feeds a leader.** Every job in the chain except the last must
pass `--chain`; a missed one closed the chain and the leader already fed.

**Orientation wrong (AirPrint path).** The image is rendered landscape and rotated 90 degrees
clockwise to match the portrait custom page size; if output is sideways, the rotation direction
in `print_label()` needs the other transpose.

**Never verify a queue with `lp -d <queue> file.txt`.** The AirPrint PPD rasterises a whole page
and the roll feeds until it finishes, wasting a long strip. Prove reachability with `ipptool`.

**CUPS transports differ by PPD, not by model family.** A queue with the AirPrint PPD wants
`ipp://host:631/ipp/print`; a queue with Brother's native driver wants `socket://host:9100`.

## Device settings through the printer's web interface

The embedded web server exposes a subset of device settings, including **Auto Power Off**, from
any browser on the LAN, with no Windows machine or USB cable:

| Page | URL |
|---|---|
| Status, and the login form | `http://<printer>/general/status.html` |
| Auto Power Off | `http://<printer>/printer/power_settings.html` |
| Network | `http://<printer>/net/net/net.html` |
| Admin password | `http://<printer>/admin/password.html` |

The pre-login menu shows only Status, Refresh, Maintenance, Find and Contact; the Printer and
Network menus appear after login, so do not conclude a setting is missing from the pre-login
view. The default password is on a sticker labelled `Pwd` on the printer body (back, bottom,
battery cavity or cassette compartment); it is neither `access` (older models) nor the
Wireless Direct key on the side sticker. Auto Power Off defaults to 1 hour on AC; `None` suits a
desk-fixture label maker. Factory reset without software: printer off, hold Power, press Feed &
Cut twice (network and password only) or six times (everything, including calibration and auto
off), release.

## Raw lpr reference

```bash
lpr -P <queue> -o PageSize=Custom.24x50mm -o fit-to-page \
    -o page-left=0 -o page-right=0 -o page-top=0 -o page-bottom=0 /path/to/image.png
# portrait image: width = printable pixels of the tape, height = label length, 180 dpi
```

CUPS options on the AirPrint PPD: `PageSize` (Letter or `Custom.WxH`), `CutMedia` (None,
EndOfPage, EndOfJob), `ColorModel` Gray, `MediaType` roll.

## Upstream

The raster protocol comes from [`nbuchwitz/ptouch`](https://github.com/nbuchwitz/ptouch)
(LGPL-2.1, on PyPI). The CLI subclasses its printer classes to thread mirror, chain and
special-tape through; [issue #20](https://github.com/nbuchwitz/ptouch/issues/20) and PRs #21 and
#22 propose those upstream. Brother's reference is the PT-E550W/P750W/P710BT Raster Command
Reference v1.02 (page numbers cited above).
