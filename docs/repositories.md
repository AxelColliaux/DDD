# Repositories métier

Ce document décrit les repositories conceptuels du domaine, formalisant la persistance des agrégats sans détail technique d'implémentation.

---

## CommandeRepository

Le `CommandeRepository` gère la persistance des commandes clients. Il constitue le point d'accès unique aux agrégats Commande et garantit leur intégrité tout au long de leur cycle de vie.

| Opération métier | Description | Contraintes / règles métier |
| --- | --- | --- |
| **Créer une commande** | Enregistre une nouvelle commande dans le système après validation du panier et confirmation du paiement. La commande reçoit un identifiant unique stable (CommandeId) utilisé dans tout le SI. | Une commande ne peut être créée que si le panier associé contient au moins un article. Le montant total doit être positif et l'adresse de livraison complète et valide. Le statut initial est obligatoirement "En attente de préparation". |
| **Rechercher une commande** | Retrouve une commande existante par son identifiant métier (CommandeId) ou par des critères de recherche (client, période, statut). Permet la consultation de l'état actuel et de l'historique des transitions. | Seules les commandes appartenant au périmètre autorisé de l'utilisateur peuvent être retournées. Une commande archivée reste accessible en lecture seule. Les données sensibles (paiement) peuvent être masquées selon le contexte d'appel. |
| **Mettre à jour une commande** | Modifie l'état d'une commande existante : changement de statut, mise à jour de l'adresse de livraison, ajout d'informations de suivi. Chaque modification est tracée avec horodatage. | Les transitions de statut doivent respecter le workflow défini (ex : impossible de passer de "Livrée" à "En préparation"). L'adresse de livraison ne peut plus être modifiée une fois le colis expédié. Une commande annulée ne peut plus être modifiée. |
| **Supprimer une commande** | Retire logiquement une commande du système actif. La suppression physique n'est jamais autorisée pour des raisons de traçabilité et conformité réglementaire. | Seules les commandes annulées depuis plus de la période légale de rétention peuvent être archivées. Les commandes liées à des litiges en cours ou des retours non clôturés ne peuvent pas être supprimées. Un historique d'audit complet doit être conservé. |

---

## StockRepository

Le `StockRepository` gère la persistance des niveaux de stock par produit et entrepôt. Il assure la cohérence des quantités disponibles et réservées, protégeant le métier contre la survente.

| Opération métier | Description | Contraintes / règles métier |
| --- | --- | --- |
| **Créer un stock** | Initialise un nouvel enregistrement de stock pour une combinaison Produit/Entrepôt. Définit les quantités initiales disponibles et le seuil d'alerte de réapprovisionnement. | Le StockId (SKU + EntrepotId) doit être unique dans le système. La quantité initiale ne peut pas être négative. Un stock ne peut être créé que pour un produit existant dans le catalogue et un entrepôt actif. |
| **Consulter un stock** | Retourne l'état actuel d'un stock : quantité disponible, quantité réservée, quantité totale. Peut être filtré par entrepôt, catégorie de produit ou seuil critique. | La lecture doit refléter l'état le plus récent (cohérence forte). Les quantités réservées correspondent aux commandes confirmées mais non encore expédiées. L'accès peut être restreint selon le périmètre géographique de l'utilisateur. |
| **Mettre à jour un stock** | Modifie les quantités d'un stock existant : réservation pour une commande, libération suite à annulation, entrée de marchandise, sortie pour expédition. Toute variation est atomique et auditée. | Une réservation ne peut pas rendre la quantité disponible négative (protection contre la survente). Les mouvements de stock doivent être justifiés par une opération métier (commande, inventaire, transfert). Les modifications concurrentes doivent être gérées de manière atomique pour éviter les incohérences. |
| **Supprimer un stock** | Désactive un enregistrement de stock lorsque le produit n'est plus commercialisé dans un entrepôt donné. La suppression est logique avec conservation de l'historique des mouvements. | Un stock ne peut être supprimé que si la quantité totale (disponible + réservée) est à zéro. Les mouvements historiques doivent être conservés pour audit et reporting. La suppression doit être validée par le gestionnaire d'entrepôt concerné. |
