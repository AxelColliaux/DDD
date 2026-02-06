# Entités et Objets Valeur — Vue conceptuelle

## Entités

### Commande

La Commande est l'agrégat racine central du **ContexteCommande**. Elle représente l'engagement d'achat d'un Client et orchestre l'ensemble du flux, de la validation du panier jusqu'à la livraison ou l'annulation.

#### Attributs

| Attribut | Type métier | Description |
|----------|------------|-------------|
| idCommande | Identifiant unique | Identifiant métier de la commande, généré à la création. Sert de référence dans tous les autres contextes (Livraison, Retour, Paiement). |
| client | Référence Client | Le client ayant passé la commande. Permet d'associer la commande à un compte et de gérer les notifications. |
| lignesCommande | Collection de LigneCommande | Ensemble des articles commandés avec leur quantité et prix figé au moment de la validation. Doit contenir au moins une ligne. |
| statut | StatutCommande | État courant dans le cycle de vie (Créée, Validée, EnPréparation, Expédiée, Livrée, Annulée). Gouverne les transitions et les actions autorisées. |
| adresseLivraison | AdresseLivraison | Adresse complète de livraison saisie par le client. Transmise au ContexteLivraison pour l'acheminement. |
| modeLivraison | ModeLivraison | Mode choisi par le client (standard, express, point relais). Détermine le délai cible et le coût de livraison. |
| montantTotal | Montant | Somme totale de la commande (articles + livraison). Calculé à partir des lignes et du mode de livraison. |
| dateCreation | Date | Date et heure de création de la commande. Sert de référence pour les délais de livraison et le droit de rétractation. |

#### Invariants

1. **Au moins une ligne de commande** — Une Commande doit toujours contenir au minimum une LigneCommande. Une commande vide est invalide et ne peut pas être créée.

2. **Montant total cohérent** — Le montantTotal doit toujours être égal à la somme des (prix × quantité) de chaque LigneCommande, plus le coût du ModeLivraison. Toute modification d'une ligne doit recalculer le total.

3. **Transitions de statut valides** — Le statut ne peut suivre que les transitions autorisées : Créée → Validée → EnPréparation → Expédiée → Livrée. Une commande Livrée ou Annulée ne peut pas revenir à un statut antérieur. L'annulation n'est possible que depuis les statuts Créée ou Validée.

---

### Stock

Le Stock est l'agrégat racine du **ContexteStock**. Il représente la quantité disponible d'un produit dans un entrepôt donné et garantit qu'aucune survente ne se produit.

#### Attributs

| Attribut | Type métier | Description |
|----------|------------|-------------|
| idStock | Identifiant unique | Identifiant métier du stock, lié à un couple (Produit, Entrepôt). Permet de tracer les mouvements d'inventaire. |
| produit | Référence Produit | Le produit concerné (identifié par SKU). Un même produit peut avoir des lignes de stock dans plusieurs entrepôts. |
| entrepot | Référence Entrepôt | L'entrepôt physique où le produit est stocké. Détermine la proximité pour le choix logistique. |
| quantiteDisponible | Quantité | Nombre d'unités disponibles à la vente (non réservées). Doit refléter l'état réel de l'inventaire en temps réel. |
| quantiteReservee | Quantité | Nombre d'unités bloquées pour des commandes en cours de traitement. Ces unités ne sont plus disponibles à la vente mais n'ont pas encore quitté l'entrepôt. |
| seuilAlerte | Quantité | Niveau en dessous duquel une alerte de réapprovisionnement est déclenchée. Défini par la Direction pour chaque produit/entrepôt. |

#### Invariants

1. **Quantité disponible non négative** — La quantiteDisponible ne peut jamais devenir négative. Toute tentative de réservation dépassant la quantité disponible doit être rejetée et déclencher un événement `StockInsuffisant`.

2. **Cohérence stock total** — La somme (quantiteDisponible + quantiteReservee) doit toujours correspondre au stock physique réel en entrepôt. Toute entrée (réapprovisionnement, retour) ou sortie (expédition) doit mettre à jour ces compteurs de manière atomique.

3. **Réservation liée à une commande** — Chaque unité réservée doit être associée à une commande identifiée. Si la commande est annulée, les unités réservées doivent être libérées et la quantiteDisponible réajustée.

---

## Objet Valeur

### AdresseLivraison

L'AdresseLivraison est un **Value Object** du ContexteCommande. Elle décrit l'adresse complète où le colis doit être livré.

#### Propriétés

| Propriété | Type métier | Description |
|-----------|------------|-------------|
| nomDestinataire | Texte | Nom complet de la personne qui réceptionnera le colis. |
| rue | Texte | Numéro et nom de la voie (ex : « 12 rue des Lilas »). |
| complementAdresse | Texte (optionnel) | Informations complémentaires (bâtiment, étage, digicode, etc.). |
| codePostal | Code postal | Code postal de la commune de livraison. |
| ville | Texte | Nom de la commune de livraison. |
| pays | Code pays | Code ISO du pays de livraison (ex : FR, BE, CH). |

#### Pourquoi est-elle immuable ?

L'AdresseLivraison est immuable car elle représente un **fait figé au moment de la commande** : l'endroit exact où le client a demandé à être livré. Si le client souhaite modifier son adresse, une nouvelle instance d'AdresseLivraison est créée et remplace l'ancienne sur la commande (tant que celle-ci n'est pas encore expédiée), plutôt que de modifier l'objet existant. Cette immuabilité garantit l'intégrité de l'historique : une fois le colis expédié, l'adresse utilisée pour la livraison reste traçable et inaltérable, ce qui est essentiel pour les litiges, les retours et l'audit.

De plus, deux AdresseLivraison sont considérées comme identiques si et seulement si toutes leurs propriétés sont égales (égalité par valeur, non par référence). Cela signifie qu'il n'y a pas d'identifiant propre : l'adresse n'a pas de cycle de vie autonome, elle n'existe qu'en tant que composant d'une Commande.
