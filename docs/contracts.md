# Contrats d'échange entre contexts

Ce document formalise les échanges de données entre les différents Bounded Contexts.

## 1. Demande de Paiement (REST)

* **Context source** : `ContexteCommande`
* **Context cible** : `ContextePaiement`
* **Type d'échange** : API REST (Appel synchrone)
* **Description** : Lorsque le client valide son de panier, le contexte Commande fait une requête au contexte Paiement pour initier et autoriser le paiement.

### Schéma du message (Requête JSON)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "DemandePaiement",
  "type": "object",
  "properties": {
    "commandeId": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant unique de la commande"
    },
    "clientId": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant du client"
    },
    "montant": {
      "type": "number",
      "minimum": 0,
      "description": "Montant total de la commande"
    },
    "devise": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "Code devise ISO 4217, ex: EUR"
    },
    "moyenPaiement": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["CARTE_BANCAIRE", "PAYPAL", "VIREMENT"]
        },
        "token": {
          "type": "string",
          "description": "Token sécurisé représentant le moyen de paiement"
        }
      },
      "required": ["type", "token"]
    }
  },
  "required": ["commandeId", "clientId", "montant", "devise", "moyenPaiement"]
}
```

### Schéma du message (Réponse JSON)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ReponsePaiement",
  "type": "object",
  "properties": {
    "transactionId": {
      "type": "string",
      "description": "Identifiant de la transaction de paiement"
    },
    "statut": {
      "type": "string",
      "enum": ["AUTORISE", "REFUSE", "EN_ATTENTE"]
    },
    "messageErreur": {
      "type": "string",
      "description": "Message d'erreur en cas de refus"
    },
    "dateTransaction": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["transactionId", "statut", "dateTransaction"]
}
```

## 2. Réservation de Stock (Événement)

* **Context source** : `ContexteCommande`
* **Context cible** : `ContexteStock`
* **Type d'échange** : Événement asynchrone (Message Broker / Kafka / RabbitMQ)
* **Description** : Une fois le paiement autorisé, le contexte Commande publie un événement `CommandePayee` ou `CommandeValidée`. Le contexte Stock écoute cet événement pour décrémenter/réserver les quantités commandées.

### Schéma du message (JSON Schema événementiel)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "EvenementCommandeValidee",
  "type": "object",
  "properties": {
    "eventId": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant unique de l'événement"
    },
    "eventType": {
      "type": "string",
      "const": "CommandeValidee"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "Date et heure de création de l'événement"
    },
    "payload": {
      "type": "object",
      "properties": {
        "commandeId": {
          "type": "string",
          "format": "uuid"
        },
        "lignes": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "sku": {
                "type": "string",
                "description": "Référence unique du produit"
              },
              "quantite": {
                "type": "integer",
                "minimum": 1
              }
            },
            "required": ["sku", "quantite"]
          },
          "minItems": 1
        }
      },
      "required": ["commandeId", "lignes"]
    }
  },
  "required": ["eventId", "eventType", "timestamp", "payload"]
}
```
