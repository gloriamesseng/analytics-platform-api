# Product Requirements Document (PRD) : Analytics Platform API

## Vision & Stratégie (MVP - 4 Semaines)
L'objectif de ce MVP est de valider l'architecture asynchrone, la résilience sous forte charge (10K utilisateurs concurrents) et la distribution des données en temps réel. La sécurité et les exports lourds sont relégués en fin de cycle pour prioriser la "plomberie" critique (Ingestion -> Broker -> Workers -> DB/Redis -> WebSockets).

---

## Epic 1 : Data Simulation & Traffic Generation Engine (Le Mocker)
*Objectif : Fournir le carburant nécessaire pour tester les limites du système.*

*   **US 1.1 : Génération de la donnée de référence**
    *   *En tant que* développeur, *je veux* générer ou importer un dataset massif (comptes, agences, utilisateurs) *afin d'avoir* un référentiel réaliste.
    *   **Critères d'Acceptation (AC) :**
        *   Le script doit pouvoir générer au moins 100 000 comptes distincts répartis sur différentes agences/régions.
        *   Les données doivent être injectées directement en base de données pour préparer l'environnement.

*   **US 1.2 : Simulation de charge et d'anomalies (Stress Test)**
    *   *En tant qu'* ingénieur QA, *je veux* lancer un script simulant des envois concurrents de transactions *afin de* tester l'API sous contrainte.
    *   **AC :**
        *   Le script doit utiliser l'asynchronisme (ex: multithreading/asyncio) pour simuler jusqu'à 10 000 requêtes/seconde.
        *   Le script doit délibérément envoyer des doublons exacts (même `transaction_id`) pour tester l'idempotence.
        *   Le script doit délibérément générer des scénarios de fraude (ex: vélocité, smurfing).

---

## Epic 2 : High-Throughput Transaction Ingestion (API de Réception)
*Objectif : Capter la donnée sans jamais bloquer le client.*

*   **US 2.1 : End-point d'ingestion asynchrone**
    *   *En tant que* système bancaire, *je veux* envoyer une transaction à l'API et recevoir une confirmation instantanée *afin de* ne pas bloquer mes propres processus.
    *   **AC :**
        *   L'API doit valider strictement le format (JSON schema) de la transaction entrante. Les payloads invalides retournent une erreur `400 Bad Request` immédiatement.
        *   Si le payload est valide, l'API **ne doit pas** insérer la donnée en base. Elle doit la pousser dans un Message Broker et retourner un statut `202 Accepted` en moins de 20ms.

---

## Epic 3 : Asynchronous Processing & Data Persistence (Workers)
*Objectif : Traiter la donnée de manière fiable et résiliente.*

*   **US 3.1 : Dépilage et garantie d'Idempotence**
    *   *En tant que* système backend, *je veux* traiter les transactions de la file d'attente *afin de* les persister définitivement.
    *   **AC :**
        *   Les workers doivent dépiler les messages à leur propre rythme (Pull ou Subscribe).
        *   **Idempotence stricte :** Si un `transaction_id` existe déjà en base, l'insertion doit échouer "silencieusement" (le doublon est ignoré, le message est marqué comme traité). La garantie doit reposer sur la base de données, pas sur un cache.
        *   En cas d'échec de traitement (ex: base inaccessible), le message doit être rejoué X fois, puis envoyé dans une Dead Letter Queue (DLQ).

---

## Epic 4 : Fraud & Anomaly Detection Engine (Moteur de Règles)
*Objectif : Identifier les comportements suspects à la volée.*

*   **US 4.1 : Évaluation des règles métier**
    *   *En tant que* Risk Analyst, *je veux* que le système flag les transactions anormales *afin de* les analyser.
    *   **AC :**
        *   L'évaluation doit se faire de manière asynchrone (soit par le même worker qui insère, soit via un event publié après l'insertion).
        *   Le moteur doit requêter l'historique récent (via la DB ou Redis) pour valider des règles complexes (ex: "plus de 5 transactions dans les 2 dernières minutes pour ce compte").
        *   Les transactions suspectes sont marquées avec un statut `is_anomaly = true` et un code de motif.

---

## Epic 5 : Real-Time Alerting (WebSockets)
*Objectif : Pousser l'information critique sans attendre de rafraîchissement manuel.*

*   **US 5.1 : Diffusion des alertes de fraude**
    *   *En tant que* Manager, *je veux* voir les alertes de fraude apparaître instantanément sur mon écran *afin de* réagir vite.
    *   **AC :**
        *   Dès qu'une transaction est flaguée par l'Epic 4, un événement doit être poussé sur un canal WebSocket.
        *   Le système de WebSockets doit gérer les déconnexions/reconnexions (Heartbeat / Ping-Pong).

---

## Epic 6 : High-Performance Analytical Dashboards (Consultation)
*Objectif : Afficher les KPIs avec une latence quasi-nulle.*

*   **US 6.1 : Lecture des KPIs à froid (Cache-First)**
    *   *En tant que* Business Analyst, *je veux* consulter les totaux quotidiens *afin de* suivre l'activité.
    *   **AC :**
        *   Le endpoint `GET /kpis` **doit répondre en moins de 50ms**.
        *   Pour atteindre ce temps, la donnée doit être lue exclusivement depuis un système de cache en mémoire (Redis).
        *   Le cache doit être mis à jour par des workers en arrière-plan (soit à chaque nouvelle transaction validée, soit via un job planifié toutes les X secondes), jamais par la requête HTTP de lecture.

---

## Epic 7 : Identity, Access & Security Management (IAM & Scopes)
*Objectif : Sécuriser la plateforme et cloisonner les données (À implémenter après le MVP technique).*

*   **US 7.1 : Authentification et RBAC**
    *   *En tant qu'* utilisateur, *je veux* m'authentifier *afin d'* accéder à l'API.
    *   **AC :** 
        *   Authentification obligatoire via token JWT.
        *   Les endpoints d'administration doivent rejeter (HTTP 403) les rôles non autorisés.

*   **US 7.2 : Isolation des données (ABAC)**
    *   *En tant que* Branch Manager, *je veux* que les KPIs et les alertes soient filtrés *afin de* ne voir que mon agence.
    *   **AC :**
        *   Les requêtes de lecture et les flux WebSockets doivent utiliser le scope du token JWT pour injecter dynamiquement des filtres (ex: `WHERE branch_id = X`). Aucune donnée hors scope ne doit transiter.

---

## Epic 8 : Asynchronous Reporting & Exports 
*Objectif : Gérer des tâches longues sans timeout HTTP.*

*   **US 8.1 : Export massif asynchrone**
    *   *En tant qu'* auditeur, *je veux* exporter tout l'historique d'une agence sur 1 an *afin de* l'étudier hors ligne.
    *   **AC :**
        *   L'API retourne un `HTTP 202 Accepted` avec un `job_id`.
        *   Un worker génère le fichier CSV en arrière-plan (gestion par chunks pour ne pas exploser la RAM).
        *   L'utilisateur peut pinger un endpoint `GET /exports/{job_id}` pour connaître le statut (Pending, Processing, Completed) et récupérer le lien de téléchargement final.