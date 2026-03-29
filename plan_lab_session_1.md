# Plan: Lab Session 1 — Web Mining Pipeline (Automotive News)

**TL;DR**: Pipeline en 2 phases dans `notebook.ipynb` — crawl + nettoyage (`trafilatura`) → `crawler_output.jsonl`, puis extraction d'entités & relations (`spaCy en_core_web_trf`) → `extracted_knowledge.csv`. Les 14 URLs déjà dans le notebook couvrent Porsche, VW, Mercedes, Stellantis, Nissan, Jaguar.

---

## Phase 0 — Setup (Cellule 1)

1. Installer les dépendances (`trafilatura`, `spacy`, `pandas`, `httpx`) et télécharger le modèle `en_core_web_trf`
2. Imports globaux + chargement du modèle spaCy + déclaration des seed URLs

---

## Phase 1 — Crawling & Cleaning (Cellules 2–5)

3. **robots.txt** — `check_robots(url)` avec `urllib.robotparser` : filtrer les URLs interdites avant tout fetch
4. **Fetch + extraction** — `fetch_and_extract(url)` : `httpx.get()` → `trafilatura.extract()` pour isoler le contenu principal
5. **Filtre qualité** — `is_useful(text, min_words=500)` → écarter les pages trop courtes
6. **Sauvegarde JSONL** — `run_crawler(urls)` : orchestre les étapes 3–5, écriture incrémentale dans `crawler_output.jsonl`
   - Format : `{"url", "domain", "word_count", "text"}`

---

## Phase 2 — Information Extraction (Cellules 6–10)

7. **Chargement** — Lire `crawler_output.jsonl` → liste de dicts *(dépend de l'étape 6)*
8. **NER** — `extract_entities(doc, source_url)` via `nlp.pipe()` — labels ciblés : `ORG`, `PERSON`, `GPE`, `DATE`, `MONEY` — filtrer entités trop courtes ou génériques *(parallèle avec étape 9 en conception)*
9. **Relations (Dependency Parsing)** — `extract_relations(doc, source_url)` : pour chaque verbe, chercher `nsubj` + `dobj`/`attr` qui sont des entités nommées → triples `(subject, verb, object)` *(parallèle avec étape 8)*
10. **Déduplication** — Normaliser (`.strip().title()`), grouper par `(entity, label)`, afficher top 20
11. **Export** — `extracted_knowledge.csv` (colonnes : `entity`, `label`, `frequency`, `source_urls`) + `extracted_relations.csv` (bonus)

---

## Fichiers produits

| Fichier | Contenu |
|---|---|
| `notebook.ipynb` | Pipeline complet |
| `crawler_output.jsonl` | Textes nettoyés + métadonnées |
| `extracted_knowledge.csv` | Entités NER extraites |
| `extracted_relations.csv` | Triples sujet→verbe→objet (bonus) |

---

## Vérification

1. Exécuter toutes les cellules sans erreur
2. `crawler_output.jsonl` : ≥ 8 articles avec `word_count ≥ 500`
3. `extracted_knowledge.csv` : présence de ORG (Porsche, Volkswagen…), GPE, PERSON, DATE
4. Minimum 3 triples extraits avec un verbe connectant deux entités nommées
5. Identifier manuellement 3 cas d'ambiguïté NER pour le rapport (ex: "Maserati" classé PERSON au lieu de ORG ?)

---

## Décisions arrêtées

- **Domaine** : Crise financière industrie automobile (Porsche, VW, Stellantis, Nissan, Mercedes, Jaguar)
- **URLs** : 14 URLs déjà définies — pas de crawl récursif, seed URLs fixes
- **Langue** : Anglais → modèle `en_core_web_trf`
- **robots.txt** : vérifié de manière non bloquante (les sites d'actualité autorisent généralement les crawlers)
