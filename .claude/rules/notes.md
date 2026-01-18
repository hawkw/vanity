# Notes

Structure:
- docs/notes/tldr/ — token-efficient summaries for Vesper (~10 lines, minimal formatting)
- docs/notes/tldr/INDEX.md — one-line description of each note
- docs/notes/ — full details for Eliza (with sources, context)

Reading (token discipline):
1. Read tldr/INDEX.md to see what exists
2. Read only relevant tldrs
3. Load full notes only for deep-dives

Writing:
1. Full note in docs/notes/<name>.md with date, author, context, sources
2. Matching tldr in docs/notes/tldr/<name>.md (same filename)
3. Add one-line entry to INDEX.md

TLDR format: no markdown headers/bullets/bold, lowercase, ~10 lines max, end with "full details: ../<name>.md"
