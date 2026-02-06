# Bounded Contexts

## Vue d'ensemble

Le système e-commerce & livraison est décomposé en **6 Bounded Contexts** cohérents, identifiés à partir des événements et commandes de l'Event Storming, et alignés sur les problématiques métier décrites dans le domaine.

## Tableau des Bounded Contexts

| Bounded Context | Type | Rôle / Responsabilité principale |
|-----------------|------|----------------------------------|
| **ContexteCatalogue** | Supporting | Gère le référentiel produits : fiches descriptives, catégories, caractéristiques techniques, prix de référence et avis clients. Il fournit les fonctionnalités de recherche et de consultation du catalogue. Ce contexte alimente les autres BC (Commande, Stock) avec les informations produits nécessaires, mais ne constitue pas le cœur de la différenciation métier. |
| **ContexteCommande** | Core | Orchestre le cycle de vie complet d'une commande, de la création du panier jusqu'à la clôture. Il gère la validation du panier, la création de la commande, la coordination avec le paiement et le stock, ainsi que le suivi des statuts (créée, validée, en préparation, expédiée, livrée, annulée). C'est le BC central du système, garant de la cohérence transactionnelle et de l'expérience client. |
| **ContextePaiement** | Generic | Encapsule l'ensemble des opérations financières : initiation du paiement, validation auprès du prestataire bancaire, gestion des échecs et relances, ainsi que le traitement des remboursements. Ce contexte s'appuie sur des solutions tierces (passerelles de paiement) et n'est pas spécifique au métier e-commerce ; il est donc classifié comme générique. |
| **ContexteStock** | Core | Assure la gestion en temps réel de l'inventaire : niveaux de stock par entrepôt, réservation lors de la commande, décrémentation à l'expédition et réintégration lors des retours. Il détecte les ruptures, déclenche les alertes de réapprovisionnement et garantit qu'aucune survente ne se produit. Sa fiabilité est critique pour l'ensemble du système. |
| **ContexteLivraison** | Core | Prend en charge toute la logistique d'acheminement : préparation des colis en entrepôt (picking, emballage), assignation aux livreurs, suivi en temps réel du transport et confirmation de la remise au client (preuve de livraison). Il gère également le calcul des délais, le choix du mode de livraison et les notifications de suivi envoyées au client. |
| **ContexteRetour** | Supporting | Gère le processus inverse de la commande : demande de retour par le client, génération de l'étiquette retour, réception et contrôle du produit retourné en entrepôt, décision de remise en stock ou destruction, et déclenchement du remboursement. Ce BC supporte la conformité réglementaire (droit de rétractation) et contribue à la satisfaction client. |
