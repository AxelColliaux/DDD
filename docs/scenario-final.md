# Scénario complet inter-contextes

## Scénario choisi : Commande -> Paiement -> Stock -> Livraison

### Contexts impliqués

- **ContexteCommande** : orchestre le parcours de bout en bout et porte le statut de la commande.
- **ContextePaiement** : autorise ou refuse la transaction financière via le PSP (avec ACL).
- **ContexteStock** : réserve les quantités pour éviter la survente avant préparation logistique.
- **ContexteLivraison** : prend en charge l'expédition, le suivi et la confirmation de livraison.

### Description narrative détaillée

Le client valide son panier depuis l'interface e-commerce et confirme son adresse de livraison ainsi que le mode de transport.
Le ContexteCommande crée la commande avec ses lignes, calcule le montant total et place la commande dans un état "En attente de paiement".
À ce stade, la commande n'est pas encore confirmée : aucune préparation d'entrepôt n'est lancée.
Le ContexteCommande appelle ensuite le ContextePaiement en REST pour initier la demande d'autorisation.
Le ContextePaiement traduit la requête métier vers le format du prestataire bancaire via sa couche anticorruption.
Le PSP répond avec un résultat d'autorisation ; le ContextePaiement renvoie ce résultat au ContexteCommande.
Si le paiement est refusé, la commande passe en échec métier et le flux s'arrête avec notification client.
Si le paiement est autorisé, le ContexteCommande publie l'événement `CommandeValidee` sur le broker.
Le ContexteStock consomme cet événement et tente de réserver les quantités pour chaque SKU commandé.
Chaque réservation est effectuée de manière atomique pour empêcher un passage en stock négatif en concurrence.
En cas de succès, le ContexteStock publie `StockReserve` avec la référence de commande et les lignes réservées.
En cas d'échec partiel ou total, il publie `StockInsuffisant` et une compensation est déclenchée côté Commande.
Quand `StockReserve` est reçu, le ContexteCommande confirme la commande et la bascule vers "Validée".
Le ContexteCommande publie alors `CommandeTransmiseALivraison` pour démarrer le flux logistique.
Le ContexteLivraison consomme l'événement, crée le colis et génère un numéro de suivi.
Lorsque le colis quitte l'entrepôt, le ContexteLivraison publie `ColisExpedie`.
Le ContexteCommande met à jour le statut en "Expédiée" afin d'exposer l'information côté client.
Au moment de la remise finale, le ContexteLivraison publie `ColisLivre` avec preuve de livraison.
Le ContexteCommande clôture le cycle et passe la commande à l'état terminal "Livrée".
Le client reçoit les notifications de confirmation, d'expédition puis de livraison, cohérentes avec le statut métier.

### Liste des événements déclenchés (dans l'ordre)

1. `CommandeValidee` (ContexteCommande -> ContexteStock)
2. `StockReserve` (ContexteStock -> ContexteCommande)
3. `CommandeTransmiseALivraison` (ContexteCommande -> ContexteLivraison)
4. `ColisPrepare` (ContexteLivraison -> ContexteCommande)
5. `ColisExpedie` (ContexteLivraison -> ContexteCommande)
6. `ColisLivre` (ContexteLivraison -> ContexteCommande)

**Branche alternative d'échec (stock insuffisant)** :

1. `CommandeValidee`
2. `StockInsuffisant`
3. `CommandeAnnulee` (avec déclenchement d'un remboursement selon la politique métier)

### Rappel des invariants concernés

- **Invariant Commande - Au moins une LigneCommande** : la commande ne doit jamais devenir vide pendant le scénario ; sinon le flux de paiement/réservation serait invalide.
- **Invariant Commande - MontantTotal cohérent** : le montant autorisé par Paiement doit correspondre strictement aux lignes + livraison calculées par Commande.
- **Invariant Commande - Transitions de statut valides** : transitions autorisées uniquement (`EnAttentePaiement -> Validee -> Expediee -> Livree`), sans retour arrière depuis un état terminal.
- **Invariant Stock - QuantitéDisponible non négative** : toute réservation doit refuser les demandes dépassant le disponible.
- **Invariant Stock - Conservation disponible + réservé** : chaque réservation transfère une quantité de disponible vers réservé sans créer/perdre d'unités.
- **Invariant Stock - Réservation rattachée à une commande** : toute réservation doit porter un `commandeId` pour garantir la traçabilité et la libération en cas de compensation.
