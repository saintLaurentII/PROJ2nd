# Documentation — Lab Session 3 : Raisonnement SWRL & Knowledge Graph Embedding

Ce document décrit **chaque cellule de code de la Session 3** du notebook `notebook.ipynb`, de manière concrète et détaillée. La Session 3 s'appuie sur la KB automotive construite en Session 2 pour y appliquer deux techniques complémentaires : le **raisonnement par règles SWRL** (OWLReady2 + Pellet) et les **plongements de graphe de connaissances** (PyKEEN — TransE & DistMult).

---

## Vue d'ensemble du pipeline Session 3

```
family.owl (ontologie de test)
    │
    ▼
[Cellule 65] OWLReady2 — chargement                ← correction IRI Windows
    │
    ▼
[Cellule 67] Règle SWRL + Pellet                  ← Person ∧ age > 60 → OldPerson
    │                                              ← namespace swrlb enregistré manuellement
    │
    ─────────────────────────────────────────────────────────────────
    │
    ▼
automotive_kb.nt (~125 884 triples)
    │
    ▼
[Cellule 71] Audit KB — filtrage URI-only          → uri_triples (~50–100k)
    │
    ▼
[Cellule 73] Split 80/10/10                        → train.txt  valid.txt  test.txt
    │
    ▼
[Cellule 75] TriplesFactory partagé               → vocabulaire entités/relations commun
    │
    ├─► [Cellule 77] TransE (dim=100, lr=0.01, 100 epochs)  → models/transe/
    │
    └─► [Cellule 79] DistMult (même config)                 → models/distmult/
    │
    ▼
[Cellule 81] Évaluation filtrée — MRR, H@1, H@3, H@10
    │
    ▼
[Cellule 83] Sensibilité à la taille de la KB     → sensitivity_plot.png
    │
    ▼
[Cellules 85/87/89] Analyse des embeddings        → nearest neighbors, tsne_plot.png, norms
    │
    ▼
[Cellules 92/93] Règle SWRL automotive            ← Organization ∧ subsidiary → ownBy
    │
    ▼
[Cellule 95] Analogie vectorielle                 ← vec(P355) + vec(Org) ≈ vec(P127) ?
    │
    ▼
[Cellule 96] Réflexion critique (Markdown)
```

---

## Partie 1 — Raisonnement par Règles SWRL

### Cellule 62 — En-tête Partie 1 *(Markdown)*

Cellule documentaire. Présente la règle SWRL à implémenter :

```
Person(?p) ∧ age(?p, ?a) ∧ swrlb:greaterThan(?a, 60) → OldPerson(?p)
```

Rappel que le raisonneur **Pellet** (Java) est requis — HermiT ne supporte pas les built-ins SWRLB comme `greaterThan`.

---

### Cellule 63 — Installation OWLReady2 *(Python)*

```python
# %pip install owlready2 --quiet
```

Commenté car déjà installé dans l'environnement `ESILV_A4`. À décommenter lors d'un premier lancement.

---

### Cellule 64 — En-tête Step 1.2 *(Markdown)*

Présente la prochaine étape : chargement de `family.owl` et inspection de son contenu.

---

### Cellule 65 — Chargement de family.owl *(Python)*

```python
from owlready2 import *
import pathlib

family_path = pathlib.Path("family.owl").resolve()

# Correction Windows : as_uri() produit file:///C:/...
# OWLReady2 supprime 'file://' (2 slashes), il reste /C:/... → chemin invalide sous Windows.
# En utilisant 2 slashes on obtient file://C:/... → après stripping : C:/... (valide)
family_iri = "file://" + str(family_path).replace("\\", "/")
onto = get_ontology(family_iri).load()
```

**Ce que fait cette cellule :**

1. Résout le chemin absolu vers `family.owl` avec `pathlib.Path.resolve()`.
2. Construit manuellement l'IRI en format `file://C:/...` — **correction spécifique Windows** :
   - `pathlib.Path.as_uri()` produit `file:///C:/...` (3 slashes)
   - OWLReady2 supprime exactement `file://` (2 slashes) avant d'ouvrir le fichier, laissant `/C:/...` — un chemin invalide sous Windows
   - La solution : construire `"file://" + str(path).replace("\\", "/")` pour obtenir `file://C:/Users/...`, ce qui après stripping donne le chemin valide `C:/Users/...`
3. Charge l'ontologie avec `get_ontology(iri).load()` et affiche :
   - L'IRI de base (`http://www.owl-ontologies.com/unnamed.owl#`)
   - Les 16 classes : `Person`, `Male`, `Female`, `Parent`, `Father`, `Mother`, `Child`, `Son`, `Daughter`, `Sibling`, `Brother`, `Sister`, `Grandparents`, `Grandparent`, `Grandfather`, `Grandmother`
   - Les 3 data properties : `age`, `name`, `nationality`
   - Les individus (0 dans cette version de `family.owl`)

> **Problème découvert :** le fichier `family.owl` de Protégé ne contient aucun individu. La cellule suivante gère ce cas en créant des individus synthétiques.

---

### Cellule 66 — En-tête Step 1.3 *(Markdown)*

Explique la règle SWRL à définir et précise le rôle de `swrlb:greaterThan`.

---

### Cellule 67 — Règle SWRL + Raisonneur Pellet *(Python)*

```python
# Enregistrement du namespace swrlb (built-ins SWRL)
swrlb_onto = get_ontology("http://www.w3.org/2003/11/swrlb")
with swrlb_onto:
    class greaterThan(AnnotationProperty):   # AnnotationProperty → BuiltinAtom dans rule.py
        pass

with onto:
    class OldPerson(onto.Person):   # Nouvelle classe déclarée dynamiquement
        pass

    # Auto-création d'individus si family.owl n'en contient pas
    if not list(onto.individuals()):
        for ind_name, ind_age in [
            ("Alice", 25), ("Bob", 40), ("Carol", 65),
            ("Dave", 70), ("Eve", 10), ("Frank", 69),
        ]:
            ind = onto.Person(ind_name)
            ind.age = [ind_age]

    rule = Imp()
    rule.set_as_rule(
        "Person(?p), age(?p, ?a), swrlb:greaterThan(?a, 60) -> OldPerson(?p)",
        namespaces=[onto, swrlb_onto],
    )

sync_reasoner_pellet(infer_property_values=True, infer_data_property_values=True)
```

**Problèmes résolus et mécanismes internes :**

#### Erreur 1 — `ValueError: Cannot find entity 'swrlb:greaterThan'`

`rule.set_as_rule()` parse la chaîne atome par atome. Quand il rencontre `swrlb:greaterThan`, il cherche un namespace dont le `.name` vaut `"swrlb"`. Si aucun namespace `swrlb` n'est enregistré dans le monde OWL, il lève `ValueError`.

**Correction en 3 étapes :**
1. Créer l'ontologie SWRLB : `get_ontology("http://www.w3.org/2003/11/swrlb")` — OWLReady2 dérive automatiquement le nom `"swrlb"` du dernier segment de l'IRI.
2. Déclarer `greaterThan` comme `AnnotationProperty` dans ce namespace — OWLReady2 traite les `AnnotationProperty` comme des `BuiltinAtom` (la branche correcte dans `rule.py`) plutôt que comme une ClassAtom ou ObjectPropertyAtom.
3. Passer `namespaces=[onto, swrlb_onto]` à `set_as_rule()` pour que les deux namespaces soient consultés lors de la résolution des noms.

#### Problème 2 — Aucun individu dans family.owl

La version standard de `family.owl` (Protégé tutorial) ne contient que des classes et des propriétés, sans instances. Comme la règle SWRL porte sur des individus (`Person(?p)`), le raisonneur ne détecterait rien.

**Correction :** si `list(onto.individuals())` est vide, on crée 6 individus synthétiques avec des âges variés. Résultat attendu après raisonnement : Carol (65), Dave (70) et Frank (69) sont classés `OldPerson`.

#### Mécanisme de Pellet

`sync_reasoner_pellet()` :
- Lance un processus Java avec le JAR Pellet inclus dans owlready2
- Sérialise le graphe OWL en OWL/XML temporaire
- Exécute le raisonnement SWRL + classification OWL
- Réimporte les assertions inférées dans le graphe Python

**Résultat attendu :**

| Individu | Âge | OldPerson inféré |
|---|---|---|
| Alice | 25 | — |
| Bob | 40 | — |
| Carol | 65 | ✓ |
| Dave | 70 | ✓ |
| Eve | 10 | — |
| Frank | 69 | ✓ |

---

## Partie 2 — Knowledge Graph Embedding (KGE)

### Cellule 68 — En-tête Partie 2 *(Markdown)*

Présente le framework **PyKEEN** et les modèles **TransE** et **DistMult** qui seront entraînés sur `automotive_kb.nt`.

---

### Cellule 69 — Installation des dépendances KGE *(Python)*

```python
%pip install pykeen torch --quiet
%pip install scikit-learn matplotlib seaborn --quiet
```

| Package | Rôle |
|---|---|
| `pykeen` | Framework KGE : modèles, pipeline, évaluation filtrée |
| `torch` | Backend tenseur (CPU suffisant pour cette KB) |
| `scikit-learn` | t-SNE pour la visualisation des embeddings |
| `matplotlib` / `seaborn` | Graphiques (sensibilité, t-SNE) |

---

### Cellule 70 — En-tête Section 1 *(Markdown)*

---

### Cellule 71 — Audit de la KB & Filtrage URI-only *(Python)*

```python
from rdflib import Graph, URIRef
g_kb = Graph()
g_kb.parse("automotive_kb.nt", format="nt")

# Ne garder que les triples (URIRef, URIRef, URIRef)
uri_triples: set[tuple[str, str, str]] = set()
for s, p, o in g_kb:
    if isinstance(s, URIRef) and isinstance(p, URIRef) and isinstance(o, URIRef):
        uri_triples.add((str(s), str(p), str(o)))
```

**Ce que fait cette cellule :**

1. Charge `automotive_kb.nt` (~125 884 triples) dans rdflib.
2. **Filtre URI-only** : les modèles KGE (TransE, DistMult) ne peuvent travailler qu'avec des IRIs comme sujet, prédicat *et* objet. Les triples dont l'objet est un **Literal** (dates, chaînes, nombres) sont supprimés car ils ne correspondent pas à des entités dans l'espace d'embedding.
3. Compte les entités uniques (`subjects ∪ objects`) et les prédicats uniques.
4. Affiche les 15 prédicats les plus fréquents.

**Exemple de triples supprimés :**
- `(wd:Q40993, wdt:P571, "1948-01-01"^^xsd:dateTime)` — date de création de Porsche (Literal → supprimé)
- `(wd:Q40993, wdt:P856, "https://..."^^xsd:anyURI)` — URL officielle (Literal → supprimé)

**Constat sur la KB :** le nombre de prédicats après filtrage est ~20–30 (inférieur à la cible 50–200 du lab), en raison de la stratégie d'expansion contrôlée de la Session 2 qui ne ciblait que 8 propriétés Wikidata.

---

### Cellule 72 — En-tête Step 2.2 *(Markdown)*

Explique la stratégie de split et la **garantie de couverture d'entités** : toute entité apparaissant en validation ou test doit être présente en entraînement (évite les entités non vues durant l'évaluation filtrée).

---

### Cellule 73 — Split Train / Validation / Test (80/10/10) *(Python)*

```python
def write_triples(path, triples):
    with open(path, "w", encoding="utf-8") as f:
        for s, p, o in triples:
            f.write(f"{s}\t{p}\t{o}\n")

def ensure_entity_coverage(split, train, train_ents):
    keep, move = [], []
    for t in split:
        s, _, o = t
        if s in train_ents and o in train_ents:
            keep.append(t)
        else:
            move.append(t)  # déplace vers train
            train_ents.add(s); train_ents.add(o)
    train.extend(move)
    return keep

triples_list = list(uri_triples)
random.shuffle(triples_list)   # seed=42 pour reproductibilité
n_train = int(0.8 * n)
n_valid = int(0.1 * n)
```

**Ce que fait cette cellule :**

1. **Mélange aléatoire** avec `random.seed(42)` pour la reproductibilité.
2. **Découpe initiale 80/10/10** par indexation directe.
3. **Garantie de couverture :** la fonction `ensure_entity_coverage()` parcourt la validation et le test. Tout triple dont le sujet ou l'objet n'apparaît pas dans l'ensemble d'entraînement est **déplacé vers train** (plutôt que supprimé), ce qui évite le biais de suppression et garantit un vocabulaire d'entités partagé.
4. **Sanity check :** vérifie que le nombre d'entités non vues dans valid/test est zéro après correction.
5. Écrit trois fichiers TSV : `train.txt`, `valid.txt`, `test.txt` (format : `sujet\tprédicat\tobjet`).

**Pourquoi ce format ?** PyKEEN accepte des fichiers texte avec ce séparateur via `TriplesFactory.from_path()`.

---

### Cellule 74 — En-tête Section 2 *(Markdown)*

Présente les hyperparamètres utilisés pour tous les modèles :

| Paramètre | Valeur |
|---|---|
| Dimension d'embedding | 100 |
| Learning rate | 0.01 |
| Batch size | 256 |
| Epochs | 100 |
| Échantillonnage négatif | Basic (uniforme) |
| Évaluation | Filtrée (vrais positifs connus exclus du ranking) |

---

### Cellule 75 — Vocabulaire partagé (TriplesFactory) & Configuration *(Python)*

```python
from pykeen.triples import TriplesFactory
from pykeen.pipeline import pipeline

tf_training = TriplesFactory.from_path("train.txt", delimiter="\t")
tf_valid    = TriplesFactory.from_path("valid.txt", delimiter="\t",
                entity_to_id=tf_training.entity_to_id,
                relation_to_id=tf_training.relation_to_id)
tf_test     = TriplesFactory.from_path("test.txt", delimiter="\t",
                entity_to_id=tf_training.entity_to_id,
                relation_to_id=tf_training.relation_to_id)

COMMON_CONFIG = dict(
    training=tf_training, validation=tf_valid, testing=tf_test,
    optimizer="Adam", optimizer_kwargs=dict(lr=0.01),
    training_kwargs=dict(num_epochs=100, batch_size=256),
    negative_sampler="basic",
    evaluator_kwargs=dict(filtered=True),
    random_seed=42, device="cpu",
)
```

**Ce que fait cette cellule :**

1. **TriplesFactory** : parse les fichiers TSV et construit deux dictionnaires bijectifs — `entity_to_id` (URI → entier) et `relation_to_id` (URI → entier). Les indices entiers sont utilisés comme indices de lignes dans les matrices d'embedding.
2. **Vocabulaire partagé** : `tf_valid` et `tf_test` reçoivent *les mêmes* `entity_to_id` et `relation_to_id` que `tf_training`. Cela garantit que les mêmes entités ont les mêmes indices dans les trois splits — condition nécessaire pour l'évaluation filtrée.
3. **`COMMON_CONFIG`** : dictionnaire de configuration partagé entre TransE et DistMult pour un benchmarking équitable.
4. **`get_metrics(result)`** : helper robuste qui tente deux API PyKEEN (`get_metric()` ≥ 1.3 ou `to_df()` fallback) pour extraire MRR, H@1, H@3, H@10 dans le cadre filtré + `"both"` (head & tail).

**Évaluation filtrée :** lors du calcul du rang d'un triple de test `(h, r, t)`, tous les autres vrais triples connus du KB qui partagent `(h, r, ?)` sont **masqués** dans le classement. Cela évite de pénaliser le modèle pour avoir prédit des faits vrais mais absents du test set.

---

### Cellule 76 — En-tête TransE *(Markdown)*

Rappelle la formulation TransE : `h + r ≈ t` (la relation est un vecteur de translation). Simple et efficace, mais ne peut pas modéliser relations symétriques ni relations 1-à-N.

---

### Cellule 77 — Entraînement TransE *(Python)*

```python
result_transe = pipeline(
    **COMMON_CONFIG,
    model="TransE",
    model_kwargs=dict(embedding_dim=100),
)
metrics_transe = get_metrics(result_transe)
result_transe.save_to_directory("models/transe")
```

**Ce que fait cette cellule :**

- Lance le pipeline complet PyKEEN : initialisation des embeddings (aléatoire), boucle d'entraînement Adam (100 epochs × batches de 256), évaluation sur `tf_test` à la fin.
- Chaque batch génère des triplets négatifs par **negative sampling basique** : remplace aléatoirement la tête ou la queue par une entité tirée uniformément du vocabulaire.
- La fonction de perte est la **margin ranking loss** ou la **NSSA loss** selon la version PyKEEN (contrôlée en interne).
- `save_to_directory("models/transe")` sérialise : poids du modèle PyTorch (`trained_model.pkl`), configuration, métriques d'entraînement.

---

### Cellule 78 — En-tête DistMult *(Markdown)*

Rappelle la formulation DistMult : `score(h, r, t) = hᵀ diag(r) t`. Symétrique par construction (le score de `(h, r, t)` = celui de `(t, r, h)`) — ne peut pas modéliser des relations antisymétriques comme `ownBy` vs `subsidiary`.

---

### Cellule 79 — Entraînement DistMult *(Python)*

```python
result_distmult = pipeline(
    **COMMON_CONFIG,
    model="DistMult",
    model_kwargs=dict(embedding_dim=100),
)
result_distmult.save_to_directory("models/distmult")
```

Identique à la cellule 77 mais avec `model="DistMult"`. Le pipeline utilise la même configuration (`COMMON_CONFIG`) pour garantir la comparabilité. Sauvegarde dans `models/distmult/`.

---

### Cellule 80 — En-tête Section 3 *(Markdown)*

Annonce le protocole d'évaluation : métriques filtrées, head & tail prédits.

---

### Cellule 81 — Tableau de comparaison des modèles *(Python)*

```python
rows = [
    {"Model": "TransE",   **metrics_transe},
    {"Model": "DistMult", **metrics_distmult},
]
df_eval = pd.DataFrame(rows).set_index("Model").round(4)
print(df_eval.to_string())
```

**Ce que fait cette cellule :**

Construit un tableau Pandas `4 colonnes × 2 lignes` :

| Modèle | MRR | H@1 | H@3 | H@10 |
|---|---|---|---|---|
| TransE | — | — | — | — |
| DistMult | — | — | — | — |

Puis calcule (si disponible) la décomposition **head vs tail** séparément, pour identifier si le modèle prédit mieux la queue ou la tête d'un triple.

**Interprétation des métriques :**
- **MRR** (Mean Reciprocal Rank) : moyenne de `1/rang` pour chaque triple de test. Plus proche de 1 = meilleur.
- **H@K** (Hits@K) : fraction des triples de test pour lesquels l'entité correcte est dans le top-K du classement. H@10 > MRR typiquement.

---

### Cellule 82 — En-tête Section 4 *(Markdown)*

Décrit l'expérience de sensibilité à la taille de la KB.

---

### Cellule 83 — Sensibilité KB Size *(Python)*

```python
SIZES = [20_000, 50_000, len(train_triples)]

for target_size in SIZES:
    sub_train = random.sample(train_triples, target_size)  # ou full
    # Filtrer valid/test aux entités connues du sous-train
    sub_ent = tf_sub.entity_to_id
    valid_sub = [(s,p,o) for s,p,o in valid_triples if s in sub_ent and o in sub_ent]
    # Entraîner TransE (50 epochs pour économiser le temps)
    res = pipeline(..., training_kwargs=dict(num_epochs=50, ...))
```

**Ce que fait cette cellule :**

1. Sous-échantillonne le train à 20k, 50k, et la taille complète.
2. **Re-filtre** validation et test pour ne garder que les triples dont les entités sont connues du sous-vocabulaire du sous-train (condition nécessaire pour l'évaluation filtrée).
3. Entraîne TransE sur chaque sous-ensemble avec seulement **50 epochs** (moitié moins que le modèle principal) pour limiter le temps de calcul.
4. Collecte MRR et H@10 pour chaque taille dans `df_sens`.
5. Trace une courbe à 2 panneaux (MRR vs taille, H@10 vs taille) et sauvegarde `sensitivity_plot.png`.

**Résultat attendu :** MRR et H@10 augmentent avec la taille du KB (plus de données = meilleur apprentissage des structures relationnelles), avec une tendance à plafonner.

---

### Cellule 84 — En-tête Section 5 *(Markdown)*

Annonce les 3 analyses d'embedding : nearest neighbors, t-SNE, et analyse des vecteurs de relations.

---

### Cellule 85 — Nearest Neighbors dans l'espace embedding *(Python)*

```python
entity_repr = result_transe.model.entity_representations[0]
emb_matrix  = entity_repr(indices=None).detach().cpu().float().numpy()  # (N, 100)

def nearest_neighbors(entity_uri, top_k=5):
    idx = entity_to_id[entity_uri]
    sims = cos_sim[idx]
    top_indices = np.argsort(-sims)[1:top_k+1]   # exclut soi-même (rang 0)
    return [(id_to_entity[i], float(sims[i])) for i in top_indices]

ANCHORS = {
    "Porsche":      "http://www.wikidata.org/entity/Q40993",
    "Volkswagen":   "http://www.wikidata.org/entity/Q246",
    "Oliver Blume": "http://automotive-crisis.org/kb/Oliver_Blume",
    "Germany":      "http://www.wikidata.org/entity/Q183",
    "Stellantis":   "http://www.wikidata.org/entity/Q98539614",
}
```

**Ce que fait cette cellule :**

1. Extrait la **matrice d'embedding** complète en appelant `entity_representations[0](indices=None)` — appel PyKEEN pour obtenir tous les vecteurs en une passe.
2. Calcule la **matrice de similarité cosinus** complète `(N × N)` via produit matriciel après normalisation L2.
3. Pour chaque entité ancre, trie les similarités (hors diagonale) et retourne les 5 voisins les plus proches.
4. Tente un **partial match** : si l'URI exacte n'est pas dans le vocabulaire, cherche une URI contenant la sous-chaîne.

**Interprétation attendue :**
- Les voisins de **Porsche** devraient inclure d'autres constructeurs automobiles (Volkswagen, BMW…) si les embeddings ont capturé la structure du graphe.
- Les voisins de **Germany** (pays) devraient inclure d'autres URIs de pays ou d'entités géographiques.
- Des voisins incohérents indiqueraient que le modèle n'a pas suffisamment vu ces entités dans des contextes relationnels variés.

---

### Cellule 86 — En-tête t-SNE *(Markdown)*

Explique le schéma de couleurs : vert = entités privées KB (org/person/GPE), bleu = entités Wikidata.

---

### Cellule 87 — Visualisation t-SNE *(Python)*

```python
from sklearn.manifold import TSNE

MAX_TSNE = 3000
sample_idx = np.random.default_rng(42).choice(n_ent, size=min(MAX_TSNE, n_ent), replace=False)
emb_sample = emb_matrix[sample_idx]

# Colorier les entités par namespace/classe ontologique
typed_org    = {str(s) for s,p,o in g_priv_check if str(o).endswith("Organization")}
typed_person = {str(s) for s,p,o in g_priv_check if str(o).endswith("Person")}
typed_gpe    = {str(s) for s,p,o in g_priv_check if str(o).endswith("GeopoliticalEntity")}

tsne = TSNE(n_components=2, perplexity=30, n_iter=1000, random_state=42, n_jobs=-1)
coords = tsne.fit_transform(emb_sample)
```

**Ce que fait cette cellule :**

1. **Sous-échantillonnage à 3 000 entités** : t-SNE a une complexité mémoire O(N²) — sur 100k entités ce serait ~80 GB. 3 000 est un compromis raisonnable visualisation/mémoire.
2. **Coloration par provenance :**
   - Rouge `#e74c3c` : organisations de la KB privée (`AUTO:Organization`)
   - Violet `#9b59b6` : personnes de la KB privée (`AUTO:Person`)
   - Vert `#2ecc71` : entités géopolitiques (`AUTO:GeopoliticalEntity`)
   - Orange `#f39c12` : autres entités privées
   - Bleu `#3498db` : entités Wikidata (`wd:` namespace)
3. Charge `automotive_kb_private.nt` et `ontology.ttl` pour classer les URIs.
4. Exécute `sklearn.manifold.TSNE` avec `perplexity=30` (standard pour des groupes de ~100 points) et 1000 itérations.
5. Trace le nuage de points et sauvegarde `tsne_plot.png`.

**Résultat attendu :** si les embeddings captent bien la structure, les entités de la KB privée (organisations, personnes) devraient former des sous-groupes distincts des entités Wikidata. La présence du cas Nissan (Q270195 = commune française) pourrait créer un outlier visible dans la visualisation.

---

### Cellule 88 — En-tête Analyse des Relations *(Markdown)*

Présente les trois tests théoriques sur les vecteurs de relations :

| Type de relation | Comportement TransE attendu | Comportement DistMult attendu |
|---|---|---|
| **Symétrique** | `r ≈ -r` → seulement si `r ≈ 0` | Gère naturellement (score symétrique) |
| **Inverse** (ownBy ↔ subsidiary) | `r₁ ≈ -r₂` | Ne peut pas modéliser |
| **Composition** | `r₁ + r₂ ≈ r₃` | Non capturé |

---

### Cellule 89 — Analyse des Vecteurs de Relations *(Python)*

```python
rel_repr = result_transe.model.relation_representations[0]
rel_matrix = rel_repr(indices=None).detach().cpu().float().numpy()  # (R, 100)

# Test 1 — Symétrie : une relation symétrique devrait avoir ||r|| ≈ 0 dans TransE
for rsubstr in ["P31", "P17", "P127", "P355", "ownBy", "subsidiary"]:
    norm = np.linalg.norm(rel_vec(rsubstr))
    cos_neg = cosine(v, -v)
    print(f"{rsubstr}: ||r||={norm:.4f}  cos(r,-r)={cos_neg:.4f}")

# Test 2 — Paire inverse : P127 (owned by) vs P355 (subsidiary)
cos_inv  = cosine(v_p127, -v_p355)   # attendu élevé si inverses
cos_same = cosine(v_p127,  v_p355)   # attendu faible

# Test 3 — DistMult : normes de ses vecteurs de relations
```

**Ce que fait cette cellule :**

1. **Test de symétrie :** dans TransE, une relation symétrique `r` devrait vérifier `h + r ≈ t` *et* `t + r ≈ h` simultanément, ce qui implique `r ≈ 0`. Une norme `||r||` faible est donc un indicateur de symétrie apprise. `cos(r, -r)` vaut toujours `-1` pour tout vecteur non nul — cet affichage sert à confirmer qu'aucune relation n'a été collapsée à zéro.

2. **Test de paire inverse :** si `P127 (owned by)` et `P355 (subsidiary)` sont inverses, TransE devrait apprendre `r_{P127} ≈ -r_{P355}`, c'est-à-dire `cos(r_{P127}, -r_{P355}) ≈ 1`. Un score élevé validerait que le modèle a capturé l'antisymétrie implicite dans la KB.

3. **Normes DistMult :** DistMult représente chaque relation par un vecteur diagonal. Des normes élevées indiquent une forte contribution de cette relation au score. Comparé à TransE (qui encode une direction), DistMult encode une *magnitude* par dimension.

---

### Cellule 90 — En-tête Section 6 *(Markdown)*

Introduit l'objectif : comparer le raisonnement symbolique SWRL avec le raisonnement analogique dans l'espace d'embedding.

**Règle choisie :**
```
Organization(?o) ∧ subsidiary(?o, ?s) → ownBy(?s, ?o)
```
En clair : *si une organisation ?o a ?s comme filiale, alors ?s est possédée par ?o.*

---

### Cellule 91 — En-tête Step 6.1 *(Markdown)*

---

### Cellule 92 — Chargement de la KB Automotive dans OWLReady2 *(Python)*

```python
from owlready2 import *

auto_world = World()   # Monde OWL séparé pour éviter les conflits avec family.owl
auto_onto = auto_world.get_ontology(
    pathlib.Path("ontology.ttl").resolve().as_uri()
).load()
auto_world.get_ontology(
    pathlib.Path("automotive_kb_private.nt").resolve().as_uri()
).load()
```

**Ce que fait cette cellule :**

1. Crée un **nouveau World** OWLReady2 (`auto_world`) complètement isolé du monde `onto` (family.owl) chargé en cellule 65. Cela évite toute contamination entre les deux ontologies.
2. Charge d'abord `ontology.ttl` (déclarations des classes et propriétés du schéma) dans `auto_world`.
3. Charge ensuite `automotive_kb_private.nt` (instancess et triples de données) dans le même `auto_world`.
4. Affiche les classes, object properties, et data properties trouvées, ainsi que le nombre d'individus.

> **Note :** `automotive_kb.nt` (125k triples) n'est *pas* chargé ici — Pellet serait extrêmement lent sur un graphe de cette taille. On travaille avec la KB privée (898 triples) et l'ontologie RDF seulement.

---

### Cellule 93 — Règle SWRL Automotive + Pellet *(Python)*

```python
with auto_onto:
    # Fallback : déclarer les classes/propriétés si non résolues par IRI
    Organization = auto_world.search_one(iri=f"{auto_ns}Organization") or \
                   auto_world.search_one(iri="*Organization")
    subsidiary_p = auto_world.search_one(iri=f"{auto_ns}subsidiary")
    ownBy_p      = auto_world.search_one(iri=f"{auto_ns}ownBy")

    # Si IRI search échoue, déclaration inline
    if Organization is None or subsidiary_p is None or ownBy_p is None:
        class Organization(Thing): namespace = auto_onto
        class subsidiary(ObjectProperty): namespace = auto_onto
        class ownBy(ObjectProperty): namespace = auto_onto

    auto_rule = Imp(world=auto_world)
    auto_rule.set_as_rule("Organization(?o), subsidiary(?o, ?s) -> ownBy(?s, ?o)")

sync_reasoner_pellet(x=auto_world, infer_property_values=True)
```

**Ce que fait cette cellule :**

1. **Résolution des classes/propriétés :** `auto_world.search_one(iri=...)` cherche dans le monde les entités correspondant aux IRIs du namespace automotive. Si la résolution échoue (ce qui arrive quand l'ontologie N-Triples ne sérialise pas les déclarations OWL explicitement), les classes sont déclarées inline.

2. **Règle SWRL :** sans recours à `swrlb:greaterThan` (pas de built-in nécessaire ici), la règle est une simple implication objet-propriété : `Organization(?o), subsidiary(?o, ?s) → ownBy(?s, ?o)`. Ce type de règle est directement supporté par Pellet sans namespace supplémentaire.

3. **Exécution Pellet :** `sync_reasoner_pellet(x=auto_world, ...)` cible explicitement `auto_world` pour éviter d'amalgamer avec le monde family.owl.

4. **Collecte des inférences :** après raisonnement, itère sur tous les individus du monde et collecte les valeurs `ownBy`. Ces valeurs ont été inférées par Pellet (elles n'existaient pas dans les données brutes).

**Exemple concret :**
Si la KB privée contient `(Volkswagen, subsidiary, Porsche)` et `Volkswagen rdf:type Organization`, Pellet infère `(Porsche, ownBy, Volkswagen)`.

---

### Cellule 94 — En-tête Step 6.2 *(Markdown)*

---

### Cellule 95 — Analogie Vectorielle *(Python)*

```python
v_subsidiary = get_rel_vec("subsidiary", rel_matrix, rel_to_id)
v_ownby      = get_rel_vec("ownBy",      rel_matrix, rel_to_id)
v_p355       = get_rel_vec("P355",       rel_matrix, rel_to_id)  # wd:subsidiary
v_p127       = get_rel_vec("P127",       rel_matrix, rel_to_id)  # wd:owned by

# Test : cos(r_P355, -r_P127) → élevé si inverses
cos_inv_wdt = cosine(v_p355, -v_p127)

# Analogie composée : vec(P355) + vec(Org_centroid) ≈ vec(P127) ?
org_centroid = emb_matrix[org_indices].mean(axis=0)
composed = v_p355 + org_centroid
cos_compose = cosine(composed, v_p127)
```

**Ce que fait cette cellule :**

1. **Récupération des vecteurs de relations :** cherche les vecteurs de `subsidiary` (privé et Wikidata P355) et `ownBy` (privé et Wikidata P127) par sous-chaîne dans les URIs.

2. **Test d'inversion relationnel :** calcule `cos(r_subsidiary, -r_ownBy)`. Si TransE a capturé la règle SWRL équivalente (subsidiary et ownBy sont inverses), ce cosinus devrait être proche de 1. C'est la version vectorielle analogique de la règle symbolique.

3. **Centroïde des organisations :** calcule la moyenne des vecteurs d'embedding de 8 entités connues de type `Organization` (Porsche, Volkswagen, BMW, etc.) pour créer un vecteur "Organisation typique".

4. **Analogie composée :** teste si `vec(P355) + vec(Org_centroid) ≈ vec(P127)`, à l'image de l'analogie célèbre `vec(king) - vec(man) + vec(woman) ≈ vec(queen)`.

**Résultat attendu :**
- `cos(P355, -P127)` élevé (> 0.3) si le modèle a vu suffisamment de triples `subsidiary` et `owned by` pour apprendre leur opposition directionnelle.
- Sur une KB dominée par P31/P17/P452, ce signal peut être faible — la réflexion critique (cellule 96) le discute.

---

### Cellule 96 — Réflexion Critique *(Markdown)*

La dernière cellule est documentaire et analyse 5 aspects critiques de l'implémentation :

#### 7.1 — Impact de la qualité de l'alignement des prédicats
~20–30 prédicats uniques dans la KB (cible : 50–200). La cause directe : l'expansion contrôlée en Session 2 ne ciblait que 8 propriétés Wikidata. Les prédicats rares (≤ 5 triples, ex. `ownBy`, `launch`) auront des vecteurs d'embedding mal entraînés.

#### 7.2 — Impact du bruit dans l'expansion de la KB
Deux erreurs d'alignement de Session 2 se propagent :
- **Nissan → Q270195** (Nissan-lez-Enserune, commune française) : ~500 triples sur un village français injectés dans une KB automobile. Crée un cluster aberrant dans t-SNE.
- **Audi** non aligné (confiance 0.6 < seuil 0.7) : aucune expansion Wikidata. Audi sera un nœud quasi-isolé dans l'espace embedding.

#### 7.3 — Open-World vs. Closed-World Assumption
Les modèles KGE supposent le **Closed-World Assumption (CWA)** : tout triple absent est supposé *faux* lors du negative sampling. RDF/OWL utilise l'**Open-World Assumption (OWA)** : l'absence ne signifie pas la fausseté. Conséquence : un triple vrai-mais-absent (ex. `Audi ownBy Volkswagen`) pourrait être pénalisé à l'évaluation.

#### 7.4 — Choix de modélisation ontologique
- Seules 3 classes sur 6 sont peuplées (Organization, Person, GeopoliticalEntity) — CarManufacturer et CarBrand restent vides.
- Le nettoyage par degré < 2 a supprimé des entités légitimes mentionnées une seule fois dans le corpus.

#### 7.5 — Tableau récapitulatif des facteurs de qualité

| Facteur | Impact |
|---|---|
| ~80% des triples proviennent de 3 prédicats (P31, P17, P452) | Embeddings fiables pour ces prédicats, faibles pour les autres |
| Nissan mal aligné | Cluster aberrant, voisins incohérents |
| ~20–30 prédicats (au lieu de 50–200 recommandés) | Espace des relations sous-spécifié |
| Expansion 2-hop limitée à 3 entités | Sous-graphe Porsche/VW/Stellantis plus dense → meilleurs embeddings |
| CWA vs OWA | Pénalise les vrais positifs absents du KB |
| 100 epochs, lr=0.01 | TransE converge ; DistMult peut nécessiter plus d'epochs ou de régularisation |

---

## Fichiers produits par la Session 3

| Fichier | Description |
|---|---|
| `train.txt` | 80% des triples URI-only, format TSV (sujet\tprédicat\tobjet) |
| `valid.txt` | 10% des triples, couverture d'entités garantie |
| `test.txt` | 10% des triples, couverture d'entités garantie |
| `models/transe/` | Poids TransE sérialisés (PyKEEN `save_to_directory`) |
| `models/distmult/` | Poids DistMult sérialisés |
| `sensitivity_plot.png` | Courbe MRR/H@10 en fonction de la taille de la KB |
| `tsne_plot.png` | Nuage t-SNE des entités colorées par classe ontologique |

---

## Résumé des choix techniques

| Choix | Justification |
|---|---|
| `file://C:/...` (2 slashes) pour OWLReady2 sur Windows | `as_uri()` donne 3 slashes → chemin `/C:/...` invalide. 2 slashes → chemin `C:/...` valide. |
| `AnnotationProperty` pour `swrlb:greaterThan` | OWLReady2 crée un `BuiltinAtom` (pas une ClassAtom/ObjectPropertyAtom) pour les `AnnotationProperty` — comportement correct pour les built-ins SWRL |
| World séparé (`auto_world`) pour la KB automotive | Évite la contamination des namespaces entre `family.owl` et la KB automotive |
| Dimension 100, 100 epochs, Adam lr=0.01 | Hyperparamètres standard pour TransE/DistMult sur KB de taille moyenne (~50–150k triples) |
| Évaluation filtrée | Protocole standard KGE; exclut les vrais positifs connus du ranking pour éviter la sous-évaluation |
| t-SNE limité à 3 000 entités | t-SNE a complexité O(N²) en mémoire ; 3 000 × 100-dim ≈ 2.4 MB de matrice de similarité, gérable en RAM |
| `ensure_entity_coverage()` dans le split | Garantit aucune entité non vue en validation/test → évaluation filtrée propre |
