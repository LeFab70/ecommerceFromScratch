# Architecture technique

Ce document couvre les décisions techniques qui ne se modélisent pas dans un MCD (voir [`README.md`](./README.md) pour le modèle de données). Il sert de référence pour l'implémentation du `backend/`.

## Vue d'ensemble

L'application est découpée en services autour des grands domaines métier du MCD :

- **Service Catalogue** : `Article`, `Categorie`, `MvtStk`.
- **Service Commande/Vente** : `CommandeClient`, `CommandeFournisseur`, `LigneCdeClt`, `LigneCdeFournisseur`, `Vente`, `LigneVente`.
- **Service Tiers** : `Client`, `Fournisseur`, `Entreprise`, `Utilisateur`.
- **Service Notification** : `EmailTemplate`, `Notification` — écoute les événements des autres services et envoie les emails.
- **Gateway** : point d'entrée unique, vérifie les jetons Keycloak, route vers les services.

Cette liste est un point de départ raisonnable, pas une contrainte figée — les services peuvent être fusionnés au début (ex. Catalogue + Commande/Vente dans un seul déployable) et scindés plus tard si la charge le justifie.

## Authentification & autorisations (Keycloak + Spring Security)

- **Keycloak** est la source de vérité pour l'identité : un realm dédié, avec deux clients (ou deux rôles) — `entreprise-app` pour les `Utilisateur`, `client-app` pour les `Client`.
- Chaque `Utilisateur` et chaque `Client` porte un `keycloakId` (voir MCD) qui référence l'identité Keycloak ; la base applicative ne stocke jamais le mot de passe en clair (le champ `motDePasse` existant est un vestige/fallback, à supprimer une fois Keycloak pleinement en place).
- **Spring Security (Resource Server / OAuth2)** valide le JWT émis par Keycloak sur chaque requête protégée.

### Règles d'accès

| Endpoint | Authentification requise |
|---|---|
| `GET /api/articles`, `GET /api/articles/{id}`, `GET /api/categories` | Aucune — consultation publique du catalogue |
| `POST /api/commandes-client` (passer une commande) | `Client` connecté (rôle `CLIENT`) |
| `GET /api/commandes-client/{id}` (suivi de commande) | `Client` propriétaire de la commande, ou `Utilisateur` de l'entreprise |
| Tout endpoint `/api/admin/**`, `/api/ventes/**`, `/api/commandes-fournisseur/**`, `/api/dashboard/**` | `Utilisateur` connecté (rôle selon `RoleUtilisateur` : `ADMIN`, `GESTIONNAIRE`, `VENDEUR`) |

Concrètement : les endpoints de lecture du catalogue sont `permitAll()` dans la config Spring Security ; tout le reste exige un JWT valide, avec un filtrage additionnel par rôle via `@PreAuthorize`.

## Trafic client à fort volume (WebFlux)

Le service Catalogue (consultation publique, potentiellement beaucoup de requêtes simultanées sans authentification) est un bon candidat pour **Spring WebFlux** plutôt que Spring MVC classique :

- Endpoints de lecture (`GET /api/articles`) en réactif (`Mono`/`Flux`), backés par R2DBC ou en gardant JPA classique derrière un `Schedulers.boundedElastic()` si une migration complète vers R2DBC n'est pas prioritaire au départ.
- Les services avec moins de trafic public (Commande/Vente, Tiers) peuvent rester en Spring MVC classique — pas besoin de tout réécrire en réactif dès le départ, seul le point d'entrée à fort trafic (consultation catalogue) en a réellement besoin.

## Communication entre services

- **OpenFeign** pour les appels synchrones inter-services quand une réponse immédiate est nécessaire (ex. le service Commande/Vente qui vérifie le stock disponible auprès du service Catalogue avant de valider une `LigneCdeClt`).
- **Kafka** pour la communication asynchrone événementielle, notamment tout ce qui déclenche une notification — le service Notification n'a pas besoin d'appeler activement les autres services, il **consomme des événements** :

### Topics Kafka proposés

| Topic | Producteur | Événement | Déclenche |
|---|---|---|---|
| `commande-fournisseur.creee` | Service Commande/Vente | Une `CommandeFournisseur` est créée | `Notification` type `CONFIRMATION_COMMANDE_FOURNISSEUR` vers le `Fournisseur` |
| `commande-client.creee` | Service Commande/Vente | Une `CommandeClient` est créée | `Notification` type `CONFIRMATION_COMMANDE_CLIENT` vers le `Client` |
| `commande-client.etat-change` | Service Commande/Vente | `CommandeClient.etat` change (ex. `VALIDEE` → `LIVREE`) | `Notification` type `SUIVI_COMMANDE` vers le `Client` |
| `vente.creee` | Service Commande/Vente | Une `Vente` est enregistrée | `Notification` type `CONFIRMATION_VENTE` vers le `Client` (si `commandeClient_id` renseigné, ou vers l'email de vente directe sinon) |
| `stock.seuil-atteint` | Service Catalogue | `Article.quantiteStock <= Article.seuilAlerte` après un `MvtStk` de type `SORTIE` | `Notification` type `ALERTE_STOCK` vers les `Utilisateur` de l'entreprise (rôle `GESTIONNAIRE`/`ADMIN`) |

Le service Catalogue calcule et publie `stock.seuil-atteint` lui-même (il est le seul à connaître `quantiteStock` et `seuilAlerte` en temps réel) plutôt que de faire vérifier ce seuil par un autre service — évite un couplage et une requête réseau à chaque mouvement de stock.

Le service Notification consomme ces 5 topics, résout le bon `EmailTemplate` via `TypeNotification`, génère le contenu, envoie l'email, et écrit une ligne `Notification` avec le `statut` correspondant (`ENVOYEE` ou `ECHEC`, avec possibilité de retry sur `ECHEC`).

## Dashboard

Le dashboard (historiques, graphes) n'introduit pas de nouvelle entité dans le MCD : c'est une couche de lecture/agrégation au-dessus des données existantes (`Vente`, `CommandeClient`, `MvtStk`, `Article`). Deux approches possibles, à trancher selon le volume de données :

1. **Requêtes d'agrégation à la volée** (`GROUP BY` sur `Vente`/`MvtStk` par période) — suffisant tant que le volume reste modeste, le plus simple à maintenir.
2. **Vue matérialisée ou table de statistiques pré-calculée**, rafraîchie périodiquement (batch) ou à chaque événement Kafka pertinent (`vente.creee`, etc.) — à envisager seulement si les requêtes à la volée deviennent trop lentes.

Démarrer avec l'option 1 ; ne pas construire l'option 2 avant d'avoir mesuré un vrai problème de performance.

## Ce qui reste à trancher

- Découpage exact des microservices au démarrage (un seul déployable au début est raisonnable, le découpage par domaine peut se faire progressivement).
- Choix du broker de message pour les retries de `Notification` en `ECHEC` (retry topic Kafka dédié, ou simple job planifié qui rescanne les notifications en échec).
- R2DBC vs JPA+boundedElastic pour la partie WebFlux du service Catalogue.
