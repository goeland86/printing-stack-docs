# 3D Printing Stack Documentation

Source for the documentation site published at
**<https://goeland86.github.io/printing-stack-docs/>**.

Covers the goeland86 3D printing fleet end-to-end:

- **StealthChanger Voron 2.4** with 5 toolheads, Recore A7 host
- **Snapmaker J1S** wrapped in a Klipper-style frontend via
  [`snapmaker_moonraker`](https://github.com/goeland86/snapmaker_moonraker)
- **NFC spool selection** across the fleet via
  [`klipper-nfc-daemon`](https://github.com/goeland86/klipper-nfc-daemon)
  and the [`goeland86/Spoolman`](https://github.com/goeland86/Spoolman)
  `pr/nfc-support` fork
- The glue between them: bridge intercepts, save_variables, prompt
  dialogs, multi-tool assignments

## Building locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Open <http://127.0.0.1:8000>.

## Publishing

The `.github/workflows/docs.yml` workflow builds with `mkdocs build --strict`
and deploys to GitHub Pages on every push to `main`.

Make sure GitHub Pages is configured to deploy from **GitHub Actions** in
the repo's *Settings → Pages*.

## License

MIT.
