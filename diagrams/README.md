# MCD — Application de gestion de stock / vente

Ce dossier contient le Modèle Conceptuel de Données (diagramme de classes) de l'application, au format [draw.io](https://app.diagrams.net/) : [`mcd-gestion-stock.drawio`](./mcd-gestion-stock.drawio).

Pour l'ouvrir : va sur [app.diagrams.net](https://app.diagrams.net/), choisis **Open Existing Diagram**, et sélectionne le fichier. Tu peux aussi utiliser l'extension VS Code "Draw.io Integration".

## Contexte

C'est le modèle d'une application multi-entreprise (SaaS) de gestion de stock, commandes et ventes. Chaque `Entreprise` a ses propres utilisateurs, articles, clients et fournisseurs.

## Entités et rôle de chacune

| Entité | Rôle |
|---|---|
| **Entreprise** | Le tenant : chaque entreprise cliente de l'application a son propre catalogue, ses utilisateurs, etc. |
| **Utilisateur** | Compte permettant de se connecter à l'espace d'une entreprise (voir `RoleUtilisateur`). |
| **Categorie** | Regroupe les articles par famille (ex. Boissons, Électronique...). |
| **Article** | Le produit vendu/acheté : prix HT, taux de TVA, prix TTC, photo. |
| **MvtStk** | Historique des mouvements de stock d'un article (entrée, sortie, correction). |
| **Fournisseur** | Tiers auprès de qui l'entreprise achète des articles. |
| **CommandeFournisseur** / **LigneCdeFournisseur** | Commande passée à un fournisseur, et le détail des articles/quantités commandés. |
| **Client** | Tiers à qui l'entreprise vend des articles. |
| **CommandeClient** / **LigneCdeClt** | Commande passée par un client, et son détail. |
| **Vente** / **LigneVente** | L'acte de vente effectif (ce qui sort réellement en caisse/facturation), et son détail. |

## Le changement apporté : lien `CommandeClient` → `Vente`

**Avant** : `Vente` n'avait aucun lien avec `CommandeClient`. Une commande client et une vente étaient deux processus totalement indépendants, alors qu'en pratique une vente découle souvent d'une commande validée (le client commande, puis on facture/vend).

**Après** : `Vente` porte désormais une clé étrangère **optionnelle** vers `CommandeClient` (`commandeClient_id`, nullable) :

- `CommandeClient (1) —— (0..*) Vente`

Pourquoi nullable plutôt qu'un lien obligatoire ? Parce que deux cas de figure coexistent dans ce type d'application :

1. **Vente issue d'une commande** : le client commande en ligne ou par téléphone (`CommandeClient`), puis quand elle est traitée on crée la `Vente` correspondante, en la reliant à la commande d'origine.
2. **Vente directe / comptoir** : un client achète sur place sans commande préalable — on crée directement une `Vente` sans commande associée, donc `commandeClient_id = null`.

Une FK obligatoire aurait cassé ce deuxième cas ; une FK nullable couvre les deux sans dupliquer le modèle.

> Repère cette relation dans le diagramme : elle est tracée en **rouge/gras** avec le label `1 — 0..* (NOUVEAU)`.

## Les énumérations ajoutées

Le diagramme d'origine ne modélisait aucun statut ni type — ces champs auraient fini en `String` libre (`"en attente"`, `"En Attente"`, `"EN_ATTENTE"`...), ce qui est une source classique de bugs (comparaisons qui échouent, valeurs invalides en base, pas d'auto-complétion côté code). On a donc introduit 4 énumérations :

| Enum | Valeurs | Utilisée par |
|---|---|---|
| **EtatCommande** | `EN_PREPARATION`, `VALIDEE`, `LIVREE`, `ANNULEE` | `CommandeClient.etat`, `CommandeFournisseur.etat` |
| **TypeMvtStk** | `ENTREE`, `SORTIE`, `CORRECTION_POS`, `CORRECTION_NEG` | `MvtStk.type` |
| **RoleUtilisateur** | `ADMIN`, `GESTIONNAIRE`, `VENDEUR` | `Utilisateur.role` |
| **TypeClient** | `PARTICULIER`, `ENTREPRISE` | `Client.type` |

Ces enums sont représentées en jaune, en pointillés, avec le stéréotype `«enumeration»`, reliées par une flèche pointillée à l'entité qui les utilise (dépendance, pas association).

**Est-ce mieux de les mettre ?** Oui, dans ce contexte, pour trois raisons concrètes :

- **Intégrité des données** : impossible d'enregistrer un état ou un rôle qui n'existe pas.
- **Code plus sûr** : en Java/TypeScript, un `switch` sur un enum est vérifié par le compilateur ; un `switch` sur une `String` ne l'est pas.
- **Évolutivité maîtrisée** : ajouter une valeur (ex. `REMBOURSEE` pour une vente) se fait à un seul endroit, sans risquer d'incohérence entre les différentes couches (BDD, backend, frontend).

`Article` et `Categorie`, en revanche, n'ont pas de statut fixe à énumérer — leurs "types" (nom de catégorie, désignation d'article) sont des données métier libres qui changent au fil de l'exploitation, donc elles restent en `String` / table de référence (`Categorie`) plutôt qu'en enum figée dans le code.

## Mise à jour : emails, notifications de stock, authentification client

Suite aux exigences fonctionnelles (envoi d'email depuis un template, alerte de seuil de stock, connexion obligatoire du client pour commander), le MCD a été complété. Voir aussi [`ARCHITECTURE.md`](./ARCHITECTURE.md) pour les décisions techniques (WebFlux, Kafka, OpenFeign, Keycloak, dashboard) qui ne sont pas du ressort d'un MCD.

### Nouvelles entités

**EmailTemplate** — un modèle de message réutilisable (objet + corps HTML avec placeholders) pour chaque type d'email à envoyer :
- `code: TypeNotification` — quel type de message ce template sert (confirmation commande fournisseur, confirmation commande client, confirmation vente, suivi de commande, alerte stock).
- `entreprise_id` **nullable** (en rouge/gras dans le diagramme, même convention que `Vente.commandeClient_id`) : si `null`, c'est le template par défaut de la plateforme ; une entreprise peut le surcharger avec son propre template.

**Notification** — la trace de chaque email (ou futur SMS) réellement envoyé :
- `canal`, `type`, `statut` : trois enums séparés (voir plus bas).
- `destinataire`, `dateCreation`, `dateEnvoi`.
- `template_id` : quel `EmailTemplate` a servi à générer le contenu.
- `referenceType` + `referenceId` : identifient l'objet à l'origine de la notification (`CommandeClient`, `CommandeFournisseur`, `Vente` ou `Article`), **sans** multiplier les FK optionnelles. C'est une association polymorphe simplifiée — un choix pragmatique pour un MCD (4 FK nullables séparées aurait été plus "propre" en apparence mais plus lourd à maintenir et à faire évoluer si un 5ème type de source apparaît). Documenté par une note jaune directement sur le diagramme.

Ces deux entités et leurs enums forment le **Module Notifications & Emails**, matérialisé par un cadre pointillé turquoise séparé du reste du MCD — c'est un sous-système transverse (il réagit à des événements des autres entités) plutôt qu'une chaîne métier de plus, donc on évite de le mélanger visuellement aux chaînes commande/vente/stock existantes.

### Attributs ajoutés aux entités existantes

| Entité | Attribut ajouté | Pourquoi |
|---|---|---|
| `Article` | `quantiteStock: Integer` | Niveau de stock courant — nécessaire pour savoir quand déclencher une `Notification` de type `ALERTE_STOCK`. |
| `Article` | `seuilAlerte: Integer` | Le seuil configurable en dessous duquel l'alerte se déclenche (propre à chaque article, pas une constante globale). |
| `Utilisateur` | `keycloakId: String` | Fait le lien entre le compte applicatif et l'identité gérée par Keycloak (l'authentification elle-même est déléguée à Keycloak, `motDePasse` reste pour compatibilité/legacy si besoin d'un fallback local). |
| `Client` | `motDePasse: String`, `keycloakId: String`, `compteActif: Boolean` | Le client doit désormais pouvoir se connecter pour passer une commande — il devient un compte authentifiable comme `Utilisateur`, avec son propre statut d'activation. |

### Nouvelles énumérations

| Enum | Valeurs | Utilisée par |
|---|---|---|
| **TypeNotification** | `CONFIRMATION_COMMANDE_FOURNISSEUR`, `CONFIRMATION_COMMANDE_CLIENT`, `CONFIRMATION_VENTE`, `SUIVI_COMMANDE`, `ALERTE_STOCK` | `EmailTemplate.code`, `Notification.type` |
| **CanalNotification** | `EMAIL`, `SMS` | `Notification.canal` — SMS non implémenté au départ, mais l'enum est prête à l'extension sans casser le modèle. |
| **StatutNotification** | `EN_ATTENTE`, `ENVOYEE`, `ECHEC` | `Notification.statut` — permet de rejouer les envois en échec (ex. si le fournisseur SMTP est temporairement indisponible). |

### Règle métier : qui doit se connecter, et quand

- **Consultation du catalogue (liste des articles)** : publique, sans authentification.
- **Passage d'une commande client** (`CommandeClient`) : nécessite que le `Client` soit connecté (`compteActif = true`).
- **Tout ce qui touche à l'espace entreprise** (gestion des articles, commandes fournisseurs, ventes, dashboard) : toujours authentifié via `Utilisateur`.

Cette règle n'est pas un champ du MCD — elle se traduit dans les règles d'autorisation de la gateway/Spring Security (voir `ARCHITECTURE.md`), mais elle explique pourquoi `Client` a maintenant besoin d'un `motDePasse`/`keycloakId` alors qu'avant il n'était qu'une fiche contact.

## Prochaines étapes possibles

- Ajouter un enum `EtatVente` (`PAYEE`, `ANNULEE`, `REMBOURSEE`) si la vente a besoin d'un cycle de vie propre.
- Décider si `MvtStk` doit tracer sa source (`Vente`, `CommandeFournisseur`) pour la traçabilité complète du stock — actuellement seul `Notification.referenceId` fait ce lien, côté notification, pas côté mouvement de stock lui-même.
- Une fois validé, dériver le schéma de base de données (DDL) et les entités JPA / modèles Angular à partir de ce MCD.
