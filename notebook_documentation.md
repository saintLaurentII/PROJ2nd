# Documentation — Lab Session 1 : Web Mining Pipeline

**Cours :** Web Mining & Semantics  
**Domaine ciblé :** Crise financière de l'industrie automobile (Porsche, Volkswagen, Stellantis, Nissan, Mercedes, Jaguar)  
**Fichier :** `notebook.ipynb`

---

## Vue d'ensemble du pipeline

```
Seed URLs (14)
    │
    ▼
[Cellule 3] robots.txt check       ← éthique web
    │
    ▼
[Cellule 5] fetch_and_extract()    ← httpx + trafilatura
    │
    ▼
[Cellule 6] is_useful()            ← filtre ≥ 500 mots
    │
    ▼
[Cellule 7] run_crawler()          → crawler_output.jsonl
    │
    ▼
[Cellule 9] load_jsonl()           ← lecture indépendante
    │
    ▼
[Cellule 10] nlp.pipe()            ← en_core_web_trf (passe unique)
    │         ├── extract_entities()  → all_entities (787 brutes)
    │         └── extract_relations() → all_relations (37 triples)
    │
    ▼
[Cellule 11] normalize_entity()    ← acronymes + alias
    │
    ▼
[Cellule 12] export CSV            → extracted_knowledge.csv
                                   → extracted_relations.csv
```

**Résultats obtenus :** 11/14 URLs retenues · 787 entités brutes · 347 uniques · 37 triples de relations

---

## Phase 0 — Setup

### Cellule 1 — Titre et description du pipeline *(Markdown)*

Cellule d'en-tête, purement documentaire. Elle rappelle le contexte du lab et liste les 3 grandes phases du notebook. Aucun code exécutable.

---

### Cellule 2 — Installation des dépendances *(Python)*

```python
# %pip install trafilatura spacy pandas httpx --quiet
# %run -m spacy download en_core_web_trf
```

**Rôle :** Installer les librairies nécessaires. Les commandes sont commentées car les packages sont déjà installés dans l'environnement `ESILV_A4`. À décommenter lors d'un premier lancement.

| Package | Rôle |
|---|---|
| `httpx` | Client HTTP asynchrone pour récupérer les pages web |
| `trafilatura` | Extraction du contenu principal d'un article (boilerplate removal) |
| `spacy` | Pipeline NLP : tokenisation, POS, NER, dependency parsing |
| `pandas` | Manipulation et export des données tabulaires |
| `en_core_web_trf` | Modèle spaCy transformer (RoBERTa-based), plus précis que `en_core_web_sm` |

---

### Cellule 3 — Imports et chargement du modèle NLP *(Python)*

```python
import json, re, pathlib, urllib.robotparser
from urllib.parse import urlparse
import httpx, trafilatura, spacy, pandas as pd

nlp = spacy.load("en_core_web_trf")
```

**Rôle :** Centraliser tous les imports et charger le modèle spaCy une seule fois. `nlp` est l'objet global qui sera utilisé dans toutes les étapes NLP suivantes.

**Pourquoi `en_core_web_trf` ?**  
C'est le modèle le plus précis de spaCy pour l'anglais. Il utilise un encodeur de type **transformer** (RoBERTa) pour le contexte, ce qui améliore significativement la précision du NER par rapport aux modèles `sm` ou `md` basés sur des vecteurs statiques. Contrepartie : plus lent et consomme plus de RAM.

**Composants activés :** `transformer → tagger → parser → ner → attribute_ruler → lemmatizer`

---

### Cellule 4 — Seed URLs *(Python)*

```python
urls = [
    "https://vision-mobility.de/en/news/profit-slump-at-mercedes-...",
    "https://www.reuters.com/business/autos-transportation/porsche-...",
    ...  # 14 URLs au total
]
```

**Rôle :** Définir les 14 URLs de départ (*seed URLs*). Ce sont des articles de presse spécialisée en anglais, couvrant les difficultés financières de constructeurs automobiles entre 2024 et 2026.

**Sources couvertes :**

| Constructeur | Sources |
|---|---|
| Porsche | Reuters, Newsroom Porsche (×2), Motor1 |
| Volkswagen | The Guardian, Fortune, MotorTrend (×2) |
| Mercedes | Vision Mobility |
| Stellantis | Detroit News, MotorTrend, ItalPassion |
| Nissan | Carscoops |
| Jaguar | The Truth About Cars |
| BMW/Toyota | Motor1 |

**Choix de design :** Pas de crawl récursif — on fixe manuellement les URLs pour garder un contrôle total sur la qualité et la pertinence du corpus.

---

## Phase 1 — Web Crawling & Cleaning

### Cellule 5 — Vérification robots.txt *(Python)*

```python
def check_robots(url: str, user_agent: str = "*") -> bool:
    parsed = urlparse(url)
    robots_url = f"{parsed.scheme}://{parsed.netloc}/robots.txt"
    rp = urllib.robotparser.RobotFileParser()
    rp.set_url(robots_url)
    try:
        rp.read()
    except Exception:
        return True  # si robots.txt inaccessible → on autorise
    return rp.can_fetch(user_agent, url)
```

**Rôle :** Respecter la politique d'exploration d'un site avant tout téléchargement. Le fichier `robots.txt` est un standard web qui liste les chemins autorisés ou interdits aux robots.

**Fonctionnement pas à pas :**
1. Extraire `scheme` + `netloc` depuis l'URL (ex: `https://www.reuters.com`)
2. Construire l'URL du fichier robots : `https://www.reuters.com/robots.txt`
3. Charger et parser ce fichier avec `RobotFileParser` de la bibliothèque standard Python
4. Appeler `can_fetch("*", url)` pour savoir si l'URL cible est autorisée
5. En cas d'erreur réseau sur robots.txt → **politique permissive** : on autorise (mieux que bloquer des URL accessibles)

**Affichage :** La cellule boucle sur toutes les URLs et affiche `✓` ou `✗ BLOCKED` pour chaque URL, permettant un audit visuel rapide.

---

### Cellule 6 — Fetch & Extraction du contenu *(Python)*

```python
HEADERS = {"User-Agent": "Mozilla/5.0 (compatible; WebMiningBot/1.0; ...)"}

def fetch_and_extract(url: str) -> dict | None:
    response = httpx.get(url, headers=HEADERS, timeout=15, follow_redirects=True)
    response.raise_for_status()
    text = trafilatura.extract(response.text, include_comments=False, ...)
    return {"url": url, "domain": ..., "word_count": ..., "text": text}
```

**Rôle :** Télécharger une page HTML et en extraire uniquement le contenu éditorial (le corps de l'article), en supprimant tout le bruit environnant.

**Deux étapes distinctes :**

**1. Fetch avec `httpx`**
- `headers` : un `User-Agent` identifié et honnête (bonne pratique éthique — un bot anonyme ressemble à une attaque)
- `timeout=15` : évite de bloquer indéfiniment sur un serveur lent
- `follow_redirects=True` : gère les redirections HTTP 301/302 automatiquement
- `raise_for_status()` : lève une exception si le code HTTP est ≥ 400 (erreur client/serveur)

**2. Extraction avec `trafilatura`**
`trafilatura` est une librairie spécialisée dans le *boilerplate removal* : il analyse la structure DOM de la page et identifie statistiquement le bloc de texte principal (l'article), en écartant :
- La navigation, header, footer
- Les publicités, pop-ups, cookies
- Les listes de liens, sidebars
- Les commentaires utilisateurs (`include_comments=False`)
- Les tableaux de données (`include_tables=False`)

**Format de retour :**
```python
{
    "url": "https://...",
    "domain": "www.reuters.com",
    "word_count": 1094,
    "text": "Porsche swung to an €11 billion quarterly loss..."
}
```

**Gestion des erreurs :**
- `HTTPStatusError` → log avec le code HTTP (ex: `[HTTP 403]` pour un paywall)
- `Exception` générique → log du message d'erreur
- `trafilatura` retourne `None` → log `[EMPTY]`

Dans tous les cas d'échec, la fonction retourne `None` et le crawler continue avec l'URL suivante.

---

### Cellule 7 — Filtre de qualité *(Python)*

```python
def is_useful(text: str, min_words: int = 500) -> bool:
    return len(text.split()) >= min_words
```

**Rôle :** Écarter les pages qui ne contiennent pas assez de contenu textuel pour être utiles à l'extraction d'entités.

**Pourquoi 500 mots ?**  
Un article de presse substantiel fait typiquement 600–1500 mots. En dessous de 500, le contenu extrait est probablement une page d'erreur, une page d'accueil, ou un article tronqué par un paywall. Le seuil de 500 est un bon compromis entre inclusivité et qualité.

**Note :** Le comptage par `split()` est une heuristique simple mais suffisante pour ce filtrage — il n'est pas nécessaire d'être précis au mot près.

---

### Cellule 8 — Orchestration du crawler + sauvegarde JSONL *(Python)*

```python
OUTPUT_JSONL = pathlib.Path("crawler_output.jsonl")

def run_crawler(url_list, min_words=500):
    for url in url_list:
        if not check_robots(url): continue           # 1. robots check
        doc = fetch_and_extract(url)                  # 2. fetch
        if doc is None: continue
        if not is_useful(doc["text"], min_words): continue  # 3. filtre
        f.write(json.dumps(doc, ensure_ascii=False) + "\n") # 4. save
```

**Rôle :** Orchestrer les étapes 1.1 à 1.3 en un seul appel et sauvegarder les résultats en **JSONL** (JSON Lines).

**Pourquoi JSONL ?**  
Le format JSONL (une ligne JSON par document) est idéal pour les pipelines de données :
- Chaque ligne est un JSON valide indépendant → lecture partielle possible
- Écriture incrémentale : si le script plante à mi-chemin, les documents déjà traités sont sauvegardés
- Compatible avec tous les outils (`pandas.read_json(..., lines=True)`, `jq`, etc.)

**Structure d'une ligne :**
```json
{"url": "https://...", "domain": "www.reuters.com", "word_count": 1094, "text": "Porsche swung..."}
```

**Résultat obtenu :** 11 documents retenus sur 14. Les 3 rejetés sont :
- `reuters.com` → paywall (HTTP 200 mais contenu chiffré côté client, `trafilatura` retourne `None`)
- `carscoops.com` → protection anti-bot
- `italpassion.fr` → texte trop court après extraction

---

## Phase 2 — Information Extraction

### Cellule 9 — Chargement du corpus JSONL *(Python)*

```python
def load_jsonl(path: pathlib.Path) -> list[dict]:
    docs = []
    with path.open("r", encoding="utf-8") as f:
        for line in f:
            docs.append(json.loads(line.strip()))
    return docs

corpus = load_jsonl(OUTPUT_JSONL)
```

**Rôle :** Relire le fichier `crawler_output.jsonl` depuis le disque pour permettre à Phase 2 de fonctionner **indépendamment** de Phase 1. Si on réexécute uniquement les cellules Phase 2 sans relancer le crawler, les données sont déjà disponibles sur disque.

**Choix de design important :** Découpler les deux phases en passant par le disque est une bonne pratique en data engineering — on évite de tout relancer si seule la Phase 2 doit être modifiée.

---

### Cellule 10 — NER (Named Entity Recognition) *(Python)*

```python
TARGET_LABELS = {"ORG", "PERSON", "GPE", "DATE", "MONEY"}

def extract_entities(doc, source_url):
    for ent in doc.ents:
        if ent.label_ not in TARGET_LABELS: continue
        if len(text) < 2 or text.islower(): continue
        entities.append({"entity": text, "label": ent.label_, "source_url": source_url})

# Passe NLP unique — réutilisée pour les relations
spacy_docs = list(nlp.pipe(texts, batch_size=4))
```

**Rôle :** Identifier les **entités nommées** dans chaque article — ce seront les **nœuds** du futur Knowledge Graph.

**Labels ciblés et leur signification :**

| Label | Signification | Exemples trouvés |
|---|---|---|
| `ORG` | Organisation | Porsche, Volkswagen, BMW, Stellantis |
| `PERSON` | Personne nommée | Oliver Blume, Carlos Tavares |
| `GPE` | Lieu géopolitique | Germany, China, United States |
| `DATE` | Référence temporelle | 2025, Q3 2024, last year |
| `MONEY` | Montant financier | €11 billion, $26 billion |

**Filtre appliqué :**
- Entités < 2 caractères → bruit (ex: articles "a", "an")
- Entités entièrement en minuscules pour ORG/PERSON/GPE → probablement des noms communs mal taggés (ex: "company", "car") — **exception** pour DATE et MONEY car les dates/montants peuvent être en minuscules ("last year", "millions")

**Optimisation clé :** `nlp.pipe(texts, batch_size=4)` traite les documents en batch, ce qui est 3-5× plus rapide que d'appeler `nlp(text)` en boucle. Les `spacy_docs` sont stockés en mémoire et **réutilisés** par la cellule suivante (pas de second appel NLP).

**Résultat :** 787 entités brutes extraites au total.

---

### Cellule 11 — Extraction de relations par Dependency Parsing *(Python)*

Cette cellule est la plus complexe du notebook. Elle implémente trois fonctions qui travaillent ensemble.

#### `get_entity_span(token)` — Résolution d'entité

```python
def get_entity_span(token):
    for ent in token.doc.ents:
        if ent.start <= token.i < ent.end and ent.label_ in TARGET_LABELS:
            return ent
    return None
```

Vérifie si un token donné fait partie d'une entité nommée connue. Nécessaire car une entité comme "Porsche AG" s'étend sur 2 tokens — il faut chercher l'entité span qui "contient" le token.

#### `_resolve_subject(token)` — Résolution du sujet

```python
def _resolve_subject(token):
    # 1. Cherche nsubj/nsubjpass direct
    # 2. Cherche un niveau plus profond (compound)
    # 3. Hérite du sujet du verbe parent (coordination/subordination)
```

Cherche l'entité qui est **sujet** du verbe courant. Trois stratégies en cascade :

1. **Sujet direct** (`nsubj` / `nsubjpass`) : le cas le plus fréquent. Ex: *"**Porsche** reported losses"* → `nsubj(reported, Porsche)`

2. **Un niveau plus profond** : parfois le sujet est un groupe nominal et l'entité est un token enfant. Ex: *"**The** **Porsche** **AG** reported"* → `nsubj` pointe vers "The", mais l'entité est sur "Porsche AG"

3. **Héritage par coordination** : quand un verbe est coordonné ou subordonné à un autre (`conj`, `relcl`, `xcomp`, `advcl`), il hérite du sujet de son verbe parent. Ex: *"**Volkswagen** announced losses and **warned** investors"* — "warned" n'a pas de `nsubj` propre, mais hérite "Volkswagen" de "announced"

#### `_resolve_object(token)` — Résolution de l'objet

```python
def _resolve_object(token):
    # dobj / attr        → objet direct / attribut
    # prep → pobj        → objet prépositonnel ("based in Germany")
    # agent → pobj       → agent passif ("acquired by BMW")
```

Cherche l'entité qui est **objet** du verbe, avec 3 patterns :

| Pattern | Exemple | Triple produit |
|---|---|---|
| `dobj` | *Stellantis **hired** Carlos Tavares* | `(Stellantis) → [hire] → (Carlos Tavares)` |
| `nsubjpass + agent` | *JLR **was acquired** by Tata Motors* | `(JLR) → [acquire_by] → (Tata Motors)` |
| `prep + pobj` | *Mercedes **is based** in Germany* | `(Mercedes) → [base_in] → (Germany)` |

La **relation** est enrichie par le nom de la préposition : `base_in`, `sell_to`, `operate_in`... ce qui évite d'avoir uniquement des verbes génériques et donne des arêtes plus expressives pour le Knowledge Graph.

#### `extract_relations(doc, source_url)` — Extraction principale

```python
def extract_relations(doc, source_url):
    seen = set()  # déduplication intra-document
    for token in doc:
        if token.pos_ != "VERB": continue
        subject_ent = _resolve_subject(token)
        object_ent, rel_suffix = _resolve_object(token)
        # ... construction du triple
```

Itère sur chaque token du document. Pour chaque **verbe** :
1. Résoudre le sujet → si absent, `continue`  
2. Résoudre l'objet → si absent ou identique au sujet, `continue`
3. Construire le triple `(subject, verb+suffix, object)`
4. Vérifier si déjà vu (`seen` set) → déduplication intra-document
5. Ajouter le triple avec la phrase source tronquée (200 chars max) pour traçabilité

**Résultat :** 37 triples extraits. Exemples réels :

```
(Mercedes-Benz)  → [celebrate_at] → (the end of January)
(Källenius)      → [scrap_in]     → (February 2024)
(China)          → [remain_for]   → (Mercedes)
(Toyota)         → [mirror]       → (BMW)
(JLR Automotive) → [own_by]       → (Tata Motors)
```

---

### Cellule 12 — Déduplication et normalisation des entités *(Python)*

```python
ALIASES = {"Vw": "Volkswagen", "VW": "Volkswagen", "Bmw": "BMW", ...}
_PRESERVE_UPPER = {"BMW", "VW", "EV", "CEO", ...}

def normalize_entity(raw: str) -> str:
    if raw.strip().upper() in _PRESERVE_UPPER:
        return raw.strip().upper()  # BMW → BMW (pas Bmw)
    titled = raw.strip().title()
    return ALIASES.get(titled, titled)  # Vw → Volkswagen
```

**Rôle :** Agréger les mentions de la même entité qui apparaissent sous des formes légèrement différentes.

**Problème résolu :** Sans normalisation, "VW", "Vw", et "vw" seraient 3 entrées distinctes dans le CSV final, alors qu'il s'agit de la même organisation. De même, Python's `.title()` transforme "BMW" en "Bmw" — un problème classique avec les acronymes.

**Stratégie en deux couches :**

1. **Préservation des acronymes connus** : si le token en majuscules appartient à `_PRESERVE_UPPER` (BMW, VW, EU, US...), on retourne directement la version tout-majuscules
2. **Table d'alias** : `ALIASES` fait correspondre les variantes connues à une forme canonique. Ex: "VW" et "Vw" → "Volkswagen"

**Agrégation pandas :**  
Un `defaultdict` accumule le comptage et l'ensemble des URLs sources pour chaque paire `(entity_normalisée, label)`. On obtient en sortie un DataFrame avec :
- `entity` : nom canonique
- `label` : type NER
- `frequency` : nombre total de mentions dans tout le corpus
- `source_count` : nombre d'articles différents où l'entité apparaît
- `source_urls` : liste des URLs (pipe-séparées)

**Résultat :**

```
      entity  label  frequency  source_count
     Porsche    ORG         38             5
    Mercedes    ORG         26             1
       China    GPE         25             8
  Stellantis    ORG         22             3
          VW    ORG         21             3
  Volkswagen    ORG         19             3
         BMW    ORG         18             2
```

---

### Cellule 13 — Export CSV *(Python)*

```python
df_entities.to_csv("extracted_knowledge.csv", index=False, encoding="utf-8")
df_relations.drop_duplicates(subset=["subject", "relation", "object"])
df_relations.to_csv("extracted_relations.csv", index=False, encoding="utf-8")
```

**Rôle :** Persister les résultats des deux phases dans des fichiers CSV standardisés — les **livrables finaux** de la Lab Session 1.

**Fichier 1 — `extracted_knowledge.csv`**  
347 lignes. Colonnes :

| Colonne | Type | Description |
|---|---|---|
| `entity` | str | Nom canonique de l'entité |
| `label` | str | Type NER (ORG, GPE, PERSON, DATE, MONEY) |
| `frequency` | int | Nombre de mentions dans tout le corpus |
| `source_count` | int | Nombre d'articles distincts |
| `source_urls` | str | URLs sources (pipe-séparées) |

**Fichier 2 — `extracted_relations.csv`**  
37 lignes (après déduplication globale). Colonnes :

| Colonne | Type | Description |
|---|---|---|
| `subject` | str | Entité sujet du triple |
| `subject_label` | str | Type NER du sujet |
| `relation` | str | Verbe lemmatisé + suffixe prép. |
| `object` | str | Entité objet du triple |
| `object_label` | str | Type NER de l'objet |
| `sentence` | str | Phrase source (≤ 200 chars) |
| `source_url` | str | URL de l'article d'origine |

Ces deux fichiers seront utilisés dans les sessions suivantes pour construire le Knowledge Graph.

---

## Résultats globaux du pipeline

| Étape | Résultat |
|---|---|
| URLs testées | 14 |
| URLs retenues (≥ 500 mots) | 11 (79%) |
| Mots moyen par article | ~965 mots |
| Entités brutes extraites | 787 |
| Entités uniques (après dédup) | 347 |
| Triples de relations | 37 |
| Taille `extracted_knowledge.csv` | 347 lignes × 5 colonnes |
| Taille `extracted_relations.csv` | 37 lignes × 7 colonnes |

**Distribution des entités par type :**

| Label | Unique | Total mentions |
|---|---|---|
| ORG | 114 | 337 |
| DATE | 103 | 233 |
| GPE | 34 | 91 |
| PERSON | 47 | 74 |
| MONEY | 49 | 52 |

---

## Limites et pistes d'amélioration

1. **VW ≠ Volkswagen dans le CSV** : malgré le dictionnaire d'alias, 40 mentions restent séparées entre "VW" (21) et "Volkswagen" (19). Il faudrait ajouter `"VW": "Volkswagen"` directement dans `ALIASES` en retirant VW de `_PRESERVE_UPPER`.

2. **37 triples seulement** : la presse automobile utilise beaucoup la voix passive et les nominalisations ("the profit decline of Porsche" au lieu de "Porsche declined profits"). Les patterns de dependency parsing ne capturent pas les groupes nominaux.

3. **3 URLs non crawlées** : Reuters et Carscoops utilisent du JavaScript côté client ou des paywalls. Solution : utiliser une API (ex: NewsAPI) ou un headless browser (Playwright) pour ces sources.

4. **Dates non normalisées** : "2025", "last year", "Q3 2025" sont trois entités DATE distinctes alors qu'elles pourraient référer à la même période. Une normalisation temporelle (ex: avec `dateparser`) améliorerait la qualité du Knowledge Graph.
