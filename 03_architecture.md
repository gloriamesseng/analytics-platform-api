# Architecture Technique : Analytics Platform API

## 1. Stack Technique
*   **API Gateway & Endpoints :** Python + FastAPI (Choisi pour ses performances asynchrones natives, Pydantic pour la validation stricte, et la gestion des WebSockets).
*   **Message Broker :** Apache Kafka (ou Redpanda). (Choisi pour absorber des pics massifs d'ingestion sans perte de données, avec un modèle *append-only*).
*   **Base de Données (OLTP/Persistance) :** PostgreSQL. (Choisi pour les transactions ACID, les contraintes d'unicité garantissant l'idempotence, et le support du *Row-Level Security* pour l'ABAC).
*   **Cache & Pub/Sub (Dashboarding & Alerting) :** Redis. (Choisi pour stocker les agrégats financiers et répondre aux requêtes de KPIs en moins de 50ms, et pour router les alertes via son Pub/Sub).
*   **Background Workers :** Python pur (connecté aux topics Kafka). (Choisi pour séparer physiquement le calcul lourd de la réception HTTP).

---

## 2. Modèle de Données (Database Schema - PostgreSQL)

*Le schéma est optimisé pour éviter les Race Conditions et faciliter la détection de fraudes.*

*   `regions` (region_id [PK], name)
*   `branches` (branch_id [PK], region_id [FK], name)
*   `accounts` (account_id [PK], branch_id [FK], status, created_at)
*   `transactions`
    *   `internal_id` (BigSerial, PK)
    *   `transaction_id` (Varchar, **UNIQUE** - *Garantie Idempotence*)
    *   `account_id` (UUID, FK -> accounts)
    *   `amount` (Decimal), `currency` (Varchar)
    *   `location_lat` (Float), `location_lon` (Float)
    *   `status` (Enum), `is_anomaly` (Boolean)
    *   `timestamp` (Timestamptz)
    *   *Index critiques :* `(account_id, timestamp)` pour la recherche de vélocité.
*   `fraud_alerts` (alert_id [PK], transaction_id [FK], reason_code, resolved)

---

## 3. Flux Système & Cycles de Vie (System Design)

### Flux A : Ingestion et Traitement Asynchrone (Le Chemin Critique)
1.  **Réception HTTP :** Le "Mocker" envoie un POST sur `/api/v1/transactions`.
2.  **Validation :** FastAPI valide le payload (Pydantic).
3.  **Mise en Queue :** FastAPI produit un message JSON dans le topic Kafka `raw-transactions`.
4.  **Réponse HTTP :** FastAPI retourne un `202 Accepted` au client. (Durée totale < 20ms).
5.  **Dépilage :** Le groupe de *Workers Transactionnels* consomme le topic `raw-transactions`.
6.  **Idempotence & Insertion :** Le Worker tente un `INSERT` dans PostgreSQL. 
    *   *Cas nominal :* Succès.
    *   *Race condition/Doublon :* PostgreSQL lève une exception d'unicité sur `transaction_id`. Le Worker l'attrape, ignore silencieusement, et commit l'offset Kafka (ACK).

### Flux B : Moteur de Fraude & WebSockets
1.  **Déclenchement :** Une fois la transaction insérée en DB, un message est poussé dans le topic Kafka `validated-transactions`.
2.  **Évaluation :** Le *Fraud Worker* dépile, vérifie les règles (requête SQL rapide sur les 5 dernières minutes avec l'index `account_id, timestamp`).
3.  **Alerte :** Si une fraude est détectée :
    *   Update de la table `transactions` (`is_anomaly = True`).
    *   Insertion dans `fraud_alerts`.
    *   Envoi d'un message JSON dans **Redis Pub/Sub** sur le canal `alerts:{branch_id}`.
4.  **WebSockets :** Les serveurs FastAPI maintiennent des connexions WebSockets avec les navigateurs des Managers. Ils sont abonnés à Redis Pub/Sub et poussent l'alerte au client instantanément.

### Flux C : Consultation Haute Performance (Dashboards)
1.  **Mise à jour du Cache :** Les *Workers Transactionnels* incrémentent des compteurs dans Redis (ex: `INCR daily_tx_count:{branch_id}`) de manière asynchrone pour chaque transaction.
2.  **Lecture :** Le Manager ouvre son dashboard. Le navigateur fait un GET sur `/api/v1/analytics/kpis`.
3.  **Restitution :** FastAPI lit directement la clé Redis (sans toucher PostgreSQL) et retourne les datas (Durée < 50ms).

---

## 4. Stratégie de Sécurité (RBAC / ABAC)
*   **JWT Claims :** Chaque token contient les rôles (ex: `branch_manager`) et les scopes (ex: `branch_id: 12345`).
*   **Isolation Backend :** FastAPI extrait le `branch_id` du token et l'injecte dans la requête de KPIs (clé Redis spécifique) ou s'abonne uniquement au canal WebSocket de cette agence.
*   **Isolation Database (Row-Level Security) :** Pour les exports et requêtes analytiques profondes, le script injecte le contexte utilisateur à PostgreSQL via `SET LOCAL application.current_branch_id = '12345';`. Les policies RLS de PostgreSQL empêcheront toute lecture hors périmètre, même si la requête SQL est mal codée.