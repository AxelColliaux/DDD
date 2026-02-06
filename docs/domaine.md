# Scénario choisi

E-commerce & livraison

# Contexte métier

Le secteur du e-commerce connaît une croissance soutenue, portée par l'évolution des habitudes de consommation et la digitalisation des échanges commerciaux. Notre système s'inscrit dans le contexte d'une entreprise de vente en ligne généraliste (marketplace) qui propose un catalogue de produits variés (électronique, vêtements, alimentation, etc.) à des clients particuliers et professionnels. L'organisation doit gérer l'ensemble de la chaîne de valeur : de la mise en ligne des produits jusqu'à la livraison effective au client final, en passant par la gestion des stocks, le traitement des commandes et le suivi logistique.

Les enjeux principaux sont multiples. D'un point de vue commercial, il s'agit de proposer une expérience d'achat fluide et fiable afin de fidéliser la clientèle dans un marché très concurrentiel. Sur le plan opérationnel, l'entreprise doit garantir des délais de livraison courts et respectés, maintenir une gestion rigoureuse des stocks pour éviter les ruptures ou les surstocks, et assurer la traçabilité complète de chaque commande. La satisfaction client repose en grande partie sur la capacité à livrer le bon produit, au bon moment et au bon endroit.

Enfin, l'entreprise fait face à des contraintes réglementaires (droit de rétractation, protection des données personnelles, facturation conforme) et à des exigences de scalabilité : le système doit pouvoir absorber des pics de charge importants lors d'opérations commerciales (soldes, Black Friday) sans dégradation de service. La coordination entre les équipes commerciales, logistiques et le service client constitue un défi organisationnel majeur que le système doit faciliter.

# Rôles utilisateurs

| Rôle | Type | Description |
|------|------|-------------|
| Responsable commercial | Direction | Supervise la stratégie de vente, définit les promotions et les politiques tarifaires. Il pilote les indicateurs de performance (chiffre d'affaires, taux de conversion, panier moyen) et prend les décisions stratégiques sur le catalogue produits. |
| Préparateur de commande | Opérationnel | Prend en charge les commandes validées, effectue le picking des produits en entrepôt, assemble et emballe les colis. Il met à jour le statut de préparation et signale les éventuelles anomalies (produit endommagé, rupture de stock). |
| Livreur | Opérationnel | Assure l'acheminement des colis depuis l'entrepôt jusqu'au client final. Il gère son itinéraire de livraison, confirme la remise des colis et collecte la preuve de livraison (signature, photo). |
| Client | Client | Parcourt le catalogue, passe des commandes, effectue le paiement en ligne et suit l'avancement de sa livraison. Il peut également retourner un produit, laisser un avis ou contacter le service client en cas de problème. |

# Problématiques métier

1. **Gestion des stocks en temps réel** : Le système doit refléter à tout instant l'état exact des stocks disponibles afin d'éviter la vente de produits indisponibles (sur-vente) et permettre un réapprovisionnement anticipé. Un décalage entre le stock réel et le stock affiché engendre des annulations de commandes et une perte de confiance du client.

2. **Fiabilité et traçabilité de la livraison** : Chaque commande doit être suivie de bout en bout, depuis la validation du panier jusqu'à la remise au client. Les retards, colis perdus ou erreurs de livraison doivent être détectés rapidement et gérés de manière transparente pour le client.

3. **Cohérence du processus de commande** : Le passage d'une commande implique plusieurs étapes critiques (validation du panier, réservation du stock, paiement, préparation, expédition). Une défaillance à n'importe quelle étape doit être gérée de façon cohérente (compensation, annulation partielle) sans laisser le système dans un état incohérent.

4. **Gestion des retours et remboursements** : Le droit de rétractation et les cas de produits défectueux imposent un processus de retour structuré : demande du client, validation, réception du produit retourné, remise en stock ou destruction, et remboursement. Ce flux inverse doit être aussi rigoureux que le flux de commande directe.

5. **Passage à l'échelle lors des pics de charge** : Lors d'événements commerciaux (soldes, ventes flash, Black Friday), le volume de commandes peut être multiplié par dix. Le système doit absorber ces pics sans dégradation du temps de réponse ni perte de commandes, tout en maintenant la cohérence des stocks.

# Scénario fil rouge

1. **Le client parcourt le catalogue** : Un client se connecte à la plateforme et recherche un casque audio sans fil. Il consulte plusieurs fiches produits, compare les caractéristiques et les avis, puis sélectionne le modèle qui lui convient.

2. **Ajout au panier et passage de commande** : Le client ajoute le casque à son panier ainsi qu'une housse de protection. Il accède à son panier, vérifie les articles et le montant total, puis lance le processus de commande.

3. **Choix de la livraison et paiement** : Le client saisit son adresse de livraison, choisit une livraison express à J+1, puis procède au paiement par carte bancaire. Le système valide le paiement auprès du prestataire bancaire.

4. **Validation et réservation du stock** : Une fois le paiement confirmé, le système crée la commande, réserve les deux articles dans le stock de l'entrepôt et envoie une confirmation par e-mail au client avec un récapitulatif et un numéro de suivi.

5. **Transmission à l'entrepôt** : La commande est transmise au préparateur de commande de l'entrepôt le plus proche disposant des deux articles en stock. Le préparateur reçoit un bon de préparation sur son terminal.

6. **Préparation du colis** : Le préparateur localise les articles dans les rayonnages (picking), vérifie leur conformité, les emballe dans un colis étiqueté avec le bon de livraison et le code-barres de suivi, puis dépose le colis en zone d'expédition.

7. **Prise en charge par le livreur** : Le livreur récupère le colis en entrepôt, scanne le code-barres pour confirmer la prise en charge. Le statut de la commande passe à « En cours de livraison » et le client reçoit une notification avec un lien de suivi en temps réel.

8. **Livraison au client** : Le livreur se rend à l'adresse indiquée, remet le colis au client et collecte sa signature électronique comme preuve de livraison. Le statut de la commande passe à « Livré ».

9. **Notification et évaluation** : Le client reçoit un e-mail de confirmation de livraison et est invité à évaluer son expérience (note et commentaire). Le responsable commercial consulte ces indicateurs pour piloter la satisfaction client.

10. **Demande de retour** : Deux jours plus tard, le client constate que la housse de protection ne correspond pas à son modèle de casque. Il initie une demande de retour depuis son espace client en précisant le motif. Le système génère une étiquette de retour prépayée, le produit est réceptionné en entrepôt, contrôlé, puis remis en stock, et le client est remboursé du montant de la housse sous 48 heures.
