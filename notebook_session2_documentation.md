# Documentation — Lab Session 2 : Construction de la Base de Connaissances RDF

Ce document décrit **chaque cellule de code de la Session 2** du notebook `notebook.ipynb`, de manière concrète et détaillée. La Session 2 transforme les CSV produits par la Session 1 (NLP/NER) en un graphe de connaissances RDF complet, aligné sur Wikidata, prêt pour le Knowledge Graph Embedding (Session 3).

---

## Vue d'ensemble du pipeline Session 2

```
extracted_knowledge.csv  ──┐
                            ├─► Phase 3 ─► g_private (KB privée, ~905 triples)
extracted_relations.csv  ──┘               │
                                            ▼
                                   Phase 4 ─► g_alignment (owl:sameAs, ~50 triples)
                                            │
                                   Phase 5 ─► g_alignment enrichi (equivalentProperty)
                                            │
                                   Phase 6 ─► g_expanded (Wikidata, ~125k triples)
                                            │
                                   Phase 7 ─► g_final (merged, nettoyé) ─► automotive_kb.nt
                                                                         ─► ontology.ttl
                                   Phase 8 ─► Rapport statistique
```

---

## Phase 3 — Construction de la KB Privée

### Cellule : Imports & Namespaces (`#VSC-2eb9b8a5`)

```python
AUTO = Namespace("http://automotive-crisis.org/kb/")
WD   = Namespace("http://www.wikidata.org/entity/")
WDT  = Namespace("http://www.wikidata.org/prop/direct/")
g_private = Graph()
```

**Ce que fait cette cellule :**

- Importe `rdflib` (manipulation de graphes RDF), `requests` (appels HTTP), `time` (délais anti-rate-limit).
- Définit **trois namespaces** :
  - `AUTO` : l'espace de noms personnalisé pour toutes nos entités/prédicats privés (ex. `AUTO:Porsche`, `AUTO:ownBy`).
  - `WD` : URIs des entités Wikidata (ex. `WD:Q40993` = Porsche).
  - `WDT` : URIs des propriétés directes Wikidata (ex. `WDT:P127` = "owned by").
- Crée le graphe principal `g_private` qui contiendra toute la KB privée, et y "bind" les préfixes pour produire un Turtle/N-Triples lisible.

**Concrètement :** après cette cellule, on a un graphe RDF vide prêt à recevoir des triples.

---

### Cellule : Entités NER → Instances RDF (`#VSC-8b094d7d`)

```python
LABEL_CLASS = {"ORG": AUTO.Organization, "PERSON": AUTO.Person, "GPE": AUTO.GeopoliticalEntity}
for _, row in df_ent.iterrows():
    g_private.add((node, RDF.type, cls))
    g_private.add((node, RDFS.label, Literal(row["entity"], lang="en")))
    g_private.add((node, AUTO.mentionFrequency, Literal(int(row["frequency"]), datatype=XSD.integer)))
```

**Ce que fait cette cellule :**

1. Recharge les CSVs de la Session 1 (`extracted_knowledge.csv`, `extracted_relations.csv`) pour que cette phase puisse tourner indépendamment.
2. Mappe les labels NER vers des classes OWL :
   - `ORG` → `AUTO:Organization`
   - `PERSON` → `AUTO:Person`
   - `GPE` → `AUTO:GeopoliticalEntity`
   - `DATE` et `MONEY` sont **ignorés ici** (ils apparaîtront comme littéraux dans les triples de relations).
3. Pour chaque entité unique, ajoute **3 triples** :
   - `(AUTO:Porsche, rdf:type, AUTO:Organization)` — déclaration de type
   - `(AUTO:Porsche, rdfs:label, "Porsche"@en)` — libellé lisible
   - `(AUTO:Porsche, AUTO:mentionFrequency, 38^^xsd:integer)` — fréquence de mention dans le corpus
4. Déclare aussi la hiérarchie de classes : `AUTO:Organization rdfs:subClassOf AUTO:Entity`.
5. La fonction `uri_slug(name)` nettoie les noms pour les URI : `"Jaguar Land Rover"` → `"Jaguar_Land_Rover"`.

**Concrètement :** ~200 entités dans le CSV → ~170 nœuds URI créés (ORG/PERSON/GPE seulement).

---

### Cellule : Triples de relations → RDF (`#VSC-3780e523`)

```python
def relation_to_camel(rel: str) -> str:
    parts = str(rel).split("_")
    return parts[0] + "".join(p.capitalize() for p in parts[1:])
# ex : "sell_in" → "sellIn", "own_by" → "ownBy"
```

**Ce que fait cette cellule :**

1. Pour chaque ligne de `extracted_relations.csv` (43 triples issus de la dépendance syntaxique) :
   - Convertit la relation en camelCase : `own_by` → `ownBy`, `sell_in` → `sellIn`. Cela produit un URI de prédicat valide : `AUTO:ownBy`.
   - Si le sujet n'est pas un `ORG/PERSON/GPE`, le triple est **ignoré** (on ne peut pas créer un nœud URI pour un montant d'argent).
   - L'objet est converti en nœud URI si c'est une entité (`ORG/PERSON/GPE`), ou en `Literal` si c'est une date/montant.
2. Déclare chaque prédicat comme `owl:ObjectProperty` (si objet = URI) ou `owl:DatatypeProperty` (si objet = littéral).
3. Ajoute un triple de **provenance** : `(AUTO:Porsche, AUTO:fromSource, "https://article.com/...")` pour tracer l'origine de chaque relation.

**Exemple concret :**

| subject | relation | object | → Triple |
|---|---|---|---|
| Jaguar Land Rover | own_by | Tata Motors | `(AUTO:Jaguar_Land_Rover, AUTO:ownBy, AUTO:Tata_Motors)` |
| Stellantis | lose | $26 Billion | `(AUTO:Stellantis, AUTO:lose, "$26 Billion"^^xsd:string)` |

---

### Cellule : Sérialisation en N-Triples (`#VSC-c144c8c6`)

```python
g_private.serialize(destination="automotive_kb_private.nt", format="nt")
```

**Ce que fait cette cellule :**

Exporte le graphe `g_private` dans le format **N-Triples** (`.nt`), le format RDF le plus simple : une ligne = un triple. Résultat : `automotive_kb_private.nt` (~113 KB, **905 triples**), qui dépasse largement l'objectif minimum de 100 triples.

---

## Phase 4 — Alignement des Entités sur Wikidata

### Cellule : Lookup API Wikidata (`#VSC-89122757`)

```python
CORE_ENTITIES = {
    "ORG":    ["Porsche", "Volkswagen", "Mercedes-Benz", ...],
    "PERSON": ["Oliver Blume", "Carlos Tavares", ...],
    "GPE":    ["China", "Germany", "United States", ...],
}
def wikidata_search(label, entity_type) -> dict:
    # GET https://www.wikidata.org/w/api.php?action=wbsearchentities&search=Porsche
```

**Ce que fait cette cellule :**

1. Définit une liste de **27 entités centrales** à aligner (15 ORG, 5 PERSON, 7 GPE).
2. Pour chaque entité, appelle l'API `wbsearchentities` de Wikidata qui retourne les 5 meilleurs candidats.
3. Applique un **scoring de confiance** sur le meilleur résultat :
   - `1.0` : correspondance exacte de la chaîne (ex. `"Porsche"` → `"Porsche"`)
   - `0.8` : correspondance partielle (ex. `"BMW"` → `"BMW Group"`)
   - `0.6` : correspondance lâche (ex. `"Audi"` → `"album"` — faux positif)
4. `time.sleep(0.5)` entre chaque appel pour ne pas dépasser le rate limit de l'API publique.
5. Construit un `DataFrame` `df_alignment` avec les colonnes : `label, ner_type, wikidata_id, wikidata_uri, match_label, confidence`.

**Résultats obtenus :** 27 entités, 25 liées dont 12 à confiance 1.0.

> ⚠️ **Cas particuliers à noter :**
> - `"Nissan"` → Q270195 (*Nissan-lez-Enserune*, commune française) — faux positif, confidence 0.8
> - `"Audi"` → confidence 0.6 → éliminé par le seuil, reste non lié

---

### Cellule : Construction du graphe d'alignement (`#VSC-b729f59d`)

```python
CONFIDENCE_THRESHOLD = 0.7
if row["confidence"] >= 0.7:
    g_alignment.add((auto_uri, OWL.sameAs, wd_uri))
    g_alignment.add((auto_uri, AUTO.alignmentConfidence, Literal(0.8, datatype=XSD.float)))
else:
    g_alignment.add((auto_uri, RDF.type, cls))  # définition locale
```

**Ce que fait cette cellule :**

Pour chaque entité du `df_alignment` :
- Si `confidence ≥ 0.7` : ajoute un triple `owl:sameAs` qui déclare l'équivalence entre notre URI privée et l'URI Wikidata. Ex : `AUTO:Porsche owl:sameAs wd:Q40993`. Ajoute aussi la confiance comme annotation.
- Si `confidence < 0.7` (cas de `Audi`) : définit l'entité localement dans le graphe d'alignement avec un commentaire explicatif.

**Pourquoi `owl:sameAs` ?** C'est le prédicat OWL standard pour déclarer que deux URIs désignent la même chose du monde réel. Cela permettra aux algorithmes de KGE d'exploiter la connaissance Wikidata sur nos entités.

---

### Cellule : Export des alignements (`#VSC-54ff46b6`)

```python
df_alignment.to_csv("entity_alignment.csv", index=False, encoding="utf-8")
g_alignment.serialize(destination="alignment.ttl", format="turtle")
```

**Ce que fait cette cellule :**

Exporte deux fichiers :
- `entity_alignment.csv` : tableau lisible (27 lignes × 6 colonnes) pour vérification humaine.
- `alignment.ttl` (partiel, ~3.3 KB) : version Turtle du graphe d'alignement, sera enrichi en Phase 5.

---

## Phase 5 — Alignement des Prédicats via SPARQL

### Cellule : Recherche SPARQL de propriétés Wikidata (`#VSC-71740671`)

```python
query = f"""
SELECT ?property ?propertyLabel WHERE {{
  ?property a wikibase:Property .
  ?property rdfs:label ?propertyLabel .
  FILTER(CONTAINS(LCASE(?propertyLabel), "{keyword.lower()}"))
  FILTER(LANG(?propertyLabel) = "en")
}}
LIMIT 10
"""
```

**Ce que fait cette cellule :**

1. Extrait les **verbes de base** de chaque relation dans `extracted_relations.csv` : `"sell_in"` → `"sell"`, `"own_by"` → `"own"`, etc.
2. Pour chaque verbe, envoie une requête SPARQL au endpoint de Wikidata (`https://query.wikidata.org/sparql`) pour trouver des propriétés Wikidata dont le libellé contient ce mot-clé.
3. Affiche les candidats (ID + libellé) pour **inspection manuelle**.
4. `time.sleep(1.0)` entre chaque requête.

**Concrètement :** pour `"own"` on obtient des candidats comme `P127 = owned by`, `P1830 = owner of`, etc.

---

### Cellule : Validation manuelle & `equivalentProperty` (`#VSC-f9adc441`)

```python
PREDICATE_ALIGNMENT = {
    "ownBy":   ("P127",  "equivalent"),   # owned by → owl:equivalentProperty
    "lose":    ("P1342", "subPropertyOf"),# quantity → rdfs:subPropertyOf
    "build":   ("P176",  "equivalent"),   # manufacturer
    ...
}
for pred_local, (wdt_id, rel_type) in PREDICATE_ALIGNMENT.items():
    g_alignment.add((auto_pred, OWL.equivalentProperty, wdt_pred))
```

**Ce que fait cette cellule :**

À partir du dictionnaire `PREDICATE_ALIGNMENT` (édité manuellement après inspection des candidats SPARQL) :
- `"equivalent"` → `owl:equivalentProperty` : le prédicat privé et le prédicat Wikidata ont **exactement le même sens**. Ex : `AUTO:ownBy owl:equivalentProperty wdt:P127`.
- `"subPropertyOf"` → `rdfs:subPropertyOf` : le prédicat privé est plus spécifique ou son sens est approché. Ex : `AUTO:lose rdfs:subPropertyOf wdt:P1342`.

Ces déclarations enrichissent `alignment.ttl` et permettront à un raisonneur OWL ou à un algorithme de KGE d'exploiter les alignements sémantiques entre la KB privée et Wikidata.

---

## Phase 6 — Expansion de la KB via SPARQL

### Cellule : Collecte des QIDs alignés (`#VSC-e6477386`)

```python
LINKED = {
    row["label"]: row["wikidata_id"]
    for _, row in df_alignment.iterrows()
    if row["confidence"] >= 0.7
}
# {"Porsche": "Q40993", "Volkswagen": "Q246", ...}
```

**Ce que fait cette cellule :**

Construit le dictionnaire `LINKED` qui mappe `nom → QID Wikidata` pour toutes les entités avec confiance ≥ 0.7. Ce dictionnaire sera la **graine** des requêtes d'expansion. Définit aussi `EXCLUDED_PREDICATES` : un ensemble de prédicats verbeux à filtrer (`schema:description`, `skos:altLabel`, etc.) qui ne contribuent pas à la structure du graphe.

---

### Cellule : Expansion 1-hop (`#VSC-f7006bbb`)

```python
query = f"""
SELECT ?p ?o WHERE {{
  wd:{qid} ?p ?o .
  FILTER(isIRI(?o) || (isLiteral(?o) && LANG(?o) = "en") || (isLiteral(?o) && LANG(?o) = ""))
}}
LIMIT 500
"""
```

**Ce que fait cette cellule :**

Pour **chaque entité alignée** (25 entités × LIMIT 500) :
- Récupère tous les triplets directs `(wd:Q_entité, ?propriété, ?valeur)` depuis Wikidata.
- Le filtre SPARQL ne garde que : (1) les objets URI, (2) les littéraux anglais, (3) les littéraux sans langue (nombres, dates…).
- La fonction `binding_to_node()` convertit les résultats SPARQL en termes rdflib (`URIRef` ou `Literal`), et **ignore** les nœuds blancs et les littéraux non-anglais.
- Les triplets filtrés sont ajoutés au graphe `g_expanded`.
- `time.sleep(1.5)` entre chaque entité.

**Résultat :** ~12 500 triples ajoutés au total.

---

### Cellule : Expansion par prédicats contrôlés (`#VSC-eadaebd1`)

```python
EXPANSION_PREDICATES = {
    "P31":  ("instance of",  20_000),
    "P17":  ("country",      20_000),
    "P452": ("industry",     20_000),
    "P127": ("owned by",     20_000),
    "P355": ("subsidiary",   20_000),
    ...
}
query = f"""
SELECT ?s ?o WHERE {{
  ?s wdt:{pid} ?o .
  FILTER(isIRI(?o) || ...)
}}
LIMIT {limit}
"""
```

**Ce que fait cette cellule :**

C'est le moteur principal du volume. Au lieu de partir d'entités spécifiques, elle tire des **tranches de Wikidata entières** selon des propriétés pertinentes pour le domaine :

| Propriété | Sens | Volume |
|---|---|---|
| `P31` | instance of (ex. *"est une entreprise automobile"*) | 20k |
| `P17` | country | 20k |
| `P452` | industry | 20k |
| `P127` | owned by | 20k |
| `P355` | subsidiary | 20k |
| `P1366` | replaced by | 10k |
| `P571` | inception (date de création) | 10k |
| `P856` | official website | 10k |

Un garde-fou `if len(g_expanded) >= TRIPLE_TARGET_MAX: break` stoppe l'expansion à 200 000 triples.

**Résultat :** ~110 000 triples supplémentaires → total ~125 000 triples dans `g_expanded`.

---

### Cellule : Expansion 2-hop (`#VSC-917bfad7`)

```python
query = f"""
SELECT ?intermediate ?p ?o WHERE {{
  wd:{qid} wdt:P31|wdt:P452 ?intermediate .
  ?intermediate ?p ?o .
  FILTER(isIRI(?o) || ...)
}}
LIMIT 10000
"""
```

**Ce que fait cette cellule :**

Pour les 3 entités centrales (**Porsche, Volkswagen, Stellantis**) :
1. Suit d'abord les propriétés `P31` (instance of) et `P452` (industry) pour trouver des **entités intermédiaires** (ex. entités du secteur automobile).
2. Puis récupère tous les triplets de ces intermédiaires.

Cela "densifie" le voisinage de nos entités clés dans le graphe, ce qui améliore la qualité des embeddings de KGE.

---

## Phase 7 — Nettoyage & Export

### Cellule : Fusion & Déduplication (`#VSC-04f0f6e6`)

```python
g_final = Graph()
for triple in g_private:
    g_final.add(triple)
for triple in g_expanded:
    g_final.add(triple)
```

**Ce que fait cette cellule :**

1. **Fusion** : merge `g_private` + `g_expanded` dans `g_final`. rdflib gère nativement la déduplication — si le même triple est présent dans les deux graphes, il n'est stocké qu'une fois.

2. **Nettoyage des prédicats verbeux** : supprime les triples ayant des prédicats inutiles pour le KGE (`schema:description`, `schema:dateModified`, `skos:altLabel`, `rdfs:comment`). Ces prédicats ne portent pas de structure relationnelle.

3. **Suppression des nœuds isolés** : calcule le degré de chaque nœud URI (nombre d'apparitions comme sujet ou objet). Les nœuds de degré < 2 (connectés à un seul triple) sont considérés isolés et leurs triples sont supprimés. Des nœuds isolés ne peuvent pas être correctement intégrés dans un embedding.

**Exemple** : un URI `wd:Q12345` qui n'apparaît qu'une seule fois comme objet sans jamais être sujet → supprimé.

**Résultat :** ~125 842 triples finaux, nettoyés et cohérents.

---

### Cellule : Écriture de l'ontologie (`#VSC-2422eb0a`)

```python
classes = {
    AUTO.Entity:             ("Entity",               None),
    AUTO.Organization:       ("Organization",          AUTO.Entity),
    AUTO.Person:             ("Person",                AUTO.Entity),
    AUTO.GeopoliticalEntity: ("Geopolitical Entity",  AUTO.Entity),
    AUTO.CarManufacturer:    ("Car Manufacturer",      AUTO.Organization),
    AUTO.CarBrand:           ("Car Brand",             AUTO.Organization),
}
```

**Ce que fait cette cellule :**

Construit un graphe d'ontologie séparé `g_onto` avec :
- **6 classes OWL** dans une hiérarchie `Entity → Organization → CarManufacturer`, etc.
- **3 Object Properties** (`AUTO:ownBy`, `AUTO:subsidiary`, `AUTO:manufacture`) avec domaine et portée définis.
- **4 Datatype Properties** (`AUTO:mentionFrequency`, `AUTO:alignmentConfidence`, `AUTO:nerLabel`, `AUTO:fromSource`) avec leurs types XSD.
- Un en-tête d'ontologie (`OWL:Ontology`) avec titre et description.

Exporte dans `ontology.ttl` (Turtle, 1.8 KB). Ce fichier est la **définition formelle du schéma** de la KB, séparée des données.

---

### Cellule : Export final en N-Triples (`#VSC-fb766ad8`)

```python
g_final.serialize(destination="automotive_kb.nt", format="nt")
g_verify = Graph()
g_verify.parse("automotive_kb.nt", format="nt")
```

**Ce que fait cette cellule :**

- Sérialise `g_final` dans `automotive_kb.nt` (~15.8 MB, 125 842 lignes).
- **Vérifie** immédiatement le fichier produit en le reparsant dans un nouveau graphe `g_verify` et en comptant les triples. Si le nombre correspond, le fichier est valide.
- Affiche un résumé de taille pour tous les fichiers de livraison.

N-Triples est le format recommandé pour les pipelines KGE (OpenKE, PyKEEN) car il est simple, compact et facile à parser.

---

## Phase 8 — Rapport Statistique

### Cellule : Rapport final (`#VSC-cb08eaf7`)

**Ce que fait cette cellule :**

Calcule et affiche un rapport complet sur la KB finale :

1. **Comptage des triples** : KB privée (905) + Wikidata étendu (~125k) + alignement (~50).
2. **Taille du graphe** : nombre d'entités uniques (URIs) et de prédicats uniques.
3. **Vérification des objectifs** :
   - ✓ Triples entre 50k et 200k
   - Entités entre 5k et 30k
   - Relations entre 50 et 200
4. **Taux d'alignement** : X/27 entités liées à Wikidata (confiance ≥ 0.7).
5. **Top 15 prédicats** par fréquence d'utilisation dans le graphe final.

Ce rapport constitue l'annexe quantitative du rapport de session.

---

## Fichiers générés

| Fichier | Taille | Contenu |
|---|---|---|
| `automotive_kb_private.nt` | 113 KB | KB extraite du corpus (905 triples) |
| `automotive_kb.nt` | 15,8 MB | KB finale complète (~125 842 triples) |
| `ontology.ttl` | 1,8 KB | Schéma OWL (classes + propriétés) |
| `alignment.ttl` | 3,3 KB | Alignement owl:sameAs + equivalentProperty |
| `entity_alignment.csv` | 2,1 KB | Table d'alignement lisible (27 entités) |

---

## Points d'attention

- **Nissan** est mal lié : Q270195 = *Nissan-lez-Enserune* (commune française). Le bon QID est Q875942 (*Nissan Motor*). À corriger dans `CORE_ENTITIES`.
- **Audi** a été filtré (confidence 0.6 < 0.7) car le top résultat Wikidata était "album". Le bon QID est Q23317 (*Audi AG*). À ajouter manuellement dans `PREDICATE_ALIGNMENT`.
- **Encodage** : `Ola Källenius` est stocké comme `Ola KÃ¤llenius` dans le CSV — bug d'encodage UTF-8/Latin-1 dans `.to_csv()`. Corriger avec `encoding='utf-8-sig'`.
