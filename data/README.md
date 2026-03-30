# Data — Automotive Crisis Knowledge Graph

## KGE Split Files (root directory)

| File | Triples | Description |
|---|---|---|
| `train.txt` | ~100,699 | Full training split (80%) |
| `valid.txt` | ~12,587 | Validation split (10%) |
| `test.txt` | ~12,588 | Test split (10%) |
| `train_20k.txt` | 20,000 | 20k-subsample training |
| `valid_20k.txt` | 2,500 | 20k-subsample validation |
| `test_20k.txt` | 2,500 | 20k-subsample test |
| `train_50k.txt` | 50,000 | 50k-subsample training |
| `valid_50k.txt` | 6,250 | 50k-subsample validation |
| `test_50k.txt` | 6,250 | 50k-subsample test |
| `train_full.txt` | same as train.txt | Alias for full split |

Format: tab-separated `subject \t predicate \t object` (URI strings).

## samples/

Contains a 500-triple excerpt of `train_20k.txt` for quick inspection without downloading large files.

## Full KB

The full `automotive_kb.nt` (~125,873 triples, ~70 MB) is stored in the repository root.
If it exceeds GitHub's file size limit, it can be downloaded from the project release assets.
