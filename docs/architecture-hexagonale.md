# Architecture hexagonale

## Description des couches

### Couche Domain

La couche Domain est le cœur du système : elle contient toute la logique métier pure, sans aucune dépendance vers l'infrastructure ou le monde extérieur. On y trouve les **agrégats** (`Commande` avec `LigneCommande`, `Stock` avec `RéservationStock`, `Colis`), les **objets valeur** (`AdresseLivraison`, `Montant`, `NumeroSuivi`, `ModeLivraison`, `StatutCommande`) ainsi que les **événements de domaine** (`CommandeCréée`, `StockRéservé`, `StockInsuffisant`, `ColisExpédié`, `ColisLivré`). Les règles métier et les invariants y sont appliqués : c'est ici que la Commande vérifie qu'elle contient au moins une `LigneCommande`, que le `MontantTotal` est cohérent, ou que le `StatutCommande` ne peut pas régresser. La couche Domain définit également les **ports** (interfaces abstraites) dont elle a besoin vers l'extérieur (ex. : `ICommandeRepository`, `IStockRepository`, `IPaymentPort`), mais ne connaît jamais leurs implémentations concrètes.

### Couche Application

La couche Application orchestre les cas d'usage métier. Elle reçoit une intention (ex. : `PasserCommandeUseCase`, `RéserverStockUseCase`, `InitierRetourUseCase`), récupère les agrégats via les ports de repository, délègue la logique métier à la couche Domain, puis propage les résultats (persistance, publication d'événements, notifications) via les ports sortants. Elle ne contient **aucune logique métier propre** : elle coordonne, ne décide pas. C'est aussi elle qui gère les transactions applicatives (s'assurer que la sauvegarde de la commande et la publication de l'événement `CommandeCréée` se font de façon cohérente). Les services applicatifs dépendent uniquement des interfaces définies dans le Domain, jamais directement des adaptateurs.

### Couche Adapters

La couche Adapters contient les implémentations concrètes qui connectent le système au monde extérieur. On distingue deux types :

- **Adaptateurs primaires (pilotants)** — ils initient les appels vers l'application. Exemples : le contrôleur REST (`POST /commandes`) qui reçoit la requête HTTP et appelle le `PasserCommandeUseCase`, ou le consommateur de messages (broker) qui écoute les événements entrants du `ContexteStock`.

- **Adaptateurs secondaires (pilotés)** — ils sont appelés par l'application pour interagir avec l'infrastructure. Exemples : `CommandeRepositoryPostgres` qui implémente `ICommandeRepository` pour persister les commandes en base, `RabbitMQEventPublisher` qui publie `CommandeCréée` sur le broker, `StripePaymentAdapter` (ACL) qui traduit les appels internes vers l'API propriétaire du prestataire de paiement, ou `EmailNotificationAdapter` pour l'envoi des notifications client.

---

## Schéma de l'architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ADAPTATEURS PRIMAIRES                           │
│   ┌──────────────────────┐       ┌──────────────────────────────────┐   │
│   │  Contrôleur REST     │       │  Consommateur Broker             │   │
│   │  POST /commandes     │       │  (écoute StockRéservé, etc.)     │   │
│   └──────────┬───────────┘       └──────────────┬───────────────────┘   │
└──────────────┼──────────────────────────────────┼───────────────────────┘
               │                                  │
               ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          COUCHE APPLICATION                             │
│                                                                         │
│   PasserCommandeUseCase          RéserverStockUseCase                   │
│   InitierPaiementUseCase         InitierRetourUseCase                   │
│                                                                         │
│   (orchestration — pas de logique métier, appel aux ports)              │
└──────┬──────────────────────────────────────────────────────────┬───────┘
       │                                                          │
       ▼                                                          ▼
┌──────────────────────────────────┐     ┌────────────────────────────────┐
│          COUCHE DOMAIN           │     │       PORTS SORTANTS           │
│                                  │     │    (interfaces abstraites)     │
│  Agrégat Commande                │     │                                │
│    └─ LigneCommande              │     │  ICommandeRepository           │
│    └─ AdresseLivraison (OV)      │     │  IStockRepository              │
│    └─ Montant (OV)               │     │  IPaymentPort                  │
│    └─ StatutCommande (OV)        │     │  IEventPublisher               │
│                                  │     │  INotificationPort             │
│  Agrégat Stock                   │     │                                │
│    └─ RéservationStock           │     └────────────────────────────────┘
│                                  │
│  Agrégat Colis                   │
│    └─ NumeroSuivi (OV)           │
│                                  │
│  Événements de domaine :         │
│    CommandeCréée                 │
│    StockRéservé / Insuffisant    │
│    ColisExpédié / Livré          │
└──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        ADAPTATEURS SECONDAIRES                          │
│                                                                         │
│  CommandeRepositoryPostgres   │  RabbitMQEventPublisher                 │
│  StockRepositoryPostgres      │  StripePaymentAdapter (ACL)             │
│  ColisRepositoryPostgres      │  EmailNotificationAdapter               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Exemple de flux : PasserCommande (requête → réponse)
## Exemple de flux : PasserCommande (requête → réponse)

**Contexte** : le client valide son panier (casque + housse) avec livraison express J+1 et paiement par carte.

**1 — Adaptateur primaire (REST)**
Le contrôleur reçoit `POST /commandes` avec les articles, l'adresse et le mode de paiement. Il crée un DTO `PasserCommandeCommand` et appelle le `PasserCommandeUseCase`.

**2 — Couche Application**
Le use case récupère les données produits via `ICataloguePort`, construit les `LigneCommande`, crée l'agrégat `Commande`, puis appelle `IPaymentPort.initierPaiement()`.

**3 — Couche Domain**
L'agrégat `Commande` vérifie les invariants (au moins une ligne, calcul du `MontantTotal`), initialise le statut à `Créée` et produit l'événement `CommandeCréée`.

**4 — Adaptateur secondaire (Paiement)**
Le `StripePaymentAdapter` traduit l'appel interne vers l'API Stripe et retourne l'autorisation (acceptée ou refusée).

**5 — Couche Application (suite)**
Si autorisé, le use case passe le statut à `Validée`, persiste via `ICommandeRepository.save()` et publie `CommandeCréée` sur le broker.

**6 — Réponse REST**
Le contrôleur retourne `201 Created` avec le `CommandeId` et le statut `Validée`. Un e-mail est envoyé via `INotificationPort`.
