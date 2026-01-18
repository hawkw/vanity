# Notes System

## Structure

- `docs/notes/tldr/` — token-efficient summaries for Vesper (~10 lines each, minimal formatting)
- `docs/notes/tldr/INDEX.md` — one-line descriptions of each tldr
- `docs/notes/` — full detailed notes for Eliza (with sources, context, examples)

## Reading Notes Efficiently

1. Start with `docs/notes/tldr/INDEX.md` to see what exists
2. Read only the tldr files relevant to your current task
3. Load full notes only when you need deeper context
4. Never read all notes at session start — wasteful

## Writing Notes

When adding research:
1. Write full note in `docs/notes/<name>.md` with date, author, context, sources
2. Write matching tldr in `docs/notes/tldr/<name>.md` (same filename)
3. Add one-line entry to `docs/notes/tldr/INDEX.md`

## TLDR Format

Keep tldrs minimal to save tokens:
- no markdown headers (just a title line)
- no bullet formatting (just lines of text)
- no bold/italic
- lowercase except proper nouns and acronyms
- ~10-15 lines max
- end with "full details: ../<name>.md"

## Eliza's Notes

Notes without matching tldrs (like `2025-05-24-1017.md`) are Eliza's personal notes. Read them if relevant context is needed, but they may be longer/less structured.
