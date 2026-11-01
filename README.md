# Catalogue de cartes Krakened

Catalogues de cartes publiés pour l'application [Krakened](https://github.com/krakenedcards) (deckbuilder Magic: The Gathering et Altered), téléchargés à la demande par l'application (ADR-0007 du dépôt de l'application).

**Portée : test fermé.** Ces catalogues sont destinés aux testeurs de Krakened ; le dépôt étant public, les fichiers restent techniquement téléchargeables par tous. Aucune image n'est publiée.

## Contenu

- Une publication (*release*) par version et par jeu, nommée `<jeu>-<date des données>-r<run>-a<tentative>` :
  - `manifest.json` : format 2 du paquet Krakened (version, portée, sources avec origine, date de collecte et conditions, taille, SHA-256 et nombre de cartes du fichier) ;
  - `manifest.json.sig` : signature Ed25519 du manifeste (128 caractères hexadécimaux) ;
  - `magic.jsonl.gz` ou `altered.jsonl.gz` : une fiche canonique par ligne.
- `index/<jeu>.json` : version publiée la plus récente du jeu (`catalog_version`, `release_tag`, `manifest_sha256`, `cards`). Une publication est refusée si le nombre de cartes baisse de plus de 5 % (variable `CATALOG_MAX_DROP_PERCENT`).

L'application vérifiera (tâche T-45) la signature avec la clé publique qu'elle embarque, puis la taille et le SHA-256 du fichier, et refusera un catalogue aux données plus anciennes que celui qu'elle a déjà.

## Sources et conditions

| Jeu | Source | Conditions |
|---|---|---|
| Magic | Export `all_cards` de [Scryfall](https://scryfall.com/docs/api), réduit aux impressions `en` et `fr` | Textes des cartes © Wizards of the Coast. Krakened n'est ni produit ni approuvé par Wizards of the Coast. |
| Altered | API communautaire [Altered Core](https://cards.alteredcore.org/api) (non affiliée à Equinox) ; cartes Uniques exclues | Données des cartes © Equinox. Krakened n'est pas affilié à Equinox. |

## Publier

Onglet *Actions*, workflow **Publish the Magic catalog** ou **Publish the Altered catalog**, *Run workflow*. Publication uniquement à la demande. Les étapes communes sont dans `.github/workflows/publish.yml`.

Secrets du dépôt :

- `CATALOG_SIGNING_KEY` : graine Ed25519 de 32 octets en hexadécimal (`openssl rand -hex 32`), jamais versionnée ; la clé publique correspondante est versionnée dans l'application (`apps/krakened/src-tauri/catalog/catalog-signing.pub`) ;
- `KRAKENED_DEPLOY_KEY` : clé SSH privée d'une **clé de déploiement en lecture seule** du dépôt de l'application (outils de préparation et de signature). Créer la paire avec `ssh-keygen -t ed25519 -N "" -f krakened-deploy`, ajouter `krakened-deploy.pub` dans *Settings > Deploy keys* du dépôt de l'application (sans accès en écriture), et le contenu de `krakened-deploy` dans ce secret.

Les publications ne partent que de `main`. Les journaux des workflows sont publics. Le dépôt étant public, les catalogues publiés sont accessibles à tous ; cette exposition est couverte par la décision SRC-006 du responsable (2026-11-01).

### Changer de clé de signature

1. Nouvelle graine : `openssl rand -hex 32`, gardée hors de tout dépôt.
2. Clé publique : `CATALOG_SIGNING_KEY=<graine> cargo run -p krakened-core --bin catalog-sign -- public-key`, versionnée dans `apps/krakened/src-tauri/catalog/catalog-signing.pub` et livrée dans une nouvelle version de l'application.
3. Remplacer le secret `CATALOG_SIGNING_KEY`, puis republier chaque jeu. Les applications encore à l'ancienne clé refusent les nouvelles publications jusqu'à leur mise à jour.

## Vérifier une publication

```sh
cargo run -p krakened-core --bin catalog-sign -- verify manifest.json <clé publique>
sha256sum magic.jsonl.gz   # à comparer à manifest.json
```
