# Exemples d'API REST

Ce document présente les maquettes d'API REST du domaine e-commerce, sans implémentation technique. Les noms respectent l'Ubiquitous Language défini dans le projet.

---

## Endpoint 1 : Création d'une Commande

**Méthode** : `POST`

**URL** : `/api/commandes`

**Description** : Crée une nouvelle Commande à partir d'un Panier validé. Cette opération transforme les LignePanier en LigneCommande, initie la réservation du Stock et déclenche le processus de Paiement. La Commande est créée avec le StatutCommande "EnAttenteValidation" jusqu'à confirmation du Paiement.

### Exemple de requête

```json
{
  "panierId": "panier-abc123",
  "clientId": "client-789xyz",
  "adresseLivraison": {
    "nomDestinataire": "Jean Dupont",
    "rue": "15 rue de la Paix",
    "complementAdresse": "Bâtiment B, 3ème étage",
    "codePostal": "75002",
    "ville": "Paris",
    "pays": "France"
  },
  "modeLivraison": "EXPRESS_J1",
  "montantTotal": {
    "valeur": 149.99,
    "devise": "EUR"
  }
}
```

### Exemple de réponse

```json
{
  "commandeId": "CMD-2026-0331-001542",
  "statutCommande": "EnAttenteValidation",
  "dateCreation": "2026-03-31T13:30:00Z",
  "clientId": "client-789xyz",
  "lignesCommande": [
    {
      "produitId": "SKU-CASQUE-001",
      "nomProduit": "Casque audio sans fil Pro",
      "quantite": 1,
      "prixUnitaire": {
        "valeur": 129.99,
        "devise": "EUR"
      }
    },
    {
      "produitId": "SKU-HOUSSE-042",
      "nomProduit": "Housse de protection premium",
      "quantite": 1,
      "prixUnitaire": {
        "valeur": 19.99,
        "devise": "EUR"
      }
    }
  ],
  "adresseLivraison": {
    "nomDestinataire": "Jean Dupont",
    "rue": "15 rue de la Paix",
    "complementAdresse": "Bâtiment B, 3ème étage",
    "codePostal": "75002",
    "ville": "Paris",
    "pays": "France"
  },
  "modeLivraison": "EXPRESS_J1",
  "dateLivraisonEstimee": "2026-04-01",
  "montantTotal": {
    "valeur": 149.99,
    "devise": "EUR"
  },
  "reservationStockConfirmee": true
}
```

---

## Endpoint 2 : Consultation du Stock

**Méthode** : `GET`

**URL** : `/api/stocks/{stockId}`

**Description** : Retourne l'état actuel du Stock pour une combinaison Produit/Entrepôt identifiée par le StockId. Permet de consulter la quantité disponible, la quantité réservée (bloquée pour des Commandes en cours) et le seuil d'alerte de réapprovisionnement. Cette information est essentielle pour éviter la RuptureStock et la survente.

### Exemple de requête

```
GET /api/stocks/SKU-CASQUE-001_ENTREPOT-IDF-01
```

*(Pas de corps de requête pour une méthode GET)*

### Exemple de réponse

```json
{
  "stockId": "SKU-CASQUE-001_ENTREPOT-IDF-01",
  "produit": {
    "produitId": "SKU-CASQUE-001",
    "nomProduit": "Casque audio sans fil Pro"
  },
  "entrepot": {
    "entrepotId": "ENTREPOT-IDF-01",
    "nomEntrepot": "Entrepôt Île-de-France",
    "localisation": "Roissy-en-France"
  },
  "quantiteDisponible": 42,
  "quantiteReservee": 8,
  "quantiteTotale": 50,
  "seuilAlerte": 10,
  "statutStock": "Disponible",
  "derniereMiseAJour": "2026-03-31T12:45:00Z"
}
```
