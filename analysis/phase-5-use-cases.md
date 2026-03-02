# Phase 5 - Validation du Modèle par Cas d'Usage

## Introduction : Vérifier le modèle DDD en conditions réelles

Après la modélisation tactique (Phase 3) et l'architecture des Bounded Contexts (Phase 4), cette phase valide que le modèle WorkSpace+ fonctionne correctement sur des **scénarios métier réalistes**.

L'objectif n'est pas de tester une implémentation technique, mais de vérifier la **cohérence métier** du modèle :

- les invariants sont bien protégés ;
- les agrégats coopèrent correctement ;
- les Bounded Contexts s'intègrent sans contradiction ;
- les cas limites (concurrence, annulation, suspension) sont gérables.

---

## 5.1 Méthodologie de validation

La validation repose sur des cas d'usage orientés métier, dérivés directement du glossaire et des règles établies dans les phases précédentes.

### Axes de validation

1. **Validité fonctionnelle** : le flux métier aboutit au résultat attendu.
2. **Protection des invariants** : les règles critiques ne peuvent pas être contournées.
3. **Cohérence inter-contextes** : les échanges entre contexts restent maîtrisés.
4. **Résilience** : les erreurs et interruptions sont traitées sans incohérence.

### Références utilisées

- `analysis/phase-3-tactical-modeling.md`
- `analysis/phase-4-bounded-contexts.md`
- `docs/glossary.md`
- `diagrams/aggregates.mmd`
- `diagrams/context-map.mmd`

---

## 5.2 Matrice de couverture

| Cas d'usage | Objectif métier | Contexts impliqués | Invariants validés | Pattern de consistance |
|---|---|---|---|---|
| UC-01 Réservation confirmée | Réserver un espace avec paiement réussi | Booking, Space, Pricing, Analytics | Pas de confirmation sans paiement ; pas de créneau invalide | Saga + lock distribué |
| UC-02 Conflit concurrent | Empêcher la double réservation | Booking, Space | Un espace ne peut pas être réservé deux fois sur un créneau chevauchant | Strong consistency |
| UC-03 Annulation & remboursement | Appliquer la politique d'annulation selon délai/abonnement | Booking, Pricing, Payment (ACL), Analytics | Remboursement cohérent et borné ; transitions d'état valides | Saga compensatoire |
| UC-04 Suspension abonnement | Bloquer les nouvelles réservations si membre Suspendu | Pricing, Booking, Analytics | Membre Suspendu => pas de nouvelle réservation | Eventual consistency pilotée par événements |
| UC-05 Changement tarifaire en cours | Garantir cohérence tarifaire durant une réservation Provisoire | Pricing, Booking, Analytics | Traçabilité de la règle tarifaire appliquée ; stabilité du montant pendant TTL | Eventual consistency + versioning |

---

## 5.3 Cas d'usage détaillés

## UC-01 - Réserver un espace avec abonnement actif

### Acteurs

- Membre (principal)
- Booking & Reservation Context
- Space & Facility Context
- Pricing & Subscription Context
- Système de paiement (via ACL)
- Analytics Context

### Préconditions

- Le membre est `Actif`.
- Le créneau demandé respecte les horaires du site.
- Aucun lock actif concurrent sur le même créneau.

### Flux nominal

1. Le membre sélectionne un espace et un créneau.
2. Booking crée une **Réservation Provisoire** (TTL de 10 minutes).
3. Booking demande à Space la disponibilité et l'obtention du lock distribué.
4. Booking interroge Pricing pour calculer le prix final (remises incluses).
5. Booking déclenche le paiement via l'ACL Payment.
6. Si paiement OK, Booking confirme la réservation.
7. Space marque le créneau comme occupé.
8. Booking publie l'événement `ReservationConfirmed`.
9. Analytics met à jour ses projections.

### Extensions / erreurs

- **Paiement refusé** : compensation (libération lock, réservation marquée `Expirée` ou `Annulée`).
- **Timeout du statut Provisoire** : expiration automatique et libération du créneau.

### Invariants validés

- Une réservation ne passe à `Confirmée` qu'avec paiement validé.
- Le créneau doit rester disponible jusqu'à confirmation.
- Un membre ne possède qu'une seule réservation `Provisoire` simultanée.

### Verdict

✅ Le modèle supporte un flux nominal robuste et protège les invariants critiques de réservation.

---

## UC-02 - Conflit de réservation concurrente

### Acteurs

- Membre A
- Membre B
- Booking Context
- Space Context

### Préconditions

- Deux demandes arrivent quasi simultanément sur le même espace et le même créneau.

### Flux nominal

1. Les deux demandes atteignent Booking.
2. Booking tente d'obtenir le lock distribué auprès de Space.
3. Une seule demande obtient le lock (ex. Membre A).
4. Le flux de Membre A continue (UC-01).
5. La demande de Membre B reçoit un refus immédiat de disponibilité.

### Extensions / erreurs

- Si Membre A abandonne avant paiement, lock libéré et Membre B peut relancer.
- Si timeout lock atteint, libération automatique pour éviter le blocage.

### Invariants validés

- Aucune double réservation sur un même créneau.
- Le verrouillage est limité dans le temps et ne bloque pas durablement le système.

### Verdict

✅ La stratégie de strong consistency (lock distribué + TTL) est cohérente avec les exigences métier.

---

## UC-03 - Annulation et remboursement

### Acteurs

- Membre
- Booking Context
- RefundCalculationService
- Pricing & Subscription Context
- Système de paiement (ACL)
- Analytics Context

### Préconditions

- Réservation en état `Confirmée`.
- Membre authentifié et autorisé à annuler.

### Flux nominal

1. Le membre demande l'annulation.
2. Booking vérifie l'éligibilité avec `RefundCalculationService`.
3. Le montant remboursable est calculé selon délai d'annulation et type d'abonnement.
4. L'ACL paiement exécute le remboursement.
5. Booking passe la réservation à l'état `Annulée`.
6. Space libère le créneau.
7. Booking publie `ReservationCancelled` pour Analytics.

### Extensions / erreurs

- **Hors fenêtre d'annulation gratuite** : remboursement partiel ou nul.
- **Échec technique remboursement** : réservation reste en état intermédiaire contrôlé avec retry/idempotence.

### Invariants validés

- Le remboursement ne dépasse jamais le montant payé.
- Les transitions `Confirmée -> Annulée` respectent les règles métier.
- L'espace est effectivement libéré après annulation valide.

### Verdict

✅ Le modèle couvre correctement la politique d'annulation et garantit l'intégrité financière.

---

## UC-04 - Suspension d'abonnement et impact réservation

### Acteurs

- Pricing & Subscription Context
- Booking Context
- Membre
- Analytics Context

### Préconditions

- Échec de renouvellement ou incident de facturation.

### Flux nominal

1. Pricing détecte l'incident de facturation.
2. `SubscriptionLifecycleService` passe l'abonnement à `Suspendu`.
3. Événement `MemberSubscriptionChanged` publié.
4. Booking reçoit l'information et refuse toute nouvelle réservation.
5. Le membre reçoit un message explicite avec action attendue (régularisation).

### Extensions / erreurs

- Après régularisation, statut remis à `Actif` puis réservations réautorisées.

### Invariants validés

- Un membre `Suspendu` ne peut pas initier de nouvelle réservation.
- Le statut d'abonnement est propagé de manière traçable entre contexts.

### Verdict

✅ Le modèle applique correctement la règle d'accès liée au statut d'abonnement.

---

## UC-05 - Changement tarifaire pendant une réservation Provisoire

### Acteurs

- Membre
- Booking Context
- Pricing Context
- Analytics Context

### Préconditions

- Une réservation Provisoire est en cours de paiement.
- Une nouvelle règle tarifaire est activée en parallèle.

### Flux nominal

1. Pricing calcule un montant avec une version de règle `Vn`.
2. Booking conserve ce montant durant la fenêtre Provisoire (TTL).
3. Une nouvelle règle `Vn+1` est publiée pendant ce délai.
4. Si le paiement est finalisé dans le TTL, Booking confirme au prix `Vn`.
5. Si le TTL est dépassé, nouvelle tentative avec recalcul au prix `Vn+1`.

### Extensions / erreurs

- Si différence de prix significative, confirmation explicite membre exigée avant reprise du paiement.

### Invariants validés

- Le prix présenté au membre reste stable pendant la fenêtre de paiement.
- La règle tarifaire appliquée est traçable (versionnée).

### Verdict

✅ Le modèle est cohérent, avec une recommandation de formaliser un **snapshot tarifaire versionné** côté Booking.

---

## 5.4 Validation transversale de l'architecture

### Cohérence des agrégats

- Les frontières d'agrégats définies en Phase 3 tiennent sur les scénarios critiques.
- Les mises à jour atomiques restent locales à chaque agrégat.
- Les références inter-agrégats (`MembreId`, `EspaceId`) évitent le couplage fort.

### Cohérence des Bounded Contexts

- Les relations **Customer/Supplier** fonctionnent sans inversion de responsabilités.
- Le pattern **Conformist** d'Analytics reste adapté (lecture/projection).
- Les **ACL** protègent bien le modèle interne vis-à-vis des systèmes externes.

### Patterns de consistance

- **Strong consistency** justifiée pour les conflits de réservation.
- **Eventual consistency** pertinente pour analytics et propagations de statut.
- **Saga** adaptée pour les transactions métier longues (réservation/paiement/remboursement).

---

## 5.5 Écarts identifiés et ajustements recommandés

1. **Snapshot tarifaire explicite**
   - Recommandation : introduire un objet `TarificationSnapshot` (montant, devise, version règle, expiration) pour tracer précisément le prix accepté.

2. **Idempotence remboursement/paiement**
   - Recommandation : imposer une clé métier idempotente (`ReservationId + action`) pour éviter les doubles exécutions en retry.

3. **Politique de suspension sur réservations futures**
   - Recommandation : documenter explicitement si les réservations déjà confirmées sont conservées ou automatiquement annulées lors d'une suspension.

4. **SLA de lock distribué**
   - Recommandation : formaliser timeout, retry et monitoring pour prévenir les blocages de créneaux.

---

## Conclusion de la Phase 5

La validation par cas d'usage confirme que le modèle DDD de WorkSpace+ est **globalement solide, cohérent et exploitable** sur les flux métier critiques.

### Résultats clés

- ✅ Invariants métier majeurs validés (double réservation, intégrité financière, contrôle d'accès).
- ✅ Cohérence entre agrégats, services de domaine et Bounded Contexts.
- ✅ Stratégies de consistance adaptées selon la criticité métier.
- ✅ Identification d'améliorations ciblées pour sécuriser l'industrialisation.

Cette phase prépare directement la **Phase 6 (livrable final)**, qui consolidera les décisions architecturales, les arbitrages et les recommandations de mise en œuvre.