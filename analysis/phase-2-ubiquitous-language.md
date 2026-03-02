# Phase 2 - Construction de l'Ubiquitous Language

## Introduction Méthodologique

La construction du langage ubiquitaire (Ubiquitous Language) constitue l'une des pratiques les plus fondamentales du Domain-Driven Design. Ce n'est pas un simple exercice de documentation, mais la création d'un **langage vivant** qui unifie la compréhension entre experts métier et développeurs.

L'objectif est d'éliminer toute ambiguïté conceptuelle qui pourrait corrompre la fidélité du modèle métier dans le code. Chaque terme utilisé en réunion doit correspondre exactement au vocabulaire utilisé dans l'implémentation.

## 2.1 Clarification des Concepts Ambigus

### Méthode d'Analyse

Pour chaque concept métier critique, j'applique une grille d'analyse systématique :
1. **Définition intuitive** : Ce que le terme évoque spontanément
2. **Ambiguïtés identifiées** : Sources potentielles de confusion
3. **Impact architectural** : Conséquences sur la modélisation
4. **Décision retenue** : Définition précise et justifiée

### Tableau de Clarification des Concepts

| Terme | Définition Précise | Problème Potentiel | Décision Retenue |
|-------|-------------------|-------------------|------------------|
| **Réservation** | Engagement d'un membre sur un espace donné pour une période déterminée, avec cycle de vie évolutif | Ambiguïté : inclut-elle les réservations non payées ? Est-elle valide avant confirmation bancaire ? Existe-t-elle pendant la sélection ? | Une **Réservation** existe dès la validation du créneau par l'utilisateur, mais évolue selon un statut explicite : `Provisoire` (10min), `Confirmée` (payée), `Annulée`, `Expirée`. La confirmation bancaire fait transiter de Provisoire à Confirmée. |
| **Disponibilité** | État dérivé indiquant qu'un espace peut être réservé sur une période donnée | Confusion entre donnée stockée et donnée calculée dynamiquement. Problème de cohérence temps réel si stockée, problème de performance si calculée. | La **Disponibilité** est un état **calculé dynamiquement** à partir des réservations existantes, jamais stocké. Cela garantit la cohérence temps réel et évite les problèmes de synchronisation. Un service de domaine `AvailabilityCalculator` en est responsable. |
| **Abonnement** | Contrat donnant des droits d'usage privilégiés sur la plateforme selon un modèle de facturation récurrent | Incertitude : est-ce une simple propriété du membre ou une entité à part entière ? Gestion des crédits complexe. | Un **Abonnement** est une **entité** avec identité propre, cycle de vie (Actif, Suspendu, Expiré, Résilié) et comportements spécifiques (calcul crédits, application remises). Il possède une relation d'agrégation avec les Crédits. |
| **Membre** | Utilisateur authentifié ayant souscrit à la plateforme WorkSpace+ avec un profil métier complet | Distinction floue avec Client/Utilisateur. Risque de mélanger authentification et métier. | Un **Membre** représente l'entité métier centrale post-authentification. Distinct de User (technique) et de Customer (commercial). Il porte l'historique, les préférences, et l'état d'abonnement. |
| **Espace** | Ressource physique réservable avec caractéristiques et équipements spécifiques dans un site WorkSpace+ | Confusion entre l'espace physique et sa représentation système. Gestion des équipements variable. | Un **Espace** est l'entité métier représentant la ressource réservable. Il encapsule : type (bureau privé, salle réunion, espace ouvert), capacité, équipements, site, et politiques tarifaires spécifiques. |
| **Créneau** | Période temporelle définie avec début et fin, associée à un espace pour matérialiser une réservation | Ambiguïté entre concept temporel abstrait et réservation effective. Gestion des chevauchements. | Un **Créneau** est un Value Object temporel défini par `[DateTimeDebut, DateTimeFin]` avec invariants (fin > début, durée minimale). Il appartient au langage partagé mais ne constitue pas une entité. |
| **Site** | Localisation physique où sont hébergés les espaces WorkSpace+, avec gestion administrative propre | Confusion entre adresse géographique et entité métier. Gestion des horaires d'ouverture. | Un **Site** est une entité métier regroupant des Espaces avec : adresse, horaires d'ouverture, gestionnaire responsable, politiques locales. Il influence les règles tarifaires et de réservation. |
| **Crédit** | Unité de valeur accordée par un abonnement et consommée lors des réservations selon des règles métier spécifiques | Confusion entre monnaie virtuelle et temps d'usage. Gestion de l'expiration et du report variable. | Un **Crédit** est un Value Object avec montant, type (`Heures`, `Euros`, `Points`), date d'expiration et règles de consommation. Les crédits sont gérés en portefeuille par l'Abonnement avec politique FIFO d'expiration. |

### Analyse Critique des Décisions

#### Réservation : Pourquoi une entité avec cycle de vie ?

**Justification métier :** Dans le contexte WorkSpace+, une réservation n'est pas un événement instantané mais un processus. L'utilisateur sélectionne un créneau, le système le "bloque" temporairement (Provisoire), puis la confirmation de paiement la valide définitivement.

**Impact architectural :** Cette approche permet de gérer proprement :
- Les abandons de panier (expiration automatique)
- Les conflits de réservation concurrente  
- Les politiques d'annulation différentielles selon le statut
- La traçabilité des opérations métier

**Alternative rejetée :** Considérer qu'une réservation n'existe qu'après paiement créerait des incohérences (comment gérer les créneaux "en cours de réservation" ?) et compliquerait la gestion des conflits.

#### Disponibilité : Pourquoi calculée et non stockée ?

**Justification technique :** Stocker la disponibilité créerait un problème de cohérence distribué classique. Chaque création/modification/annulation de réservation nécessiterait de mettre à jour de multiples enregistrements de disponibilité.

**Justification métier :** La disponibilité n'est pas une donnée métier intrinsèque mais un état dérivé. Elle dépend des réservations existantes, des horaires d'ouverture du site, des maintenances planifiées.

**Performance :** Un service `AvailabilityCalculator` avec cache intelligent (invalidé à chaque mutation de réservation) offre de meilleures performances qu'une synchronisation permanente.

#### Abonnement : Entité plutôt qu'attribut ?

**Justification métier :** L'abonnement dans WorkSpace+ n'est pas qu'un "flag premium". Il possède :
- Un cycle de vie propre (souscription, modification, suspension, résiliation)
- Des règles métier complexes (calcul crédits, remises, renouvellements)
- Un historique des changements à conserver

**Impact sur le Core Domain :** L'abonnement influence directement les politiques tarifaires (notre Core Domain identifié). Il doit donc être modélisé comme une entité riche avec comportements métier.

## 2.2 Analyse Critique : Questions Fondamentales

### Une réservation existe-t-elle avant paiement ?

**Réponse argumentée :** **Oui**, mais avec un statut `Provisoire` et une durée de vie limitée.

**Arguments métier :**
1. **Expérience utilisateur** : L'utilisateur doit voir son créneau "réservé" pendant qu'il procède au paiement
2. **Gestion des conflits** : Empêcher qu'un autre utilisateur réserve le même créneau simultanément
3. **Conversion commerciale** : Permettre le suivi des abandons de réservation pour optimisation

**Arguments techniques :**
1. **Cohérence** : Éviter les conditions de course dans les réservations concurrentes
2. **Audit** : Tracer toutes les tentatives de réservation pour analyse métier
3. **Évolutivité** : Faciliter l'ajout de fonctionnalités (rappels d'abandon, offres de récupération)

**Règles métier associées :**
- Durée de vie : 10 minutes maximum en statut Provisoire
- Nettoyage automatique : Processus batch d'expiration des réservations abandonnées
- Unicité : Un membre ne peut avoir qu'une seule réservation Provisoire simultanée

### Une disponibilité est-elle calculée ou stockée ?

**Réponse argumentée :** **Calculée** dynamiquement avec cache intelligent.

**Justification par l'absurde :** Si on stockait la disponibilité :
1. Chaque réservation nécessiterait la mise à jour de N enregistrements de disponibilité
2. Les annulations créeraient des problèmes de cohérence temporelle complexes
3. L'ajout de nouvelles règles (maintenance, fermetures exceptionnelles) nécessiterait des migrations de données massives

**Architecture retenue :**
```
Service AvailabilityCalculator {
  calculateAvailability(espace, periode) : Boolean
  - Vérifie horaires site
  - Contrôle réservations existantes  
  - Applique règles maintenance
  - Cache avec TTL court + invalidation événementielle
}
```

**Avantages :**
- Source de vérité unique (les réservations)
- Flexibilité maximale pour évolution règles métier
- Cohérence garantie par conception
- Performance acceptable avec cache approprié

### Un abonnement est-il une entité ou un simple attribut ?

**Réponse argumentée :** **Entité** avec identité et cycle de vie propres.

**Analyse comportementale :**
Un abonnement dans WorkSpace+ :
- **Évolue dans le temps** : souscription → activation → modifications → résiliation
- **Possède des règles métier complexes** : calcul de crédits, application de remises, gestion des renouvellements
- **Interagit avec d'autres entités** : influence les réservations, génère des factures
- **Nécessite un historique** : changements de formule, suspensions, réactivations

**Impact sur le Core Domain :**
L'abonnement est central dans notre Core Domain "Politique Tarifaire & Optimisation". Il ne peut être réduit à un simple attribut car il porte une logique métier différenciante complexe.

**Modélisation retenue :**
```
Entité Abonnement {
  - Identité : AbonnementId
  - État : Actif | Suspendu | Expiré | Résilié
  - Formule : Premium | Standard | Entreprise
  - Période : DateDebut, DateFin
  - Crédits : Collection<Crédit>
  
  Comportements :
  - calculerRemiseApplicable(typeEspace, duree)
  - consommerCredits(montant, typeReservation)
  - renouvelerAutomatiquement()
}
```

## Conclusion de l'Analyse Terminologique

Cette clarification terminologique établit les **fondations sémantiques** indispensables à la modélisation tactique qui suivra. Chaque décision prise ici :

1. **Élimine les ambiguïtés** qui pourraient corrompre l'implémentation
2. **Anticipe les évolutions** métier futures de WorkSpace+
3. **S'aligne sur le Core Domain** identifié en Phase 1
4. **Facilite la communication** entre équipes métier et technique

Ces définitions constituent désormais le **référentiel sémantique** du projet. Elles seront consolidées dans un glossaire métier central et doivent être respectées scrupuleusement dans toutes les phases suivantes.

L'effort investi dans cette précision terminologique conditionnera directement la qualité et la maintenabilité de l'architecture DDD de WorkSpace+.