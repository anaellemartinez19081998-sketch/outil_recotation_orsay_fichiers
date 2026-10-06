# Outil de cotation LC : Bibliothèque du musée d'Orsay

Notebook Colab d'aide à la conversion des cotes Orsay de la bibliothèque du musée d'Orsay (BM'O) vers la Library of Congress Classification (LCC).

L'outil propose une cote pour chaque notice, une à la fois via un formulaire ou par lot à partir d'un fichier CSV. Chaque proposition indique sa source et son niveau de confiance, et les cas incertains sont renvoyés à la personne qui catalogue.

## Fonctionnement

Chaque notice passe par une cascade de règles. La première qui donne un résultat l'emporte.

1. **Monographie d'artiste** (lettre Clément B) : cote partielle `NY [ARTISTE] .n`, où `n` est le type d'ouvrage détecté (1 catalogue d'exposition, 2 catalogue raisonné ou de collection, 3 écrits de l'artiste, 4 biographie ou étude critique).
2. **Niveau 1, ISBN** : recherche de la cote réelle dans le catalogue de la Library of Congress (SRU), puis dans Open Library en repli.
3. **Niveau 0, lettre Clément** : la lettre de la cote BM'O donne la grande classe LC (ex. `P` → `ND`, `C` → `PN`). Les lettres hors périmètre (G, H, I, J, M) et les lettres de forme (Q, X, Z) sont signalées pour traitement manuel.
4. **Niveau 3, sous-classe** : un modèle de langage (Mistral) choisit la sous-classe, au choix :
   - dans le **plan réduit BM'O** (cotes validées par la bibliothèque, avec ses consignes) ;
   - dans le **LCC complet** : deux colonnes de propositions, l'une du modèle, l'autre par similarité sémantique (embeddings) avec le barème LC. La personne choisit.

Le cutter auteur est ensuite ajouté (tables BM'O cinéma et littérature, sinon calcul approché d'après la table de Cutter de la LC), puis l'année.

## Prérequis

- Un compte Google pour Google Colab.
- Une clé API Mistral, enregistrée dans les Secrets de Colab sous le nom `MISTRAL_API_KEY` (icône clé dans le panneau de gauche, avec l'accès au notebook activé).

## Installation

Les fichiers de données de ce dépôt doivent se trouver dans le répertoire de travail de la session Colab.

**Option A : cloner le dépôt** (dans une cellule, avant §0)

```
!git clone https://github.com/<utilisateur>/<depot>.git
%cd <depot>
```

**Option B : dépôt manuel** via le panneau Fichiers de Colab.

Attention : les fichiers déposés dans une session Colab sont effacés à la fermeture de la session. Il faut les redéposer (ou recloner) à chaque nouvelle session.

## Fichiers attendus

| Fichier | Contenu | Obligatoire |
|---|---|---|
| `plan_lcc_orsay.csv` | Plan réduit BM'O (colonnes `classe ; groupe ; cote_range ; description ; commentaire ; statut`) | Oui |
| `plan_lcc_officiel_reel_updated.csv` | Barème LC complet extrait des PDF de la Library of Congress (`classe ; cote_range ; description`) | Oui |
| `table_cutter_cinema.csv` | Table de cutters BM'O cinéma (`nom ; cutter`) | Non |
| `table_cutter_litterature.csv` | Table de cutters BM'O littérature (`nom ; identifiant_lc`) | Non |
| `embeddings_bareme_complet.npy` + `embeddings_bareme_complet_index.csv` | Cache des embeddings du barème (les deux fichiers vont ensemble) | Non, recalculé s'il manque |
| `pool_fewshot_bmo.csv` | Notices déjà cotées pour l'option few-shot (`Titre ; Sujets ; Cote_LC`) | Non |


Sans cache d'embeddings, le premier lancement de §4 calcule les vecteurs du barème (environ 48 000 entrées, plus d'une heure). En cas de coupure ou de limite de débit, relancer la cellule : le calcul reprend au dernier lot sauvegardé. Si le barème change, supprimer les deux fichiers de cache.

## Utilisation

1. Exécuter les cellules §0 à §8 dans l'ordre.
2. Dans §3, remplacer `EMAIL_CONTACT` par une adresse de contact réelle (Open Library demande aux applications de s'identifier).
3. Régler les options en §7 (prises en compte au moment du clic, sans relancer) :
   - chercher dans le LCC complet plutôt que dans le plan réduit ;
   - prioriser les sujets RAMEAU sur le titre ;
   - utiliser des exemples few-shot (grisé si le pool est absent) ;
   - calculer le cutter auteur.
4. Coter :
   - **une notice** (§9) : remplir le formulaire et cliquer sur « Coter cette notice ». En plan réduit, les boutons « Autre proposition » et « Voir le LCC complet » permettent de relancer. En LCC complet, un champ « Préciser » permet d'orienter les deux sources ;
   - **un lot** (§10) : importer ou indiquer un fichier CSV, puis cliquer sur « Traiter le fichier ».

### Format du CSV d'entrée

Colonnes attendues : `Cote ; Titre ; Auteur ; Sujet ; ISBN ; Annee`

- `Cote` : cote BM'O, ex. `4 P 72`
- `Auteur` : de préférence `Nom, Prénom`
- `Sujet` : sujets RAMEAU

Les colonnes supplémentaires sont conservées telles quelles dans le fichier de sortie.

### Fichier de sortie

Le fichier `cote_<nom du fichier d'entrée>.csv` reprend les colonnes d'origine et ajoute :

- `cote_lc`, `source`, `a_trancher` pour les résultats directs ;
- `candidat_1` à `candidat_6` et leurs descriptions en mode LCC complet (`a_trancher = oui`) ;
- `cutter`, `cutter_source` si l'option est cochée.

Une erreur sur une notice n'interrompt pas le lot : elle est notée dans la colonne `source`.

## Niveaux de confiance

| Source | Confiance | Remarque |
|---|---|---|
| Niveau 0 (Clément) | 1.0 | Grande classe seulement, jamais une cote complète |
| Niveau 1 (LC SRU / Open Library) | 0.9 | Cote d'une notice existante, déjà complète |
| Niveau 3 (plan réduit) | 0.7 | Proposition du modèle, à valider |
| Branche monographie | 0.5 | Cote partielle, type d'ouvrage à vérifier |
| LCC complet | | Choix humain entre plusieurs candidats |
| Traitement manuel | 0.0 | Lettre hors périmètre ou de forme |

## Limites connues

- L'identifiant artiste des monographies n'est pas encore défini par la BM'O : le placeholder `[ARTISTE]` est à compléter à la main.
- Le cutter calculé est une approximation : à la LC, le numéro final dépend des ouvrages déjà en rayon.
- Les tables de cutters BM'O sont amenées à évoluer.
- Les propositions du modèle ne sont pas vérifiées automatiquement contre le barème.
- Les appels à Mistral sont soumis à des limites de débit ; le traitement d'un gros lot peut être long.

## Sources et crédits

- Table de Cutter de la Library of Congress, *Classification and Shelflisting Manual*, G 63 : https://www.loc.gov/aba/pcc/053/table.html
- Calcul du cutter adapté de [veralvx/csan](https://github.com/veralvx/csan)
- Nettoyage des ISBN inspiré du code de Belvadi (2023), via [isbnlib](https://pypi.org/project/isbnlib/)
- API : Mistral AI (`open-mistral-nemo`, `mistral-embed`), Library of Congress SRU, Open Library

Développé par Anaëlle Martinez dans le cadre d'un stage de master 2 TNAH (École nationale des chartes) à la bibliothèque du musée d'Orsay, 2026.
