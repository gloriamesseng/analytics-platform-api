# Project Brief : Analytics Platform API (Banking & Finance)

## 1. Vision du Produit
La plateforme est un système centralisé d'analyse de données bancaires. Elle a pour but de capter des flux de transactions financières à haute fréquence, d'en assurer la traçabilité stricte, de détecter des anomalies (fraudes potentielles) en quasi-temps réel, et de fournir des indicateurs de performance (KPIs) fiables pour la prise de décision managériale.

## 2. Origine et Cycle de Vie de la Donnée
*   **Origine :** Systèmes bancaires simulés (générateurs de données à grande échelle) frappant les endpoints d'ingestion de l'API.
*   **Cycle de vie (Chemin critique) :**
    1.  **Réception :** L'API reçoit la transaction, valide le format de base.
    2.  **Acceptation rapide :** L'API répond immédiatement avec un `HTTP 202 Accepted` pour libérer la connexion.
    3.  **Mise en file d'attente :** La transaction est poussée dans un Message Broker / Queue.
    4.  **Traitement (Workers) :** Des workers en arrière-plan dépilent les transactions, vérifient les règles métier, et les insèrent de manière définitive dans la base de données.
    5.  **Analyse & Alerting :** En parallèle, les transactions sont évaluées par le moteur de règles de fraude. En cas d'anomalie, une alerte est poussée en temps réel.

## 3. Acteurs et Isolation des Données (RBAC & ABAC)
La sécurité repose sur une combinaison de rôles et de périmètres organisationnels/géographiques :
*   **Admin :** Accès global à toutes les régions, agences et transactions.
*   **Regional Manager :** Accès limité aux agences et transactions de sa région.
*   **Branch Manager :** Accès restreint uniquement aux opérations de sa succursale.
*   **Risk & Fraud Analyst :** Accès focalisé sur les transactions flaguées comme "anomalies" dans son périmètre assigné.
*   **Business Analyst :** Accès en lecture sur des vues analytiques agrégées (souvent anonymisées ou macrodonnées).
*   **Auditeur :** Accès en lecture seule sur les traces d'audit et l'historique inaltérable des transactions.

## 4. Cas d'Usage Principaux et Choix Technologiques Induits
*   **Ingestion massive et Idempotence :** Capacité à absorber 10K utilisateurs concurrents via des traitements asynchrones. Garantie d'idempotence via une contrainte d'unicité forte (`transaction_id`) au niveau de la base de données.
*   **Détection d'Anomalies (Règles métier) :**
    *   Montants inhabituels ou "Smurfing" (ex: 10 transactions de 95 000 en 10 min).
    *   Incohérences géographiques temporelles (impossible travel).
    *   Vélocité anormale (>5 transactions en <2 min).
    *   Comportements suspects (retrait massif immédiat après création de compte, multiples bénéficiaires en <1h, succès après de multiples échecs).
*   **Alerting Temps Réel (WebSockets) :** Remontée instantanée des alertes de fraude sur les dashboards des Risk Analysts et Managers.
*   **Reporting Asynchrone (Background Jobs) :** Génération de rapports lourds et exports Excel/PDF réalisés en arrière-plan sans bloquer les requêtes HTTP.
*   **Dashboards Hautes Performances (Redis) :** Mise en cache des compteurs quotidiens et des KPIs pour garantir un affichage de l'accueil du dashboard en moins de 50ms.

## 5. Défis Techniques Identifiés
1.  **Data Consistency :** S'assurer qu'aucune transaction n'est perdue ou dupliquée malgré les échecs réseau ou les crashs des workers.
2.  **Scalabilité Horizontale :** Le système doit pouvoir ajouter des nœuds API et des Workers sans créer de conflits d'écriture.
3.  **Performances de Lecture vs Écriture :** Séparer conceptuellement la charge d'écriture massive (ingestion) de la charge de lecture analytique (dashboards/reporting).
4.  **Application stricte des Scopes (Sécurité) :** Éviter à tout prix l'injection de failles permettant à un utilisateur d'un périmètre d'accéder aux données d'un autre (Data Leakage).