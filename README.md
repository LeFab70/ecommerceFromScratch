# Gestion de stock — Spring Boot + Angular

Application multi-entreprise de gestion de stock, commandes fournisseurs/clients et ventes.

## Structure du dépôt

```
.
├── backend/       # API Spring Boot (Java)
├── frontend/      # Application Angular
├── diagrams/      # MCD (draw.io) et documentation de conception
│   ├── mcd-gestion-stock.drawio
│   ├── README.md        # explication détaillée du modèle de données
│   └── ARCHITECTURE.md  # WebFlux, Kafka, OpenFeign, Keycloak, dashboard
└── ressources/    # cahier des charges, maquettes, exports, notes diverses
```

## Où commencer

- Le modèle de données (entités, relations, énumérations) est expliqué dans [`diagrams/README.md`](./diagrams/README.md).
- Les choix techniques (WebFlux, Kafka, OpenFeign, Keycloak, dashboard) sont dans [`diagrams/ARCHITECTURE.md`](./diagrams/ARCHITECTURE.md).
- `backend/` et `frontend/` sont pour l'instant vides : à initialiser (Spring Initializr côté backend, Angular CLI côté frontend).
