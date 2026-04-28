# Vision de déploiement

Le système est déployé sous forme de services conteneurisés, avec un service applicatif par Bounded Context.
Le service `bc-commande` porte l'orchestration du cycle de vie de commande et expose une API REST publique.
Le service `bc-paiement` est isolé pour encapsuler l'ACL vers le PSP et limiter l'impact des dépendances externes.
Le service `bc-stock` gère les réservations et expose des endpoints internes pour les opérations de stock.
Le service `bc-livraison` traite l'expédition, le suivi colis et les statuts de transport.
Le service `bc-retour` gère les retours produits et les demandes de remboursement métier.
Le service `bc-catalogue` fournit les données produits en lecture aux autres contextes.
Un broker de messages (Kafka ou RabbitMQ) est aussi conteneurisé pour les échanges asynchrones inter-contextes.
Chaque service est construit à partir d'une image dédiée, versionnée par tag applicatif (`vX.Y.Z`).
La configuration est externalisée via variables d'environnement : `PORT`, `DB_URL`, `BROKER_URL`, `LOG_LEVEL`, `SERVICE_NAME`.
Les secrets sensibles (`PSP_API_KEY`, mots de passe DB, certificats) ne sont pas dans l'image et sont injectés au runtime.
Chaque conteneur expose `/health` pour le monitoring technique et `/metrics` pour les métriques métier.
Les communications inter-services passent par un réseau Docker privé ; seule l'API Commande est publiée vers l'extérieur.
Les volumes persistants sont réservés aux bases de données et au broker, pas aux services stateless.
En production, la même logique peut être orchestrée par Docker Compose (petite échelle) ou Kubernetes (grande échelle).

## Exemple annoté de Dockerfile (pseudo-code)

```dockerfile
# PSEUDO-CODE - non fonctionnel
# Image multi-stage pour un service de Bounded Context (ex: bc-commande)

FROM runtime-lang:version AS builder
WORKDIR /app

# Copier les manifests et restaurer les dépendances
COPY package-manifest.* ./
RUN install-dependencies

# Copier le code source du contexte puis construire
COPY src/ ./src/
RUN build-application --context=commande --output=/out

FROM runtime-lang:slim AS runtime
WORKDIR /service

# Variables de configuration (injectées au runtime)
ENV SERVICE_NAME=bc-commande
ENV PORT=8080
ENV LOG_LEVEL=info

# Copier uniquement l'artefact de build (image plus légère)
COPY --from=builder /out ./

# Endpoint attendu: /health et /metrics
EXPOSE 8080
CMD ["run-service", "--port", "8080"]
```
