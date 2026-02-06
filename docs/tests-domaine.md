# Scénarios de test de domaine

## Invariants

### Invariant 1 : Au moins une LigneCommande

#### Scénario 1 – Happy path (Invariant 1)

- Given une Commande en statut "Créée" avec 1 LigneCommande (casque x1)
- When la commande est enregistrée
- Then la Commande est valide et conserve au moins une LigneCommande

#### Scénario 2 – Sad path (Invariant 1)

- Given une Commande en statut "Créée" avec 0 LigneCommande
- When on tente d’enregistrer la commande
- Then l’opération est refusée car une commande ne peut pas être vide

### Invariant 2 : MontantTotal cohérent avec les lignes + livraison

#### Scénario 1 – Happy path (Invariant 2)

- Given une Commande avec 2 lignes (casque 100€ x1, housse 20€ x1) et une livraison express 10€
- When le MontantTotal est calculé
- Then le MontantTotal vaut 130€

#### Scénario 2 – Sad path (Invariant 2)

- Given une Commande avec les mêmes lignes et livraison mais un MontantTotal fixé à 120€
- When on valide la commande
- Then la validation échoue car le MontantTotal est incohérent

### Invariant 3 : Transitions de StatutCommande valides

#### Scénario 1 – Happy path (Invariant 3)

- Given une Commande en statut "Validée"
- When on déclenche la préparation
- Then le statut devient "EnPréparation"

#### Scénario 2 – Sad path (Invariant 3)

- Given une Commande en statut "Livrée"
- When on tente de la repasser en statut "EnPréparation"
- Then l’opération est refusée car une commande livrée est terminale

### Invariant 4 : QuantitéDisponible non négative

#### Scénario 1 – Happy path (Invariant 4)

- Given un Stock avec QuantitéDisponible = 5
- When on réserve 2 unités pour une commande
- Then QuantitéDisponible devient 3 et la réservation est acceptée

#### Scénario 2 – Sad path (Invariant 4)

- Given un Stock avec QuantitéDisponible = 1
- When on tente de réserver 2 unités
- Then la réservation est refusée et un événement "StockInsuffisant" est produit

### Invariant 5 : Conservation du stock : disponible + réservé cohérent

#### Scénario 1 – Happy path (Invariant 5)

- Given un Stock avec Disponible = 5 et Réservée = 0 (total = 5)
- When on réserve 2 unités
- Then Disponible = 3 et Réservée = 2 (total reste = 5)

#### Scénario 2 – Sad path (Invariant 5)

- Given un Stock avec Disponible = 5 et Réservée = 0
- When une réservation de 2 unités est appliquée en décrémentant Disponible sans incrémenter Réservée
- Then l’état est invalide car le total devient incohérent

### Invariant 6 : RéservationStock toujours rattachée à une Commande

#### Scénario 1 – Happy path (Invariant 6)

- Given une CommandeId "CMD-123" et un Stock disponible
- When une RéservationStock est créée pour "CMD-123" (quantité 1)
- Then la quantité réservée est traçable et rattachée à la commande "CMD-123"

#### Scénario 2 – Sad path (Invariant 6)

- Given un Stock disponible
- When on tente de réserver 1 unité sans fournir de CommandeId
- Then l’opération est refusée car une réservation doit être rattachée à une commande

