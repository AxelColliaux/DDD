# Observabilité

## Correlation ID

Le Correlation ID est un identifiant unique qui permet de suivre une même transaction métier de bout en bout, même lorsqu'elle traverse plusieurs Bounded Contexts. Dans notre système, il est créé au point d'entrée de la requête client, au moment de la validation/paiement d'une commande dans le ContexteCommande. Il est ensuite propagé dans tous les appels REST sortants (en en-tête HTTP `X-Correlation-ID`) et embarqué dans les événements publiés sur le broker (champ `correlationId` dans le payload ou les métadonnées). Chaque service le réutilise tel quel, sans le régénérer, pour garantir une trace continue entre Commande, Paiement, Stock et Livraison. En cas d'incident, ce même identifiant permet de reconstituer rapidement la chronologie complète dans les logs et les métriques.

## Métriques métier

Métriques exposables via l'endpoint `/metrics` :

| Nom de la métrique | Description | Type (counter / gauge / histogram) |
| --- | --- | --- |
| `commande_paiement_refuse_total` | Nombre cumulé de tentatives de paiement refusées pour des commandes (par exemple carte refusée, fonds insuffisants). Permet de surveiller la friction au checkout. | counter |
| `stock_reservation_pending` | Nombre courant de demandes de réservation de stock en attente de traitement côté ContexteStock (file d'événements non consommés). Indique la pression opérationnelle en temps réel. | gauge |
| `commande_to_stock_reservation_seconds` | Durée entre l'événement de validation de commande et la confirmation de réservation de stock. Sert à mesurer la latence inter-contextes et la qualité perçue du flux commande. | histogram |

## Logging structuré

Exemple de log JSON structuré (un événement applicatif tracé avec corrélation inter-contextes) :

```json
{
	"timestamp": "2026-04-27T10:45:12.381Z",
	"level": "INFO",
	"service": "contexte-commande",
	"environment": "prod",
	"correlationId": "c9b9f3be-3bf6-4be6-95cb-6a34714eb8f4",
	"traceId": "7b5f9c13d2b4436f96f91f97176f6fe4",
	"spanId": "8d1a6c7fb62c2f10",
	"event": "CommandeValidee",
	"commandeId": "8f4fe6ec-65aa-4107-a92a-4f54f3e86e5f",
	"clientId": "f522b2cd-2e2f-42b6-9490-fa656d960f9a",
	"targetContext": "ContexteStock",
	"integrationType": "event",
	"status": "PUBLISHED",
	"durationMs": 42,
	"message": "Publication de l'evenement EvenementCommandeValidee sur le topic order-events"
}
```
