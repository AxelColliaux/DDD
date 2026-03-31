# Scénarios d'intégration

Ce document décrit les scénarios end-to-end traversant plusieurs couches et Bounded Contexts du système e-commerce.

---

## Scénario 1 : Validation d'une commande avec réservation de stock

### Contexte métier

Un client finalise son achat après avoir constitué son panier. Ce scénario illustre le flux complet depuis la requête HTTP externe jusqu'aux mises à jour dans les différents domaines (Commande, Paiement, Stock), en passant par les couches applicatives et les événements de domaine.

### Narration du scénario (10-20 lignes)

1. **Requête externe** : Le client clique sur "Valider ma commande" dans l'interface web. Une requête `POST /api/commandes` est envoyée au système avec le panierId, l'adresse de livraison et le mode de livraison choisi.

2. **Couche API (Infrastructure)** : Le contrôleur REST reçoit la requête, valide le format des données entrantes et transmet la commande applicative `ValiderCommande` au service applicatif du ContexteCommande.

3. **Service applicatif (Application)** : Le service `CommandeApplicationService` orchestre le cas d'usage. Il récupère le Panier existant via le `PanierRepository`, vérifie que le panier n'est pas vide et que le client est authentifié.

4. **Création de l'agrégat Commande (Domaine)** : Le service invoque la factory `Commande.creerDepuisPanier()` qui transforme les LignePanier en LigneCommande, calcule le montant total et initialise le StatutCommande à "EnAttenteValidation". Les règles métier sont vérifiées (montant positif, adresse complète).

5. **Communication inter-contextes (Paiement)** : Le ContexteCommande publie l'événement de domaine `CommandeInitiee`. Le ContextePaiement écoute cet événement via un mécanisme de messaging et déclenche `InitierPaiement` auprès du prestataire externe (via ACL).

6. **Réponse du prestataire (ACL)** : La couche anticorruption traduit la réponse du prestataire bancaire en `AutorisationPaiement`. Si acceptée, l'événement `PaiementAutorise` est publié.

7. **Réservation du stock (ContexteStock)** : Le ContexteStock écoute `PaiementAutorise`. Pour chaque ligne de commande, le service de domaine `StockService` invoque `Stock.reserver(quantite)` sur l'entrepôt approprié. Si le stock est suffisant, une `ReservationStock` est créée et l'événement `StockReserve` est émis.

8. **Confirmation de la commande (ContexteCommande)** : Le ContexteCommande reçoit `StockReserve`. L'agrégat Commande transite vers le StatutCommande "Validee" via `commande.confirmer()`. L'événement `CommandeValidee` est publié.

9. **Persistance (Infrastructure)** : Le `CommandeRepository` persiste l'agrégat Commande mis à jour. Le `StockRepository` enregistre les nouvelles quantités réservées. Les transactions sont commitées.

10. **Réponse au client** : La couche API construit la réponse JSON avec le commandeId, le statut "Validee" et la date de livraison estimée. Le client reçoit un code HTTP 201 Created.

11. **Notification (Side effect)** : L'événement `CommandeValidee` déclenche également l'envoi d'un email de confirmation au client via le service de notification.

---

### Diagramme de séquence UML

```
┌──────────┐    ┌───────────┐    ┌─────────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Client  │    │ API REST  │    │ CommandeService │    │  Commande   │    │  Paiement   │    │    Stock    │
│  (HTTP)  │    │(Infra)    │    │  (Application)  │    │  (Domaine)  │    │   (ACL)     │    │  (Domaine)  │
└────┬─────┘    └─────┬─────┘    └───────┬─────────┘    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
     │                │                  │                     │                  │                  │
     │ POST /api/     │                  │                     │                  │                  │
     │ commandes      │                  │                     │                  │                  │
     │───────────────>│                  │                     │                  │                  │
     │                │                  │                     │                  │                  │
     │                │ ValiderCommande  │                     │                  │                  │
     │                │─────────────────>│                     │                  │                  │
     │                │                  │                     │                  │                  │
     │                │                  │ creerDepuisPanier() │                  │                  │
     │                │                  │────────────────────>│                  │                  │
     │                │                  │                     │                  │                  │
     │                │                  │    <<Commande>>     │                  │                  │
     │                │                  │<────────────────────│                  │                  │
     │                │                  │                     │                  │                  │
     │                │                  │        publish: CommandeInitiee       │                  │
     │                │                  │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ >│                  │
     │                │                  │                     │                  │                  │
     │                │                  │                     │  InitierPaiement │                  │
     │                │                  │                     │  (prestataire)   │                  │
     │                │                  │                     │                  │──┐               │
     │                │                  │                     │                  │  │ Appel externe │
     │                │                  │                     │                  │<─┘               │
     │                │                  │                     │                  │                  │
     │                │                  │          publish: PaiementAutorise    │                  │
     │                │                  │<─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │                  │
     │                │                  │                     │                  │                  │
     │                │                  │                 publish: PaiementAutorise                 │
     │                │                  │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ >│
     │                │                  │                     │                  │                  │
     │                │                  │                     │                  │   reserver()     │
     │                │                  │                     │                  │                  │──┐
     │                │                  │                     │                  │                  │  │
     │                │                  │                     │                  │                  │<─┘
     │                │                  │                     │                  │                  │
     │                │                  │                         publish: StockReserve             │
     │                │                  │<─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│
     │                │                  │                     │                  │                  │
     │                │                  │    confirmer()      │                  │                  │
     │                │                  │────────────────────>│                  │                  │
     │                │                  │                     │                  │                  │
     │                │                  │  <<StatutValidee>>  │                  │                  │
     │                │                  │<────────────────────│                  │                  │
     │                │                  │                     │                  │                  │
     │                │                  │ save(commande)      │                  │                  │
     │                │                  │─────────────────────────────────────────────────────────> │
     │                │                  │                     │                  │      [Repository]│
     │                │                  │                     │                  │                  │
     │                │  JSON Response   │                     │                  │                  │
     │                │<─────────────────│                     │                  │                  │
     │                │                  │                     │                  │                  │
     │  201 Created   │                  │                     │                  │                  │
     │  + commandeId  │                  │                     │                  │                  │
     │<───────────────│                  │                     │                  │                  │
     │                │                  │                     │                  │                  │
```

---

### Bounded Contexts impliqués

| Bounded Context | Rôle dans le scénario |
|-----------------|----------------------|
| **ContexteCommande** | Orchestre le flux principal, crée et confirme l'agrégat Commande |
| **ContextePaiement** | Gère l'autorisation via le prestataire externe (ACL) |
| **ContexteStock** | Réserve les quantités pour éviter la survente |

### Événements de domaine émis

| Événement | Émetteur | Consommateurs |
|-----------|----------|---------------|
| `CommandeInitiee` | ContexteCommande | ContextePaiement |
| `PaiementAutorise` | ContextePaiement | ContexteCommande, ContexteStock |
| `StockReserve` | ContexteStock | ContexteCommande |
| `CommandeValidee` | ContexteCommande | Service Notification |

### Points d'intégration critiques

1. **ACL Paiement** : Traduit les réponses du prestataire externe en concepts du domaine
2. **Messaging asynchrone** : Les événements permettent le découplage entre contextes
3. **Compensation** : Si le stock est insuffisant, un événement `StockInsuffisant` annule la commande
