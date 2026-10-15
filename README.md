# ptouch-labels: a Claude Code plugin

One skill that lets Claude print tape labels on Brother P-touch network label printers
(PT-P750W, PT-P950NW) through the [`ptouch-label`](https://github.com/johntrandall/ptouch-label)
CLI, and teaches it the things that waste tape when you get them wrong: probe the loaded tape
width before every job, size the label to the object, preview first, batch identical labels in
one feed, put half-cuts between labels, and read the printer's quirks (sleeping radios,
mutually exclusive Wi-Fi and Ethernet on the PT-P950NW, descender clipping).

## Install

```
/plugin marketplace add johntrandall/claude-plugins
/plugin install ptouch-labels@johntrandall-plugins
```

## Requirements

- macOS with CUPS, a Brother PT-P750W or PT-P950NW on the network with TZe tape.
- The CLI: `pipx install git+https://github.com/johntrandall/ptouch-label` (or `uv tool install`
  the same URL). Its config file names your printers; the skill shows the shape.

## Licence

MIT, see [LICENSE](LICENSE). The skill cites Brother's Raster Command Reference by page number
and the `nbuchwitz/ptouch` library (LGPL-2.1, installed as a dependency, not bundled).
