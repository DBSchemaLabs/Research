# Phase 2 Plan: What research and tools have tried

Date: Oct 10, 2026

### Story So Far...

Postgres on its own can keep the old shape over new storage, but the translation leaks - indexes, upserts, prepared statements, lock queues, take your pick. I'm clearly not the first person to hit this. People have been writing papers about changing schemas while the database is running for well over a decade, and some of them built actual systems. So before I try to be clever myself, I should probably find out what they already tried.

### The plan

Rebuilding every paper in my sandbox would be a lot of work and a lot of bugs, most of them mine. So first I'll just read, and keep track of what I don't understand or what the paper doesn't cover.

1. **Read the papers, one at a time.** PRISM, InVerDa, QuantumDB, F1, BullFrog, Tesseract, KVolve, and the Lawrence & Adachi study. Notes go section by section, skipping the history and related-work parts, then a summary of what the paper claims, then my doubts about what it doesn't cover. One commit per paper.
2. **Answer my own doubts.** Once all of them are read, go back and answer each doubt as best I can. Some will be answered by a later paper, and some will turn out to be obvious.
3. **Compile what's left.** Everything still unanswered goes into one list of gaps. That list decides Phase 3: which ones need an experiment, and which I can just decide.

Note template: [TEMPLATE.md](TEMPLATE.md)
