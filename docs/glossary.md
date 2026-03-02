# Glossaire Métier - WorkSpace+ DDD

## Référentiel Terminologique Central

Ce glossaire constitue la source unique de vérité pour tous les concepts métier de la plateforme WorkSpace+. Il matérialise l'**Ubiquitous Language** établi en Phase 2 et doit être respecté par toutes les équipes.

> **Principe fondamental :** Chaque terme utilisé en réunion métier doit correspondre exactement au vocabulaire utilisé dans le code.

---

## 📋 Concepts Métier Fondamentaux

### Abonnement
**Définition :** Contrat donnant des droits d'usage privilégiés sur la plateforme selon un modèle de facturation récurrent.

**Type DDD :** Entité avec identité et cycle de vie propres

**États possibles :** `Actif`, `Suspendu`, `Expiré`, `Résilié`

**Comportements clés :**
- Calcul des remises applicables selon type d'espace et durée
- Consommation automatique des crédits selon règles métier
- Renouvellement automatique avec gestion des échéances

**Relations :** Appartient à un Membre, contient une collection de Crédits

---

### Crédit
**Définition :** Unité de valeur accordée par un abonnement et consommée lors des réservations selon des règles métier spécifiques.

**Type DDD :** Value Object immuable

**Types possibles :** `Heures`, `Jours`, `Euros`

**Propriétés :** Montant, Type, Date d'expiration, Règles de consommation

**Gestion :** Portefeuille géré par l'Abonnement avec politique FIFO d'expiration

---

### Créneau
**Définition :** Période temporelle définie par un début et une fin, utilisée pour délimiter les réservations.

**Type DDD :** Value Object temporel

**Format :** `[DateTimeDebut, DateTimeFin]`

**Invariants :** 
- DateTimeFin > DateTimeDebut
- Durée minimale respectée selon règles métier
- Cohérence avec horaires d'ouverture du site

---

### Disponibilité
**Définition :** État dérivé indiquant qu'un espace peut être réservé sur une période donnée.

**Type DDD :** État calculé (jamais stocké)

**Calcul :** Dynamique via service de domaine `AvailabilityCalculator`

**Facteurs :** Réservations existantes, horaires site, maintenances planifiées

**Performance :** Cache intelligent avec invalidation événementielle

---

### Espace
**Définition :** Ressource physique réservable avec caractéristiques et équipements spécifiques dans un site WorkSpace+.

**Type DDD :** Entité métier centrale

**Types :** `Bureau privé`, `Salle de réunion`, `Espace ouvert`, `Cabine téléphonique`

**Propriétés :** Capacité, Équipements, Site d'appartenance, Politiques tarifaires

**Comportements :** Calcul de disponibilité, Application de tarifs spécifiques

---

### Membre
**Définition :** Utilisateur authentifié ayant souscrit à la plateforme WorkSpace+ avec un profil métier complet.

**Type DDD :** Entité métier centrale

**Distinction :** 
- ≠ `User` (couche technique d'authentification)
- ≠ `Customer` (entité commerciale/facturation)

**Propriétés :** Historique réservations, Préférences, État abonnement, Profil métier

**Relations :** Peut posséder un Abonnement, effectue des Réservations

---

### Réservation
**Définition :** Engagement d'un membre sur un espace donné pour une période déterminée, avec cycle de vie évolutif.

**Type DDD :** Entité avec cycle de vie complexe

**États du cycle de vie :**
- `Provisoire` : Créneau bloqué temporairement (10 min max)
- `Confirmée` : Réservation payée et définitive
- `Annulée` : Annulation avec politique de remboursement
- `Expirée` : Expiration automatique non payée

**Transitions :**
- Provisoire → Confirmée : Via confirmation paiement
- Provisoire → Expirée : Via timeout automatique
- Confirmée → Annulée : Via demande d'annulation

**Règles métier :**
- Un membre ne peut avoir qu'une réservation Provisoire simultanée
- Nettoyage automatique des réservations expirées
- Politiques d'annulation différentielles selon statut et délai

---

### Site
**Définition :** Localisation physique où sont hébergés les espaces WorkSpace+, avec gestion administrative propre.

**Type DDD :** Entité métier d'agrégation

**Propriétés :** Adresse, Horaires d'ouverture, Gestionnaire responsable, Politiques locales

**Comportements :** Influence sur règles tarifaires et de réservation

**Relations :** Contient une collection d'Espaces

---

## 🔧 Services de Domaine

### AvailabilityCalculator
**Responsabilité :** Calcul dynamique de la disponibilité des espaces

**Méthode principale :** `calculateAvailability(espace: Espace, periode: Créneau): Boolean`

**Logique :**
1. Vérification horaires d'ouverture du site
2. Contrôle des réservations existantes sur la période
3. Application des règles de maintenance
4. Mise en cache avec invalidation intelligente

---

## 📊 Règles Métier Transversales

### Politique de Réservation
- **Anticipation maximale :** 30 jours pour membres Standard, 60 jours pour Premium
- **Durée minimale :** 30 minutes
- **Durée maximale :** 8 heures pour bureaux privés, 4 heures pour salles de réunion
- **Annulation gratuite :** Jusqu'à 2 heures avant pour Premium, 4 heures pour Standard

### Politique Tarifaire (Core Domain)
- **Tarification dynamique :** Selon type d'espace, durée, statut d'abonnement
- **Heures de pointe :** Supplément de 20% en journée (9h-17h)
- **Remises abonnés :** 30% Premium, 15% Standard
- **Crédits d'abonnement :** Expiration après 6 mois, politique FIFO

---

## 🔄 Évolutions du Vocabulaire

### Historique des Changements
| Date | Concept | Modification | Justification |
|------|---------|-------------|---------------|
| 02/03/2026 | Réservation | Ajout statut `Provisoire` | Gestion conflits concurrents |
| 02/03/2026 | Disponibilité | Passage de stockée à calculée | Cohérence temps réel |

### Processus de Mise à Jour
1. **Proposition** par équipe métier ou technique
2. **Analyse d'impact** sur modèle existant
3. **Validation** par Domain Expert
4. **Mise à jour** documentation et code
5. **Communication** à toutes les équipes

---

## 🎯 Notes d'Utilisation

### Pour les Développeurs
- Utiliser exactement ces termes dans les noms de classes, méthodes et variables
- Respecter les invariants définis pour chaque Value Object
- Implémenter les comportements métier dans les entités concernées

### Pour les Équipes Métier
- Utiliser cette terminologie dans toutes les spécifications
- Signaler immédiatement tout écart ou ambiguïté découverte
- Valider que les évolutions respectent l'Ubiquitous Language

### Pour la Documentation
- Référencer ce glossaire dans toute documentation technique
- Maintenir la cohérence terminologique dans les API et interfaces
- Mettre à jour ce document avant toute modification conceptuelle majeure

---

*Ce glossaire est un document vivant, maintenu par l'équipe DDD WorkSpace+ et validé par les Domain Experts.*