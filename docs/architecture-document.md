# Document d'Architecture - WorkSpace+

## 1. Résumé exécutif

WorkSpace+ est une plateforme SaaS de gestion d'espaces de coworking dont l'avantage concurrentiel repose sur la **tarification intelligente** et l'**optimisation d'occupation**.

Le présent document formalise l'architecture cible du système selon une approche Domain-Driven Design (DDD), consolidée en six phases : stratégie, langage ubiquitaire, modélisation tactique, Bounded Contexts, validation par cas d'usage et synthèse finale.

L'architecture proposée vise trois objectifs prioritaires :

1. protéger les invariants métier critiques (conflits de réservation, intégrité financière, contrôle d'accès) ;
2. permettre l'évolution rapide des règles tarifaires (Core Domain) ;
3. garantir la scalabilité de la plateforme (distribution, intégrations externes, projection analytics).

---

## 2. Contexte métier et objectifs

### 2.1 Enjeux métier

- gérer des espaces hétérogènes (bureaux privés, salles, open-space) ;
- réserver à l'heure ou à la journée ;
- appliquer des politiques tarifaires dynamiques selon abonnement, durée et contexte ;
- éviter toute double réservation ;
- gérer paiements, annulations et remboursements ;
- préparer l'extension vers des partenaires franchisés.

### 2.2 Objectifs d'architecture

- refléter le métier et non un simple CRUD ;
- isoler la logique différenciante du cœur business ;
- réduire le couplage entre domaines ;
- favoriser la maintenabilité et l'évolutivité.

---

## 3. Vision DDD globale

### 3.1 Core Domain

**Core Domain identifié :** Politique tarifaire intelligente et optimisation d'occupation.

Ce domaine concentre l'investissement principal car il porte la valeur stratégique de WorkSpace+ (revenu, conversion, compétitivité).

### 3.2 Sous-domaines

- **Supporting Domains** : Booking/Reservation, Space/Facility.
- **Generic Subdomains** : composants transverses techniques (ex. intégrations externes, projections analytics).

### 3.3 Ubiquitous Language

Les concepts métier ont été unifiés et stabilisés : `Réservation`, `Créneau`, `Disponibilité`, `Membre`, `Abonnement`, `Crédit`, `Espace`, `Site`.

La gouvernance terminologique est portée par `docs/glossary.md`.

---

## 4. Modélisation tactique

### 4.1 Entités clés

- **Membre** : identité métier client, statut et droits.
- **Abonnement** : cycle de vie propre (Actif, Suspendu, Expiré, Résilié).
- **Réservation** : cycle de vie `Provisoire -> Confirmée -> Annulée/Expirée`.
- **Espace** : ressource réservable.
- **Site** : agrégation des espaces et politiques locales.

### 4.2 Value Objects clés

- **Money** : sécurité des calculs monétaires.
- **Créneau** : cohérence temporelle.
- **Crédit** : unités de consommation d'abonnement.
- **Adresse** : données géographiques immuables.

### 4.3 Agrégats et invariants

- **Agrégat Membre** (incluant Abonnement)
- **Agrégat Réservation**
- **Agrégat Site** (incluant Espaces)

Invariants majeurs :

- pas de réservation confirmée sans paiement validé ;
- pas de double réservation sur créneau chevauchant ;
- pas de consommation de crédit invalide ;
- pas de nouvelles réservations si abonnement suspendu.

### 4.4 Services de domaine

- **PricingService**
- **AvailabilityCalculator**
- **RefundCalculationService**

Ces services encapsulent la logique métier transversale non attribuable à un seul agrégat.

---

## 5. Architecture des Bounded Contexts

### 5.1 Contextes définis

1. **Pricing & Subscription Management** (Core)
2. **Booking & Reservation Management**
3. **Space & Facility Management**
4. **Analytics & Reporting**

### 5.2 Relations et patterns d'intégration

- `Pricing -> Booking` : **Customer/Supplier**
- `Space -> Booking` : **Customer/Supplier**
- `Analytics <- [Pricing, Booking, Space]` : **Conformist**
- systèmes externes : **Anti-Corruption Layers** (paiement, calendrier, IoT futur)

### 5.3 Consistance distribuée

- **Strong consistency** via lock distribué pour conflits de réservation.
- **Saga + compensation** pour les workflows longs (réservation/paiement/remboursement).
- **Eventual consistency** pour analytics et propagation de certains états.

---

## 6. Validation par cas d'usage

Le modèle a été vérifié sur 5 scénarios critiques :

1. réservation confirmée ;
2. conflit concurrent ;
3. annulation/remboursement ;
4. suspension d'abonnement ;
5. changement tarifaire pendant réservation provisoire.

Résultats : invariants principaux validés, cohérence inter-contextes confirmée, et recommandations de fiabilisation identifiées.

---

## 7. Décisions techniques structurantes

1. **Réservation Provisoire avec TTL** pour contrôler concurrence et expérience utilisateur.
2. **Lock distribué** sur `EspaceId + Créneau` pour interdire les collisions.
3. **Tarification dynamique centralisée** dans le Core Domain.
4. **ACL systématiques** pour protéger le modèle interne des fournisseurs externes.
5. **Architecture orientée événements** pour projections analytics et découplage.

---

## 8. Risques et recommandations

### 8.1 Risques résiduels

- dérive de tarification pendant paiement en cours ;
- doubles traitements lors de retries externes ;
- dérive de contrats d'événements entre contextes ;
- manque de SLA explicites sur lock distribué.

### 8.2 Recommandations prioritaires

1. introduire un `TarificationSnapshot` versionné côté Booking ;
2. imposer des clés d'idempotence métier pour paiement/remboursement ;
3. versionner officiellement les événements de domaine ;
4. formaliser SLA/alerting pour lock distribué.

---

## 9. Plan d'implémentation recommandé

### Lot A - Contrats et fondations

- APIs inter-contextes ;
- schémas d'événements ;
- ACL paiements/calendrier.

### Lot B - Cœur métier

- implémentation pricing/remises ;
- workflow réservation provisoire/confirmation ;
- prévention des conflits.

### Lot C - Fiabilisation

- sagas de compensation ;
- idempotence et retries ;
- tests automatiques d'invariants métier.

### Lot D - Pilotage et optimisation

- projections analytics ;
- KPI revenus/occupation ;
- amélioration continue des politiques tarifaires.

---

## 10. Références de livrables

### Analyses

- `analysis/phase-1-core-domain.md`
- `analysis/phase-2-ubiquitous-language.md`
- `analysis/phase-3-tactical-modeling.md`
- `analysis/phase-4-bounded-contexts.md`
- `analysis/phase-5-use-cases.md`
- `analysis/phase-6-synthesis.md`

### Diagrammes

- `diagrams/domain-overview.mmd`
- `diagrams/entities-value-objects.mmd`
- `diagrams/aggregates.mmd`
- `diagrams/context-map.mmd`
- `diagrams/use-cases-flow.mmd`

### Documentation

- `docs/glossary.md`
- `docs/workflow-guidelines.md`
- `docs/project-roadmap.md`
- `docs/project-structure.md`

---

## Conclusion

L'architecture WorkSpace+ issue de ce TP DDD est prête pour une implémentation incrémentale robuste. Elle aligne stratégie business, modélisation métier et contraintes de distribution moderne.

Le design proposé sécurise les points critiques (conflits, paiement, cohérence) tout en laissant une forte marge d'évolution sur la tarification, la croissance multi-sites et l'ouverture à des franchisés.