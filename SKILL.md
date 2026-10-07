---
name: png2ico
description: Convert PNG images into Windows .ico icon files (favicon.ico, .exe/app icons, installer and shortcut icons) containing several resolutions, using the bundled png2ico binary — no ImageMagick or Pillow needed. Use this whenever the user wants an .ico made from a PNG or logo, e.g. "make a favicon.ico from logo.png", "I need an icon for my Windows/Electron/Tauri/PyInstaller build", "turn these PNGs into an ico", or when a build config references a missing .ico and a PNG logo exists, even if they never say "ico".
allowed-tools: Bash(${CLAUDE_SKILL_DIR}/scripts/png2ico *)
---

# png2ico

Convert one or more PNGs into a single multi-resolution `.ico` with the bundled binary. It handles resizing, transparency and the ICO format itself, so there's no need to install other image tools or write conversion code.

## Command

```bash
${CLAUDE_SKILL_DIR}/scripts/png2ico [flags] <input.png> [output.ico]
${CLAUDE_SKILL_DIR}/scripts/png2ico [flags] -o <output.ico> <input.png>...
```

The script picks the right binary for the current OS and CPU from `${CLAUDE_SKILL_DIR}/bin/`. On Windows without a POSIX shell, run `${CLAUDE_SKILL_DIR}/bin/png2ico-windows-amd64.exe` (or `-arm64.exe`) directly with the same arguments.

| Flag | Meaning |
|---|---|
| `-o <path>` | Output file. Defaults to the input's name with `.ico`. A trailing `*.ico` argument also sets it. |
| `-s <sizes>` | Comma-separated sizes in px, each 1–256. Default `16,24,32,48,64,128,256`. |
| `-bmp` | Store sizes below 256 as uncompressed BMP instead of PNG (legacy readers only, see below). |
| `-f` | Overwrite the output if it already exists. Without it, an existing file is an error. |
| `-q` | Skip the summary line. |

Flags may come before or after the file names. Exit codes: `0` success, `1` conversion error (unreadable or non-PNG input, existing output, write failure), `2` bad arguments. Errors go to stderr, prefixed with `png2ico:`.

On success it prints one line you can relay to the user:

```
wrote assets/app.ico: 7 images (16, 24, 32, 48, 64, 128, 256 px), 45.1 KB
```

## Choosing sizes

Pick the sizes from what the icon is for. Each extra size makes the file bigger, so a favicon shouldn't carry a 256px image nobody will use.

| Use | `-s` |
|---|---|
| Website `favicon.ico` | `16,32,48` |
| Windows app, `.exe`, installer, Electron/Tauri/PyInstaller/NSIS build | default (omit `-s`) |
| Windows shortcut, folder or file-type icon | default |
| The user names specific sizes | exactly those |

## Workflow

1. **Make sure the input really is a PNG.** The tool only decodes PNG, and a `.png` extension isn't proof. Check with `file <input>` if unsure. For JPEG, WebP or other formats, convert to PNG first (on macOS: `sips -s format png in.jpg --out in.png`). For SVG, render it to a PNG at least 256px wide (for example `rsvg-convert -w 512 in.svg -o in.png`, or `qlmanage -t -s 512 -o . in.svg` on macOS).

2. **Check the source resolution.** The best source is square and at least as large as the largest size requested (256px for the default set). A smaller source still works, but the larger sizes are upscaled and come out blurry. The tool prints a warning naming them. When that happens, tell the user, and either ask for a larger source or drop the upscaled sizes with `-s`.

3. **Non-square images are centered on a transparent square canvas, not stretched.** That's usually right for logos. If the user would rather crop, crop first.

4. **Run the command.** Use `-f` only when the user asked to replace the existing file, or when the existing file is clearly an earlier output of this same task. Otherwise, the "already exists" error is a cue to ask the user or pick another name. Overwriting someone's hand-made icon can't be undone.

5. **Report** the output path, the sizes, the file size and any warnings. For a website, remind the user that browsers request `/favicon.ico` from the site root by default; otherwise, the page needs `<link rel="icon" href="/path/to/favicon.ico">`.

## Several PNGs → one icon

If the user has separate PNGs for different sizes (for example a pixel-tuned `logo-16.png` plus a large `logo-512.png`), pass them all with `-o`:

```bash
${CLAUDE_SKILL_DIR}/scripts/png2ico -o app.ico logo-16.png logo-32.png logo-512.png
```

For each size, the tool uses the smallest input that is at least that large. A PNG that already matches a size exactly is used pixel-for-pixel. This beats downscaling one big image, because small icons are often hand-adjusted to stay crisp. To make one `.ico` per PNG instead, run the command once per file.

## When to use `-bmp`

The default stores every size as PNG inside the ICO. PNG entries are compact and supported by Windows Vista and later, all browsers, and current build tools. Use `-bmp` only if the user targets Windows XP-era software or a tool that rejects PNG-compressed icons, such as some old resource compilers and icon libraries. The 256px entry stays PNG either way, because a 256px BMP would add about 256 KB.

## Troubleshooting

- **`not a valid PNG`**: the file is some other format with a `.png` name. Convert it as in step 1.
- **`no bundled binary for <os>-<arch>`**: the skill ships binaries for macOS, Linux and Windows on x86-64 and ARM64 only. Tell the user which platform is missing, and ask before falling back to another tool such as ImageMagick.
- **`already exists (use -f to overwrite)`**: see step 4.
- **`would overwrite an input`**: the output path points at one of the source PNGs. Choose a different `-o`.
