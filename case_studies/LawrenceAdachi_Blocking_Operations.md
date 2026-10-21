# Case study - Lawrence & Adachi: Assessing the Occurrence of Blocking Operations in Database Schema Evolution: A Case Study (VEM, 2021)

Lawrence, Adachi (UFRN, Brazil) · [PDF](https://sol.sbc.org.br/index.php/vem/article/download/17209/17047) · a short workshop paper, five pages

## Notes-

- Real changes are mostly additive. Over 70% added an index, column, foreign key or table, and changing a column's type wasn't among the common ones. 
- Either type changes are rare, or teams avoid them because they hurt. There were also no views or triggers, because Django doesn't create them, which rules out my E3 approach, InVerDa and QuantumDB for ORM-only teams. 
- Blocking is routine but rarely painful. 87.5% of months had at least one blocking operation, but a typical one took about 18.5 ms, and only 4 months (about 5.5%) had enough in a row to plausibly be noticed. What really hurts is a full-table rewrite: adding a column with a default took up to 11 s on a million rows, the same kind of rewrite as my E1. They couldn't confirm any real downtime, because no logs were kept. 
- The team's fix is my E3 recipe by hand: add the column empty, fill it in chunks, use concurrent operations. Django hides the locks, so developers often don't know what they're taking.

