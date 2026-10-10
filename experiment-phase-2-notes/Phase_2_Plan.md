# Phase 2 Plan: What research and tools have tried

Date: Oct 10, 2026

### Story So Far...

Postgres on its own can keep the old shape over new storage, but the translation leaks - indexes, upserts, prepared statements, lock queues, take your pick. I'm clearly not the first person to hit this. People have been writing papers about changing schemas while the database is running for well over a decade, and some of them built actual systems. So before I try to be clever myself, I should probably find out what they already tried.

### The plan

1. **Read the literature.** Get a map of the main approaches to changing a schema while it's in use. Also work out why industry mostly hasn't picked them up. If the papers solved this years ago, why am I still writing expand/contract by hand?
2. **Rebuild each paper's core idea.** For each one (PRISM, InVerDa, QuantumDB, F1, BullFrog, Tesseract, KVolve and the Lawrence & Adachi study) I'll rebuild the main mechanism in my sandbox and then try to break it. My versions will be simplified imitations, not the authors' code. One note per paper.
3. **Check what's actually runnable.** Which of these have code I can build and run today, and which only exist as a PDF and a nice diagram?
