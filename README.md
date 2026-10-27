# hfd

A single-file Bash CLI for downloading models and datasets from Hugging Face, with support for
multi-threaded downloads, file filtering, gated/private repos, and proxies.

Downloads are driven by [aria2c](https://aria2.github.io/) by default (or `wget`), with automatic
resume, so re-running an interrupted command picks up where it left off.

## Features

- **Fast downloads** — multi-connection, multi-file parallel downloads via `aria2c`.
- **Selective downloads** — `--include` / `--exclude` with wildcard patterns.
- **Gated & private repos** — authenticate with a Hugging Face token.
- **Datasets** — `--dataset` switches to the dataset API and path layout.
- **Proxy support** — `--proxy` applies to metadata fetch *and* file downloads.
- **Resumable** — interrupted downloads continue from the same command.
- **Zero dependencies beyond the shell** — no Python, no `huggingface_hub`, no virtualenv.
- **Mirror friendly** — honors the `HF_ENDPOINT` environment variable.

## Requirements

| Tool | Required | Notes |
| --- | --- | --- |
| `bash` | yes | Any Bash 3.2+ (macOS default works) |
| `curl` | yes | Used to fetch repo metadata |
| `aria2c` | yes\* | Default download tool |
| `wget` | yes\* | Alternative download tool (`--tool wget`) |
| `jq` | recommended | Much faster metadata parsing; a slower `grep`/`awk` fallback is used when absent |

\* Only the selected tool is required (`aria2c` unless `--tool wget` is passed).

Install on macOS:

```bash
brew install aria2 jq
```

Install on Debian/Ubuntu:

```bash
sudo apt-get install -y aria2 jq curl wget
```

## Installation

```bash
git clone <this-repo> hfd-cli
cd hfd-cli
chmod +x hfd.sh

# optional: put it on your PATH
sudo ln -s "$(pwd)/hfd.sh" /usr/local/bin/hfd
```

The script is self-contained — copying `hfd.sh` anywhere is enough.

## Usage

```
hfd <REPO_ID> [options]
```

### Arguments

| Argument | Description |
| --- | --- |
| `REPO_ID` | **Required.** The Hugging Face repo ID, e.g. `gpt2` or `org_name/repo_name`. |

### Options

| Option | Default | Description |
| --- | --- | --- |
| `--include <pattern...>` | – | Only download files matching these patterns. |
| `--exclude <pattern...>` | – | Skip files matching these patterns. |
| `--hf_username <username>` | – | Hugging Face username (not email). |
| `--hf_token <token>` | – | Hugging Face access token. |
| `--tool <aria2c\|wget>` | `aria2c` | Download tool to use. |
| `-x <n>` | `4` | Number of download threads per file for `aria2c` (1–10). |
| `-j <n>` | `5` | Number of concurrent file downloads for `aria2c` (1–10). |
| `--dataset` | off | Treat the repo as a dataset. |
| `--local-dir <path>` | `<repo_name>` | Directory to store the download in. |
| `--revision <rev>` | `main` | Git revision (branch, tag, or commit SHA) to download. |
| `--proxy <url>` | *(none)* | Proxy server for all network requests. |

### Environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `HF_ENDPOINT` | `https://huggingface.co` | API/download endpoint. Point at a mirror to use one. |

## Examples

Download a model:

```bash
hfd gpt2
```

Download everything except the safetensors weights:

```bash
hfd bigscience/bloom-560m --exclude "*.safetensors" "*.md"
```

Download only the tokenizer files:

```bash
hfd meta-llama/Llama-2-7b --include "tokenizer*"
```

Download a gated repo (create a token at
[huggingface.co/settings/tokens](https://huggingface.co/settings/tokens)):

```bash
hfd meta-llama/Llama-2-7b --hf_username myuser --hf_token hf_xxx
```

Download a dataset:

```bash
hfd lavita/medical-qa-shared-task-v1-toy --dataset
```

Download a specific revision:

```bash
hfd bartowski/Phi-3.5-mini-instruct-exl2 --revision 5_0
```

Download through a proxy:

```bash
hfd bigscience/bloom-560m --proxy http://127.0.0.1:7890
```

Download through a SOCKS5 proxy, with `wget` instead of `aria2c`:

```bash
hfd bigscience/bloom-560m --tool wget --proxy socks5://127.0.0.1:1080
```

Use a mirror endpoint and a custom output directory:

```bash
HF_ENDPOINT=https://hf-mirror.com hfd bigscience/bloom-560m --local-dir /data/models
```

## Filtering patterns

`--include` and `--exclude` accept multiple space-separated patterns. Patterns are matched against
the **full file path** inside the repo (the `rfilename` in the repo metadata), and `*` acts as a
wildcard for any sequence of characters.

```bash
# everything under the vae/ directory
hfd org/repo --include "vae/*"

# every .safetensors file except the 4-bit ones
hfd org/repo --include "*.safetensors" --exclude "*4bit*"

# both at once: exclude rules win over include rules
hfd org/repo --include "*.bin" --exclude "*.pth"
```

Quote patterns so the shell does not expand them before the script sees them.

> **Note** — matching is *unanchored*: a pattern matches if it appears anywhere in the path.
> `--include "vae/*"` therefore also matches `subdir/vae/encoder.safetensors`, and
> `--include "*.bin"` also matches `weights.bin.bak`. Anchor the pattern yourself (e.g.
> `--include "^vae/*"`) if you need an exact match. Note that a literal `.` in a pattern is
> escaped, so write `^vae/*` rather than `^vae/.*`.

## Proxies

`--proxy` takes a full proxy URL, with an optional scheme:

```bash
hfd org/repo --proxy http://127.0.0.1:7890
hfd org/repo --proxy socks5://127.0.0.1:1080
hfd org/repo --proxy http://user:password@proxy.example.com:8080
```

The proxy is applied consistently across the whole run:

- **Metadata** — fetched with `curl --proxy`.
- **aria2c** — written into the generated URL list as a per-URL `all-proxy=` option *and* passed
  globally as `--all-proxy`, so resumed sessions keep using it.
- **wget** — set via `use_proxy=yes` together with `http_proxy` / `https_proxy`.

Without `--proxy`, no proxy is used at all and the behavior is identical to a direct connection.

## How it works

1. **Fetch metadata** — `GET {HF_ENDPOINT}/api/models/{REPO_ID}` is requested with `curl` and cached
   to `.hfd/repo_metadata.json`.
2. **Check access** — if the repo is gated, `--hf_username` and `--hf_token` are required.
3. **Build the file list** — the metadata is filtered by `--include` / `--exclude` and written to
   `.hfd/aria2c_urls.txt` (or `.hfd/wget_urls.txt`). For `aria2c` each entry carries its target
   directory, output filename, auth header, and proxy.
4. **Download** — `aria2c` / `wget` reads that list and downloads in parallel.

### Files created

Everything lives under a `.hfd` directory inside the local download directory:

| File | Purpose |
| --- | --- |
| `.hfd/repo_metadata.json` | Cached repo metadata (avoids re-fetching). |
| `.hfd/aria2c_urls.txt` / `.hfd/wget_urls.txt` | Generated download list. |
| `.hfd/last_download_command` | The options used last time, used to detect changes. |

### Resume behavior

- If the generated file list is missing, or any option changed since the last run (including
  `--proxy`, `--include`, `--exclude`, `--revision`, …), the list is regenerated automatically.
- If nothing changed, the existing list is reused and `aria2c` resumes the interrupted transfers.

Delete `.hfd/` to force a clean re-fetch and re-download.

## Troubleshooting

**`aria2c is not installed`** — install `aria2` (`brew install aria2` / `apt-get install aria2`), or
use `--tool wget`.

**`jq not installed` warning** — harmless; the script falls back to a slower `grep`/`awk` parser.
Installing `jq` is recommended for large repos.

**`The repository requires authentication`** — the repo is gated. Create a token at
[huggingface.co/settings/tokens](https://huggingface.co/settings/tokens) and pass both
`--hf_username` and `--hf_token`.

**`Failed to fetch metadata`** — check network connectivity, the repo ID spelling, and whether the
repo is private. If you are behind a proxy, add `--proxy <url>`.

**Download is slow or stalls** — try increasing parallelism (`-x 8 -j 8`), or switch to
`--tool wget` if `aria2c` misbehaves behind your proxy.

## License

MIT
