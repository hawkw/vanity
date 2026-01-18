# Design Process Rules for Vanity

## Incremental Design

When working on design documents:
- Write one section at a time, not entire documents in one pass
- Discuss each section with the user before moving on
- Don't write code samples in design documents — focus on high-level concepts

## Research Documentation

When you do research (web searches, documentation fetches):
- Capture findings in a dated note in `notes/`
- Include the context: why were you researching this?
- Include sources with URLs where applicable
- This prevents future sessions from repeating the same research

## Design Decisions

When making or documenting design decisions:
- Explain the rationale, not just the decision
- Note alternatives that were considered and why they were rejected
- If something is an open question, mark it clearly as such

## MDBook Documentation

The design docs use MDBook (`docs/` directory):
- Use draft chapters `- [Chapter Name]()` for unwritten sections
- Don't create placeholder files — let MDBook show them as disabled links
- Keep the SUMMARY.md in sync with actual file structure

## Skepticism

Before committing to a design:
- Enumerate the advantages and disadvantages
- Consider whether it actually fits the stated goals
- Ask: what could go wrong? What are we giving up?
- It's okay to be wrong, but be thoughtfully wrong
