# Phase 4 - Architecture des Bounded Contexts

## Introduction : De l'Architecture Tactique à l'Architecture Distribuée

La Phase 4 constitue le pont crucial entre l'**architecture tactique** établie en Phase 3 et l'**architecture système** distribuée de WorkSpace+. Elle organise les agrégats identifiés en **Bounded Contexts** cohérents, définit leurs **frontières conceptuelles** et établit les **stratégies d'intégration** nécessaires.

Cette phase ne se contente pas de découper techniquement le système : elle reflète l'**organisation métier réelle** et anticipe les **évolutions futures** (extension aux franchisés, nouveaux canaux de distribution, intégrations partenaires).

L'objectif est de produire une **Context Map** précise qui guide l'implémentation d'une architecture distribuée maintenable, évolutive et alignée avec la stratégie business de WorkSpace+.

## 4.1 Identification des Bounded Contexts

### Méthode d'Analyse

L'identification des Bounded Contexts s'appuie sur plusieurs critères DDD :

1. **Cohérence conceptuelle** : Les concepts métier qui évoluent ensemble
2. **Autonomie fonctionnelle** : Les capacités métier qui peuvent être développées indépendamment  
3. **Responsabilités organisationnelles** : L'alignement avec la structure des équipes métier
4. **Volatilité des changements** : Les domaines avec des rythmes d'évolution similaires
5. **Intégrations externes** : Les frontières naturelles avec les systèmes partenaires

### Architecture des Bounded Contexts WorkSpace+

Basé sur l'analyse tactique précédente et les caractéristiques métier de WorkSpace+, j'identifie **4 Bounded Contexts principaux** :

#### Context 1: **Pricing & Subscription Management** (Core Domain)

**Responsabilité métier :** Gestion des politiques tarifaires intelligentes et des abonnements membres

**Agrégats inclus :**
- **Membre** (avec Abonnement intégré)
- Relations : Collection de Crédits, Historique tarifaire

**Justification du regroupement :**
- **Core Domain** : C'est le domaine différenciant de WorkSpace+ 
- **Cohérence tarifaire** : Abonnements, crédits et calculs de remises forment un tout indissociable
- **Évolution rapide** : Les politiques commerciales évoluent fréquemment et nécessitent une autonomie maximale
- **Expertise métier** : Nécessite des compétences business spécifiques (yield management, loyalty programs)

**Services de domaine :**
- **PricingService** : Calcul intelligent des prix selon contexte
- **SubscriptionLifecycleService** : Gestion automatisée des cycles d'abonnement
- **LoyaltyCalculatorService** : Calcul des statuts et avantages fidélité

**Frontières conceptuelles :**
- **Inclut** : Toute la logique de calcul tarifaire et de gestion d'abonnement
- **Exclut** : La gestion des espaces physiques, les réservations concrètes, la logistique

**Évolution future :** Ce contexte devra intégrer les nouvelles formules d'abonnement (entreprise, famille) et les partenariats commerciaux.

#### Context 2: **Booking & Reservation Management**

**Responsabilité métier :** Gestion du cycle de vie complet des réservations d'espaces

**Agrégats inclus :**
- **Réservation** (avec Value Objects Créneau, Money)
- États : Workflow complet (Provisoire → Confirmée → Annulée/Expirée)

**Justification du regroupement :**
- **Processus métier unitaire** : La réservation a un cycle de vie propre et cohérent
- **Règles de gestion spécifiques** : Gestion des conflits, politiques d'annulation, workflow états
- **Performance critique** : Nécessite optimisations spécifiques pour la concurrence haute
- **Intégrations externes** : Paiements, notifications, systèmes calendrier

**Services de domaine :**
- **ConflictDetectionService** : Prévention des doubles réservations
- **RefundCalculationService** : Calcul des remboursements selon politiques
- **ReservationWorkflowService** : Orchestration des transitions d'état

**Frontières conceptuelles :**
- **Inclut** : Tout le workflow de réservation, gestion des conflits, politique d'annulation
- **Exclut** : Le calcul des prix (délégué au Context Pricing), la gestion des espaces physiques

**Évolution future :** Intégration avec systèmes de réservation partenaires, réservations récurrentes, gestion des événements.

#### Context 3: **Space & Facility Management**

**Responsabilité métier :** Gestion du patrimoine immobilier et de l'administration des sites

**Agrégats inclus :**
- **Site** (avec Espaces intégrés)
- **Espace** (avec caractéristiques et équipements)
- Value Objects : Adresse, HorairesOuverture, PolitiquesLocales

**Justification du regroupement :**
- **Unité physique et administrative** : Les sites et espaces forment une réalité opérationnelle cohérente
- **Expertise métier distincte** : Gestion immobilière, maintenance, optimisation d'occupation
- **Cycle d'évolution différent** : Les espaces physiques évoluent moins rapidement que les politiques tarifaires
- **Responsabilité organisationnelle** : Généralement gérée par des équipes facilities/operations

**Services de domaine :**
- **AvailabilityCalculator** : Calcul dynamique de disponibilité des espaces
- **OccupancyOptimizationService** : Analyse d'occupation et recommandations
- **MaintenanceSchedulingService** : Planification des maintenances

**Frontières conceptuelles :**
- **Inclut** : Gestion des espaces physiques, horaires, maintenance, capacités
- **Exclut** : Les réservations individuelles, les politiques tarifaires, les profils membres

**Évolution future :** Intégration IoT (capteurs d'occupation), gestion énergétique, expansion multi-sites.

#### Context 4: **Analytics & Reporting**

**Responsabilité métier :** Analyse des performances, reporting business et aide à la décision

**Responsabilités :**
- Consolidation des données des autres contextes
- Calculs de KPI métier (taux d'occupation, revenus, satisfaction)
- Reporting pour le management et l'optimisation opérationnelle
- Détection de tendances et recommandations business

**Justification comme contexte séparé :**
- **Nature transversale** : Utilise les données de tous les autres contextes
- **Objectifs différents** : Lecture et analyse vs. écriture opérationnelle
- **Expertise spécifique** : Business Intelligence, data science
- **Contraintes techniques différentes** : Performance de lecture, agrégations complexes

**Services de domaine :**
- **PerformanceAnalysisService** : Calcul des KPI de performance
- **PredictiveAnalyticsService** : Modèles prédictifs d'occupation et revenus
- **ReportingService** : Génération de rapports multi-dimensionnels

**Évolution future :** Machine Learning pour prédictions, tableaux de bord temps réel, analyses prédictives avancées.

## 4.2 Context Map : Relations et Intégrations

La Context Map visualise les relations entre Bounded Contexts et définit les stratégies d'intégration. Chaque relation est caractérisée par un **pattern DDD spécifique**.

### Relations Upstream/Downstream

#### Pricing & Subscription → Booking & Reservation (Customer/Supplier)

**Direction :** Pricing (Upstream) → Booking (Downstream)

**Pattern :** **Customer/Supplier**

**Interface d'intégration :**
```
Booking Service consomme Pricing Service via :
- calculerPrixReservation(EspaceId, Créneau, MembreId) : Money
- verifierEligibilitéMembre(MembreId) : Boolean
- consommerCredits(MembreId, Montant) : CreditConsumptionResult
```

**Justification du pattern :**
- **Pricing** est autonome et dicte ses règles tarifaires
- **Booking** dépend des calculs de prix mais ne peut les influencer
- **Contrat de service** : Pricing s'engage sur ses API pour ne pas casser Booking

**Gestion des évolutions :**
- Pricing publie ses changements d'API avec versioning
- Booking s'adapte aux nouvelles règles tarifaires
- Interface découplée via événements pour les mises à jour de prix

#### Space & Facility → Booking & Reservation (Customer/Supplier)

**Direction :** Space (Upstream) → Booking (Downstream)

**Pattern :** **Customer/Supplier**

**Interface d'intégration :**
```
Booking Service consomme Space Service via :
- verifierDisponibilitéEspace(EspaceId, Créneau) : Boolean
- obtenirCaracteristiquesEspace(EspaceId) : EspaceDetails  
- reserverCreneauProvisoire(EspaceId, Créneau) : ReservationToken
```

**Justification du pattern :**
- **Space** maîtrise la réalité physique des espaces
- **Booking** dépend de ces informations mais ne gère pas les espaces
- **Autonomie** : Space peut modifier ses espaces sans dépendre de Booking

### Relations Conformist

#### Analytics → [Pricing, Booking, Space] (Conformist)

**Direction :** [Tous contextes] (Upstream) → Analytics (Downstream)

**Pattern :** **Conformist**

**Interface d'intégration :**
```
Analytics Service consomme via événements :
- ReservationConfirmed(ReservationId, MembreId, EspaceId, Montant, Timestamp)
- PricingRuleChanged(RuleId, NewRule, EffectiveDate)
- SpaceStatusChanged(EspaceId, Status, Timestamp)
- MemberSubscriptionChanged(MembreId, NewSubscription, Timestamp)
```

**Justification du pattern :**
- **Analytics** n'a aucun pouvoir d'influence sur les contextes opérationnels
- **Conformité** : S'adapte aux modèles de données des autres contextes
- **Résilience** : Peut fonctionner même si certaines données sont temporairement indisponibles

### Anti-Corruption Layer

#### Intégrations Systèmes Externes

Chaque contexte exposed aux intégrations externes utilise des **Anti-Corruption Layers** :

**Pricing Context ↔ Payment Systems**
```
PaymentGatewayACL traduit :
- WorkSpace+ Money → Payment Provider Currency/Amount
- Payment Provider Response → WorkSpace+ PaymentResult
- Gestion des erreurs et retry policies spécifiques
```

**Booking Context ↔ External Calendar Systems**
```
CalendarIntegrationACL traduit :
- WorkSpace+ Créneau → RFC 5545 VEVENT
- External Calendar Event → WorkSpace+ ReservationRequest
- Gestion des conflits et synchronisation bidirectionnelle
```

## 4.3 Architecture Distribuée : Implications Techniques

### Consistency Patterns

#### Pricing ↔ Booking : Eventual Consistency

**Problématique :** Les prix peuvent changer pendant qu'une réservation est en cours.

**Solution choisie :** **Saga Pattern** avec compensation

```
Workflow ReservationSaga :
1. Booking: Créer réservation Provisoire
2. Pricing: Calculer et réserver prix
3. Payment: Traiter paiement  
4. Booking: Confirmer réservation
5. Space: Marquer espace occupé

En cas d'échec à l'étape N :
- Compenser les étapes 1 à N-1
- Libérer les ressources réservées
- Notifier l'utilisateur
```

#### Space ↔ Booking : Strong Consistency

**Problématique :** Les conflits de réservation sur le même espace doivent être évités absolument.

**Solution choisie :** **Distributed Lock** avec timeout

```
ConflictPrevention Protocol :
1. Booking demande lock sur EspaceId+Créneau
2. Space accorde lock pour durée limitée (10 min)  
3. Booking finalise réservation ou libère lock
4. Space maintient registre des locks actifs
```

### Communication Patterns

#### Synchronous vs Asynchronous

**Synchrone (Request/Response) :**
- Pricing calculations (temps réel requis)
- Space availability checks (cohérence critique)
- Payment processing (confirmation immédiate)

**Asynchrone (Événements) :**
- Analytics data ingestion (tolérance à la latence)
- Notification systems (fire-and-forget)
- Audit logging (eventual consistency acceptable)

#### Event Sourcing pour Analytics

```
Event Store Architecture :
- Tous les contextes publient leurs événements métier
- Analytics reconstruit ses vues via projection
- Possibilité de replay pour nouveaux KPI
- Audit trail complet pour conformité
```

## 4.4 Évolution et Extension : Préparation Franchisage

L'architecture des Bounded Contexts anticipe l'**extension aux franchisés** mentionnée dans le contexte métier.

### Pattern Multi-Tenant vs Multi-Instance

**Décision architecturale :** **Multi-Instance par franchisé** avec services partagés

**Justification :**
- **Isolation des données** : Chaque franchisé maîtrise ses données clients
- **Customisation** : Politiques tarifaires spécifiques par franchise
- **Scalabilité** : Croissance indépendante par région
- **Résilience** : Panne d'un franchisé n'affect pas les autres

### Shared Services Architecture

```
├── Pricing Context (instance par franchisé)
├── Booking Context (instance par franchisé)  
├── Space Context (instance par franchisé)
└── Shared Services (mutualisés)
    ├── Payment Gateway Integration
    ├── Identity & Authentication
    ├── Notification Service
    └── Analytics Aggregation
```

### Federation Pattern pour Analytics

```
Global Analytics Context :
- Agrège les données de tous les franchisés
- Benchmarking inter-franchises
- Optimisations réseau globales
- Respect RGPD avec anonymisation
```

## Conclusion de la Phase 4

Cette architecture de Bounded Contexts transforme l'analyse tactique en **blueprint d'implémentation distribuée**.

### Décisions Architecturales Clés

1. **4 Bounded Contexts** avec responsabilités métier distinctes
2. **Context Map précise** avec patterns d'intégration justifiés
3. **Consistency patterns** adaptés aux contraintes métier
4. **Architecture évolutive** préparant l'extension franchisés

### Alignement Stratégique Maintenu

- **Core Domain** : Pricing & Subscription bénéficie de l'autonomie maximale
- **Supporting Domains** : Booking et Space avec architectures adaptées
- **Scalabilité** : Préparation multi-tenant pour croissance

### Benefits Architecturaux

- **Team Autonomy** : Équipes peuvent développer indépendamment leurs contextes
- **Technology Flexibility** : Chaque contexte peut choisir sa stack optimale  
- **Fault Isolation** : Panne d'un contexte n'impacte pas les autres
- **Business Alignment** : Architecture reflète l'organisation métier réelle

Cette Phase 4 établit l'**architecture système** sur laquelle construire l'implémentation distribuée de WorkSpace+, parfaitement alignée avec la stratégie métier et préparant les évolutions futures.