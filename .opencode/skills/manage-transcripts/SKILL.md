---
name: manage-transcripts
description: |
  Search, summarize, list, and analyze TKC show transcripts from the
  TKC_TRANSCRIPT GitHub repo without cloning. Uses GitHub API and raw
  file access.
---

# Manage Transcripts

Work with transcripts from the [TKC_TRANSCRIPT](https://github.com/ProducerJenn/TKC_TRANSCRIPT) repo without cloning.

Transcript files are named `<MMDDYY>_TS.md` (e.g. `071026_TS.md` for July 10, 2026).
Date parsing: `^(\d{2})(\d{2})(\d{2})_TS\.md$` → `20YY-MM-DD`.

Two transcript formats exist:
- `(MM:SS)  text` — parenthesized timestamps
- `MM:SS\ntext` — bare timestamps on their own line

## Installing This Skill

### Download

```bash
# Linux / macOS
mkdir -p .opencode/skills/manage-transcripts
curl -sL "https://raw.githubusercontent.com/ProducerJenn/TKC_TRANSCRIPT/main/.opencode/skills/manage-transcripts/SKILL.md" \
  -o .opencode/skills/manage-transcripts/SKILL.md
```

```powershell
# Windows (PowerShell)
New-Item -ItemType Directory -Force -Path ".opencode/skills/manage-transcripts"
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/ProducerJenn/TKC/main/.opencode/skills/manage-transcripts/SKILL.md" `
  -OutFile ".opencode/skills/manage-transcripts/SKILL.md"
```

The skill is now available to your opencode agent. Reload or start a new session to pick it up.

### Prerequisites

| Tool | Linux | Windows |
|------|-------|---------|
| `curl` | `sudo apt install curl` (Debian) / `sudo dnf install curl` (Fedora) | Included in PowerShell (`Invoke-WebRequest`) |
| `jq` | `sudo apt install jq` | Download from [stedolan.github.io/jq](https://stedolan.github.io/jq/download/) or `winget install jqlang.jq` |

If using opencode on Windows, run commands in **Git Bash** or **WSL** for full compatibility.

### Verify

```bash
# Linux / macOS
curl -s "https://api.github.com/repos/ProducerJenn/TKC_TRANSCRIPT/contents" | jq '. | length'
```

```powershell
# Windows PowerShell
$resp = Invoke-WebRequest "https://api.github.com/repos/ProducerJenn/TKC_TRANSCRIPT/contents" | ConvertFrom-Json
$resp.Count
```

Should return the number of transcript files (20+).

### Variables

Set these at the top of any session:

```bash
REPO="ProducerJenn/TKC_TRANSCRIPT"
BRANCH="main"
BASE="https://raw.githubusercontent.com/$REPO/$BRANCH"
API="https://api.github.com/repos/$REPO/contents"
```

## Operations

### List available transcripts

```bash
curl -s "$API" | jq -r '.[].name' | grep '_TS.md' | sort
```

For each file with parsed date:
```bash
curl -s "$API" | jq -r '.[].name' | grep '_TS.md' | sort | while read f; do
  name=$(basename "$f" _TS.md)
  if [[ "$name" =~ ^([0-9]{2})([0-9]{2})([0-9]{2})$ ]]; then
    echo "20${BASH_REMATCH[3]}-${BASH_REMATCH[1]}-${BASH_REMATCH[2]}  ($f)"
  else
    echo "$name  ($f)"
  fi
done
```

### Search transcripts (fast)

Downloads each file individually, searches, outputs **filename + timestamp + text**:

```bash
curl -s "$API" | jq -r '.[].name' | grep '_TS.md' | sort | while read f; do
  content=$(curl -s "$BASE/$f")
  echo "$content" | while IFS= read -r line; do
    if echo "$line" | grep -qi "search term"; then
      ts=$(echo "$line" | grep -oP '^\(?\d+:\d+' || echo "??:??")
      echo "[$f @ $ts]  $line"
    fi
  done
done
```

Or a single-pass approach using temp storage (faster for multiple searches):

```bash
# Fetch and cache all transcripts
mkdir -p /tmp/tkc_transcripts
for f in $(curl -s "$API" | jq -r '.[].name' | grep '_TS.md'); do
  curl -s "$BASE/$f" -o "/tmp/tkc_transcripts/$f"
done

# Now search locally
grep -rn "/tmp/tkc_transcripts" -e "search term" --include="*.md" | while IFS=: read file line text; do
  ts=$(echo "$text" | grep -oP '^\(?\d+:\d+' || echo "??:??")
  name=$(basename "$file")
  echo "[$name @ $ts]  $text"
done
```

### Read a specific transcript

```bash
curl -s "$BASE/071026_TS.md"
```

### Summarize a transcript

1. Fetch the file: `curl -s "$BASE/<filename>"`
2. Read the full content
3. Write a single paragraph summary:
   - Opening context (show date, host, main subject)
   - Key claims with neutral attribution ("stated," "claimed," "argued")
   - No editorializing or external knowledge

### Get transcript stats

```bash
echo "=== Transcript Stats ==="
files=$(curl -s "$API" | jq -r '.[].name' | grep '_TS.md' | sort)
total=$(echo "$files" | wc -l)
echo "Total transcripts: $total"
first=$(echo "$files" | head -1 | sed 's/_TS.md//')
last=$(echo "$files" | tail -1 | sed 's/_TS.md//')
echo "First: $first"
echo "Latest: $last"
echo ""
echo "Word counts:"
for f in $files; do
  wc=$(curl -s "$BASE/$f" | wc -w)
  echo "  $f: $wc words"
done
```

## Example Output

```
[071026_TS.md @ 00:00]  Good evening and welcome to the Tony Kiddcast here on the Daily Signal...
[071026_TS.md @ 01:01]  ship full of oil to be stranded. In the meantime, after a lot of back...
[070926_TS.md @ 00:00]  Let's have a lot of breaking news to dive into, and this is a major...
```

## Notes

- No git clone needed — all operations use `curl` + GitHub raw file API
- For repeated searches, cache to `/tmp/tkc_transcripts/` to avoid re-downloading
- Rate limit: GitHub API allows 60 unauthenticated requests/hour (enough for list + individual fetches)
- GitHub Actions workflow at `.github/workflows/youtube-transcript.yml` for adding new transcripts
