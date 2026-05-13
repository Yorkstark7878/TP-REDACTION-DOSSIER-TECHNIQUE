# TP 1.4 — SupEvents : Questions & Réponses

## A.1 — Diagramme entité-relation

```mermaid
erDiagram
    User {
        UUID id PK
        VARCHAR(255) email UK
        VARCHAR(255) password_hash
        VARCHAR(100) first_name
        VARCHAR(100) last_name
        BOOLEAN is_active
        TIMESTAMP created_at
        TIMESTAMP updated_at

    Organizer {
        UUID id PK
        UUID user_id FK
        VARCHAR(255) organization_name UK
        TEXT description
        VARCHAR(20) status
        TIMESTAMP validated_at
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    Category {
        UUID id PK
        VARCHAR(100) name UK
        VARCHAR(255) slug UK
        TEXT description
    }

    Event {
        UUID id PK
        UUID organizer_id FK
        UUID category_id FK
        VARCHAR(255) title
        TEXT description
        TIMESTAMP start_date
        TIMESTAMP end_date
        VARCHAR(500) location
        INTEGER capacity
        INTEGER available_seats
        NUMERIC(10,2) price
        VARCHAR(20) status
        VARCHAR(500) banner_s3_key
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    Ticket {
        UUID id PK
        UUID user_id FK
        UUID event_id FK
        VARCHAR(20) status
        NUMERIC(10,2) amount_paid
        VARCHAR(10) currency
        TIMESTAMP confirmed_at
        TIMESTAMP cancelled_at
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    Payment {
        UUID id PK
        UUID ticket_id FK
        VARCHAR(255) payment_intent_id UK
        NUMERIC(10,2) amount
        VARCHAR(10) currency
        VARCHAR(30) status
        VARCHAR(30) stripe_status
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    Notification {
        UUID id PK
        UUID user_id FK
        UUID ticket_id FK
        VARCHAR(50) type
        VARCHAR(20) channel
        VARCHAR(20) status
        TEXT payload
        TIMESTAMP sent_at
        TIMESTAMP created_at
    }

    RefreshToken {
        UUID id PK
        UUID user_id FK
        VARCHAR(512) token_hash UK
        TIMESTAMP expires_at
        BOOLEAN revoked
        TIMESTAMP created_at
    }

    User ||--o{ Ticket : "souscrit"
    User ||--o{ Notification : "reçoit"
    User ||--o{ RefreshToken : "possède"
    User ||--|| Organizer : "devient"
    Organizer ||--o{ Event : "organise"
    Category ||--o{ Event : "classe"
    Event ||--o{ Ticket : "génère"
    Ticket ||--o| Payment : "déclenche"
    Ticket ||--o{ Notification : "produit"
```

## A.2 — Dictionnaire de données

### Entité `User`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique de l'utilisateur | Non |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE | Adresse email servant d'identifiant de connexion | Oui |
| `password_hash` | VARCHAR(255) | NOT NULL | Hash bcrypt du mot de passe (facteur de coût 12) | Oui |
| `first_name` | VARCHAR(100) | NOT NULL | Prénom de l'utilisateur | Oui |
| `last_name` | VARCHAR(100) | NOT NULL | Nom de famille de l'utilisateur | Oui |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true | Indique si le compte est actif (soft-delete) | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de création du compte | Non |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de dernière modification | Non |

### Entité `Organizer`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique du profil organisateur | Non |
| `user_id` | UUID | FK → User.id, NOT NULL, UNIQUE | Compte utilisateur associé (1 user = max 1 organizer) | Non |
| `organization_name` | VARCHAR(255) | NOT NULL, UNIQUE | Nom de l'organisation ou association | Non |
| `description` | TEXT | NULL | Description publique de l'organisateur | Non |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'pending', CHECK (status IN ('pending','validated','revoked')) | Statut de validation par un administrateur | Non |
| `validated_at` | TIMESTAMP | NULL | Date de validation administrative | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de création du profil | Non |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de dernière modification | Non |

### Entité `Category`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique de la catégorie | Non |
| `name` | VARCHAR(100) | NOT NULL, UNIQUE | Libellé (ex : "Musique", "Sport") | Non |
| `slug` | VARCHAR(255) | NOT NULL, UNIQUE | Version URL-safe du nom pour le routage front | Non |
| `description` | TEXT | NULL | Description de la catégorie | Non |

### Entité `Event`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique de l'événement | Non |
| `organizer_id` | UUID | FK → Organizer.id, NOT NULL | Organisateur créateur de l'événement | Non |
| `category_id` | UUID | FK → Category.id, NULL | Catégorie de l'événement (optionnelle) | Non |
| `title` | VARCHAR(255) | NOT NULL | Titre public de l'événement | Non |
| `description` | TEXT | NULL | Description détaillée et programme | Non |
| `start_date` | TIMESTAMP | NOT NULL | Date et heure de début | Non |
| `end_date` | TIMESTAMP | NOT NULL, CHECK (end_date > start_date) | Date et heure de fin | Non |
| `location` | VARCHAR(500) | NOT NULL | Adresse ou lieu | Non |
| `capacity` | INTEGER | NOT NULL, CHECK (capacity > 0) | Nombre total de places | Non |
| `available_seats` | INTEGER | NOT NULL, CHECK (available_seats >= 0) | Places restantes (décrémenté atomiquement à chaque inscription) | Non |
| `price` | NUMERIC(10,2) | NOT NULL, DEFAULT 0.00, CHECK (price >= 0) | Prix en euros (0 = gratuit) | Non |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'draft', CHECK (status IN ('draft','published','cancelled','ended')) | Statut du cycle de vie | Non |
| `banner_s3_key` | VARCHAR(500) | NULL | Clé S3/MinIO de l'image de bannière | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de création | Non |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de dernière modification | Non |

### Entité `Ticket`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique du ticket | Non |
| `user_id` | UUID | FK → User.id, NOT NULL | Utilisateur titulaire | Non |
| `event_id` | UUID | FK → Event.id, NOT NULL | Événement concerné | Non |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'pending', CHECK (status IN ('pending','confirmed','cancelled','refunded')) | Statut dans le cycle de vie | Non |
| `amount_paid` | NUMERIC(10,2) | NOT NULL, DEFAULT 0.00 | Montant effectivement payé (0 pour gratuit) | Non |
| `currency` | VARCHAR(10) | NOT NULL, DEFAULT 'EUR' | Devise (ISO 4217) | Non |
| `confirmed_at` | TIMESTAMP | NULL | Horodatage de la confirmation | Non |
| `cancelled_at` | TIMESTAMP | NULL | Horodatage de l'annulation | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de création | Non |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de dernière modification | Non |


### Entité `Payment`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique du paiement côté SupEvents | Non |
| `ticket_id` | UUID | FK → Ticket.id, NOT NULL | Ticket associé | Non |
| `payment_intent_id` | VARCHAR(255) | NOT NULL, UNIQUE | Identifiant PaymentIntent Stripe (ex : `pi_3Pxxx`) | Non |
| `amount` | NUMERIC(10,2) | NOT NULL | Montant en devise | Non |
| `currency` | VARCHAR(10) | NOT NULL, DEFAULT 'EUR' | Devise (ISO 4217) | Non |
| `status` | VARCHAR(30) | NOT NULL, DEFAULT 'initiated' | Statut interne SupEvents | Non |
| `stripe_status` | VARCHAR(30) | NOT NULL | Statut brut retourné par Stripe | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date d'initiation | Non |
| `updated_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de dernière mise à jour | Non |

### Entité `Notification`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique | Non |
| `user_id` | UUID | FK → User.id, NOT NULL | Destinataire | Non |
| `ticket_id` | UUID | FK → Ticket.id, NULL | Ticket à l'origine (si applicable) | Non |
| `type` | VARCHAR(50) | NOT NULL | Type métier (ex : `ticket_confirmed`, `event_cancelled`) | Non |
| `channel` | VARCHAR(20) | NOT NULL, CHECK (channel IN ('email','push','sms')) | Canal utilisé | Non |
| `status` | VARCHAR(20) | NOT NULL, DEFAULT 'pending', CHECK (status IN ('pending','sent','failed')) | Statut d'envoi | Non |
| `payload` | TEXT | NOT NULL | Contenu JSON sérialisé de la notification | Non |
| `sent_at` | TIMESTAMP | NULL | Horodatage d'envoi effectif | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de création | Non |

### Entité `RefreshToken`

| Champ | Type | Contraintes | Description | Sensibilité RGPD |
|---|---|---|---|---|
| `id` | UUID | PK, NOT NULL, DEFAULT gen_random_uuid() | Identifiant unique | Non |
| `user_id` | UUID | FK → User.id, NOT NULL | Utilisateur propriétaire | Non |
| `token_hash` | VARCHAR(512) | NOT NULL, UNIQUE | Hash SHA-256 du refresh token (le token brut n'est jamais stocké) | Oui |
| `expires_at` | TIMESTAMP | NOT NULL | Date d'expiration | Non |
| `revoked` | BOOLEAN | NOT NULL, DEFAULT false | Token révoqué manuellement | Non |
| `created_at` | TIMESTAMP | NOT NULL, DEFAULT now() | Date de création | Non |

## A.3 — Justification du stockage

PostgreSQL stocke les 8 entités principales (User, Event, Ticket, etc.). Il est choisi pour garantir des transactions ACID lors des inscriptions (éviter la survente) et gérer efficacement les jointures nécessaires aux exports et statistiques.

Redis sert à gérer les sessions utilisateurs, l’idempotence des webhooks Stripe, le cache du catalogue d’événements et le rate limiting par IP grâce à des clés temporaires (TTL).

S3 / MinIO stocke les fichiers binaires comme les bannières d’événements et les exports CSV. Les fichiers restent privés et sont accessibles via des URL présignées temporaires.

RabbitMQ gère les événements asynchrones (ticket.confirmed, payment.failed, etc.) afin d’assurer une communication fiable entre services, même en cas de forte charge ou de panne temporaire.

# PHASE B — Contrats d'API & événements


## B.1 — Tableau synoptique des endpoints REST


| Méthode | Chemin | Description | Auth | Codes retour | Dépendances aval |
|---|---|---|---|---|---|
| **— Authentification —** | | | | | |
| `GET` | `/api/v1/auth/login` | Initiation du flux OIDC — redirige vers le provider | Public | `302` `500` | AuthService, OIDC Provider |
| `GET` | `/api/v1/auth/callback` | Callback OIDC — échange le code contre les tokens | Public | `200` `400` (code invalide) `401` (CSRF invalide) `502` (provider KO) | AuthService, OIDC Provider |
| `POST` | `/api/v1/auth/refresh` | Renouvelle l'access token via le refresh token | Public | `200` `400` (body manquant) `401` (token expiré/révoqué) `429` (rate limit) | AuthService, Redis |
| `POST` | `/api/v1/auth/logout` | Révoque le refresh token courant | JWT | `204` `401` | AuthService, Redis, PostgreSQL |
| **— Catalogue d'événements —** | | | | | |
| `GET` | `/api/v1/events` | Liste paginée des événements publiés (filtres : catégorie, date, prix) | Public | `200` `400` (paramètre invalide) `500` | EventService, Redis (cache) |
| `GET` | `/api/v1/events/search` | Recherche full-text dans le catalogue | Public | `200` `400` (query manquante) `500` | EventService |
| `GET` | `/api/v1/events/{id}` | Détail complet d'un événement | Public | `200` `400` (UUID malformé) `404` `500` | EventService |
| **— Inscription à un événement —** | | | | | |
| `POST` | `/api/v1/events/{id}/tickets` | Crée un ticket (déclenche paiement si payant) | JWT | `201` `400` `404` (event inconnu) `409` (déjà inscrit) `422` (plus de places) `500` | TicketService, PaymentService, PostgreSQL |
| `GET` | `/api/v1/tickets/{id}` | Récupère le statut et détails d'un ticket | JWT | `200` `401` `403` (ticket d'un autre user) `404` `500` | TicketService |
| `DELETE` | `/api/v1/tickets/{id}` | Annule un ticket (remboursement si applicable) | JWT | `200` `400` (délai dépassé) `401` `403` `404` `409` (déjà annulé) `500` | TicketService, PaymentService, Stripe |
| **— Paiement —** | | | | | |
| `POST` | `/api/v1/payments/initiate` | Initie un PaymentIntent Stripe — retourne `client_secret` | JWT | `201` `400` `402` `404` (ticket inconnu) `500` `502` (Stripe KO) | PaymentService, Stripe |
| `POST` | `/api/v1/payments/webhook` | Reçoit les événements Stripe (succeeded, failed…) | **HMAC** | `200` `400` (signature invalide) `409` (déjà traité) `500` | PaymentService, TicketService, RabbitMQ, Redis |
| **— Tableau de bord organisateur —** | | | | | |
| `GET` | `/api/v1/organizer/events` | Liste les événements de l'organisateur connecté | JWT + role:organizer | `200` `401` `403` `500` | EventService |
| `POST` | `/api/v1/organizer/events` | Crée un événement (statut `draft`) | JWT + role:organizer | `201` `400` `401` `403` `409` (titre déjà utilisé) `500` | EventService, PostgreSQL |
| `PUT` | `/api/v1/organizer/events/{id}` | Met à jour un événement | JWT + role:organizer | `200` `400` `401` `403` `404` `409` `500` | EventService, PostgreSQL |
| `POST` | `/api/v1/organizer/events/{id}/publish` | Publie un événement (`draft` → `published`) | JWT + role:organizer | `200` `400` (champs manquants) `401` `403` `404` `409` (déjà publié) `500` | EventService |
| `POST` | `/api/v1/organizer/events/{id}/cancel` | Annule un événement ouvert aux inscriptions | JWT + role:organizer | `200` `400` `401` `403` `404` `409` (déjà annulé) `500` | EventService, RabbitMQ |
| `GET` | `/api/v1/organizer/events/{id}/participants` | Liste les participants inscrits | JWT + role:organizer | `200` `401` `403` `404` `500` | TicketService |
| `GET` | `/api/v1/organizer/events/{id}/participants/export` | Export CSV des participants | JWT + role:organizer | `200` `401` `403` `404` `500` | TicketService, S3/MinIO |
| `GET` | `/api/v1/organizer/events/{id}/kpi` | KPI de l'événement (taux remplissage, CA…) | JWT + role:organizer | `200` `401` `403` `404` `500` | AnalyticsService |
| **— Administration plateforme —** | | | | | |
| `GET` | `/api/v1/admin/organizers` | Liste tous les organisateurs avec statut | JWT + role:admin | `200` `401` `403` `500` | OrganizerService |
| `POST` | `/api/v1/admin/organizers/{id}/validate` | Valide un profil organisateur en attente | JWT + role:admin | `200` `401` `403` `404` `409` (déjà validé) `500` | OrganizerService, NotificationService |
| `POST` | `/api/v1/admin/organizers/{id}/revoke` | Révoque les droits d'un organisateur | JWT + role:admin | `200` `401` `403` `404` `409` (déjà révoqué) `500` | OrganizerService, NotificationService |

### Conventions transverses

**Format des erreurs — RFC 7807 Problem Details :**
```json
{
  "type": "https://supevents.io/problems/ticket-already-exists",
  "title": "Conflict",
  "status": 409,
  "detail": "L'utilisateur est déjà inscrit à cet événement.",
  "instance": "/api/v1/events/550e8400-e29b-41d4-a716-446655440000/tickets"
}
```

**Versioning :** intégré dans le chemin (`/api/v1/`). Un changement non rétrocompatible → bump vers `/api/v2/`. Ajout d'un champ optionnel = rétrocompatible, pas de bump.

**Rate limiting :**

| Scope | Limite | Fenêtre |
|---|---|---|
| Lecture publique | 200 req | 1 min / IP |
| Authentification | 10 req | 1 min / IP |
| Actions authentifiées | 100 req | 1 min / user |

Dépassement → `429 Too Many Requests` + header `Retry-After`.

---

## B.2 — Documentation de deux événements asynchrones

### Événement 1 : `ticket.confirmed`

#### Fiche descriptive

| Champ | Valeur |
|---|---|
| **Nom** | `ticket.confirmed` |
| **Producteur** | `TicketService` — publié après confirmation atomique d'un ticket : immédiatement pour un ticket gratuit après vérification des places, ou après réception du webhook Stripe `payment_intent.succeeded` pour un ticket payant |
| **Topic / exchange** | Exchange RabbitMQ `supevents.events` (type `topic`) — routing key : `ticket.confirmed` — version schéma : `v1` |
| **Consommateurs connus** | **NotificationService** : envoie le mail/push de confirmation avec récapitulatif du ticket. **AnalyticsService** : incrémente les compteurs de ventes et met à jour les KPI temps réel de l'organisateur. |
| **Garantie de livraison** | `at-least-once` — les consommateurs doivent être **idempotents** sur `ticket_id` : une double réception ne doit pas générer une double notification. |
| **Stratégie de retry** | Backoff exponentiel : 1s → 5s → 30s → 2min → 10min (max 5 tentatives). Après épuisement → dead letter queue `supevents.dlq` + alerte opérationnelle. |

#### Schéma JSON du payload

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "TicketConfirmedEvent",
  "description": "Publié par TicketService après confirmation d'un ticket payant ou gratuit.",
  "type": "object",
  "required": [
    "schema_version",
    "event_name",
    "ticket_id",
    "user_id",
    "event_id",
    "confirmed_at",
    "amount_paid",
    "currency"
  ],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "v1",
      "description": "Version du schéma pour évolution sans coupure"
    },
    "event_name": {
      "type": "string",
      "const": "ticket.confirmed"
    },
    "ticket_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant unique du ticket confirmé"
    },
    "user_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'utilisateur titulaire"
    },
    "event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'événement concerné"
    },
    "event_title": {
      "type": "string",
      "description": "Titre de l'événement (dénormalisé pour éviter une requête aval)"
    },
    "event_start_date": {
      "type": "string",
      "format": "date-time",
      "description": "Date et heure de début de l'événement (ISO 8601)"
    },
    "confirmed_at": {
      "type": "string",
      "format": "date-time",
      "description": "Horodatage de confirmation du ticket (ISO 8601)"
    },
    "amount_paid": {
      "type": "number",
      "minimum": 0,
      "description": "Montant payé en devise (0 pour un ticket gratuit)"
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "Code devise ISO 4217 (ex : EUR)"
    },
    "payment_intent_id": {
      "type": ["string", "null"],
      "description": "Référence Stripe — null pour un ticket gratuit"
    }
  },
  "additionalProperties": false
}
```

---

### Événement 2 : `payment.failed`

#### Fiche descriptive

| Champ | Valeur |
|---|---|
| **Nom** | `payment.failed` |
| **Producteur** | `PaymentService` — publié lorsque le webhook Stripe reçoit `payment_intent.payment_failed`, ou lorsqu'un timeout de 10 minutes est atteint sans confirmation de paiement |
| **Topic / exchange** | Exchange RabbitMQ `supevents.events` (type `topic`) — routing key : `payment.failed` — version schéma : `v1` |
| **Consommateurs connus** | **TicketService** : repasse le ticket en statut `pending` (ou `cancelled` si délai 30 min dépassé) et réincrémente `Event.available_seats` atomiquement pour remettre la place disponible. **NotificationService** : envoie un email à l'utilisateur l'informant de l'échec et lui proposant de réessayer. |
| **Garantie de livraison** | `at-least-once` — idempotence requise côté `TicketService` sur `payment_intent_id` : si le ticket est déjà `cancelled`, l'événement est ignoré silencieusement. |
| **Stratégie de retry** | Backoff exponentiel : 2s → 10s → 1min → 5min (max 4 tentatives). Dead letter queue `supevents.dlq` avec enrichissement de la raison d'échec pour investigation. |

#### Schéma JSON du payload

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "PaymentFailedEvent",
  "description": "Publié par PaymentService quand un paiement Stripe échoue ou atteint le timeout.",
  "type": "object",
  "required": [
    "schema_version",
    "event_name",
    "ticket_id",
    "user_id",
    "event_id",
    "payment_intent_id",
    "failure_reason",
    "failed_at"
  ],
  "properties": {
    "schema_version": {
      "type": "string",
      "const": "v1"
    },
    "event_name": {
      "type": "string",
      "const": "payment.failed"
    },
    "ticket_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant du ticket dont le paiement a échoué"
    },
    "user_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'utilisateur concerné"
    },
    "event_id": {
      "type": "string",
      "format": "uuid",
      "description": "Identifiant de l'événement concerné"
    },
    "payment_intent_id": {
      "type": "string",
      "description": "Référence Stripe du PaymentIntent en échec"
    },
    "failure_reason": {
      "type": "string",
      "enum": ["card_declined", "insufficient_funds", "expired_card", "timeout", "stripe_error", "unknown"],
      "description": "Raison catégorisée de l'échec"
    },
    "stripe_decline_code": {
      "type": ["string", "null"],
      "description": "Code de refus Stripe brut (ex : 'do_not_honor') — null si timeout"
    },
    "amount": {
      "type": "number",
      "minimum": 0,
      "description": "Montant du paiement ayant échoué"
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "Devise ISO 4217"
    },
    "failed_at": {
      "type": "string",
      "format": "date-time",
      "description": "Horodatage de l'échec (ISO 8601)"
    }
  },
  "additionalProperties": false
}