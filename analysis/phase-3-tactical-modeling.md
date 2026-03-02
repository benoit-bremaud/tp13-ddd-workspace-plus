# Phase 3 - Modélisation Tactique DDD

## Introduction : Du Vocabulaire à l'Architecture

La Phase 3 marque la transition cruciale entre la **compréhension stratégique** (Phases 1-2) et l'**implémentation concrète** de l'architecture DDD. Elle transforme l'Ubiquitous Language établi en concepts architecturaux précis et opérationnels.

L'objectif n'est plus seulement de **nommer** les concepts métier, mais de déterminer **comment ils collaborent** pour protéger les invariants business et encapsuler la complexité métier différenciante de WorkSpace+.

Cette phase applique les patterns tactiques du DDD : Entités, Value Objects, Agrégats, et Services de Domaine. Chaque décision architectural doit être **justifiée métier** et **alignée** avec le Core Domain identifié.

## 3.1 Entités : Identité et Cycle de Vie Métier

Une entité DDD possède une **identité propre** qui persiste dans le temps, indépendamment des mutations de ses attributs. Elle incarne une **continuité métier** et porte des **comportements** business significatifs.

### Réservation

**Justification d'identité :** Une réservation doit être suivie individuellement tout au long de son cycle de vie, même si ses attributs changent (statut, montant, créneau).

**Identité métier :** `ReservationId` - Identifiant unique généré à la création

**États du cycle de vie :**
- **Provisoire** : Créneau bloqué temporairement (10 minutes max)
- **Confirmée** : Paiement validé, réservation définitive
- **Annulée** : Annulation avec gestion des remboursements selon politique
- **Expirée** : Expiration automatique des réservations non confirmées

**Invariants protégés :**
- Une réservation ne peut être confirmée sans paiement valide
- Le créneau doit respecter les règles d'occupation de l'espace
- Les transitions d'état suivent un workflow métier strict
- Un membre ne peut avoir qu'une seule réservation Provisoire simultanée

**Comportements métier clés :**
```
Entité Réservation {
  confirmer(PaiementDetails) : void
  annuler(MotifAnnulation) : MontantRemboursement
  prolonger(NouvelleDuree) : Réservation
  calculerMontantDu() : Money
  verifierEligibiliteAnnulation() : Boolean
}
```

**Justification cycle de vie :** Le statut Provisoire est essentiel pour gérer les conflits de réservation concurrente et l'expérience utilisateur fluide. Sans cette étape, la gestion des abandons de paiement et des conflits temporels deviendrait incohérente.

### Membre

**Justification d'identité :** Le membre représente l'identité métier durable du client dans le système WorkSpace+, distincte de l'utilisateur technique et du customer commercial.

**Identité métier :** `MembreId` - Identifiant métier stable

**Cycle de vie :**
- **Actif** : Membre en règle avec accès complet aux services
- **Suspendu** : Accès temporairement restreint (problème paiement, etc.)
- **Inactif** : Membre n'ayant pas utilisé les services depuis longtemps
- **Résilié** : Fermeture définitive du compte membre

**Invariants protégés :**
- Un membre suspendu ne peut effectuer de nouvelles réservations
- L'historique des réservations doit être préservé même après résiliation
- Les données RGPD doivent être respectées selon l'état du membre

**Comportements métier clés :**
```
Entité Membre {
  souscrireAbonnement(TypeAbonnement) : Abonnement
  effectuerReservation(Espace, Créneau) : Réservation
  consulterHistorique(Période) : Collection<Réservation>
  calculerStatutLoyauté() : StatutLoyauté
  appliquerSuspension(Motif) : void
}
```

### Abonnement

**Justification d'identité :** L'abonnement n'est pas un simple attribut du membre mais une entité avec son propre cycle de vie business, ses règles de gestion et son historique.

**Identité métier :** `AbonnementId` - Identifiant unique de l'abonnement

**Cycle de vie :**
- **Actif** : Abonnement en cours avec droits et crédits disponibles
- **Suspendu** : Suspension temporaire (non-paiement, demande client)
- **Expiré** : Fin de période sans renouvellement
- **Résilié** : Annulation définitive avant terme

**Invariants protégés :**
- Les crédits ne peuvent être consommés que si l'abonnement est Actif
- La période de validité doit être cohérente avec le type d'abonnement
- Les remises ne s'appliquent que pour les abonnements actifs
- Le renouvellement automatique respecte les préférences du membre

**Comportements métier clés :**
```
Entité Abonnement {
  calculerRemiseApplicable(TypeEspace, Créneaux) : Percentage
  consommerCredits(Montant, TypeConsommation) : void
  renouvelerAutomatiquement() : Abonnement
  calculerProrataMensuel() : Money
  suspendre(Motif, DuréePrévue) : void
}
```

**Impact Core Domain :** L'abonnement est central dans notre Core Domain "Politique Tarifaire". Il porte la logique métier différenciante de calcul des remises et de gestion des crédits selon des règles WorkSpace+ spécifiques.

### Espace

**Justification d'identité :** Chaque espace physique a une identité métier propre avec ses caractéristiques, son historique d'occupation et ses politiques tarifaires spécifiques.

**Identité métier :** `EspaceId` - Identifiant unique de l'espace physique

**Cycle de vie :**
- **Disponible** : Espace opérationnel et réservable
- **Maintenance** : Indisponible temporairement pour maintenance
- **Hors-service** : Indisponible pour durée indéterminée
- **Désactivé** : Retiré définitivement de l'offre

**Invariants protégés :**
- Un espace en maintenance ne peut être réservé
- La capacité maximale ne peut être dépassée
- Les équipements doivent être cohérents avec le type d'espace
- Les politiques tarifaires doivent être définies pour chaque espace

**Comportements métier clés :**
```
Entité Espace {
  verifierDisponibilité(Créneau) : Boolean
  calculerTarif(Durée, Abonnement) : Money
  planifierMaintenance(Période) : void
  ajusterCapacité(NouvelleCapacité) : void
  appliquerPolitiqueLocale(Politique) : void
}
```

### Site

**Justification d'identité :** Le site n'est pas qu'une adresse mais une entité métier avec ses propres règles, horaires et politiques d'exploitation.

**Identité métier :** `SiteId` - Identifiant unique du site WorkSpace+

**Cycle de vie :**
- **Opérationnel** : Site ouvert avec espaces actifs
- **Maintenance** : Fermeture temporaire programmée
- **Fermé** : Fermeture définitive ou exceptionnelle

**Invariants protégés :**
- Les horaires d'ouverture doivent être cohérents
- Au moins un gestionnaire doit être assigné par site
- Les politiques locales ne peuvent contredire les politiques globales
- Les espaces d'un site fermé ne peuvent être réservés

**Comportements métier clés :**
```
Entité Site {
  definirHorairesOuverture(Horaires) : void
  appliquerFermetureExceptionnelle(Période, Motif) : void
  calculerTauxOccupationMoyen(Période) : Percentage
  assignerGestionnaire(Gestionnaire) : void
}
```

## 3.2 Value Objects : Encapsulation et Immutabilité

Les Value Objects encapsulent des **concepts immuables** définis par leurs attributs plutôt que par une identité. Ils protègent des **invariants métier** et simplifient la manipulation de données cohérentes.

### Money (Montant Monétaire)

**Justification Value Object :** Un montant monétaire n'a pas d'identité propre. Deux instances avec les mêmes valeurs sont interchangeables. L'immutabilité garantit l'intégrité des calculs financiers.

**Invariants protégés :**
- Le montant ne peut être négatif (sauf crédit explicite)
- La devise doit être spécifiée et valide
- Les opérations arithmétiques préservent la devise
- La précision décimale respecte les standards monétaires

**Implémentation conceptuelle :**
```
Value Object Money {
  Montant : BigDecimal
  Devise : Currency
  
  add(Money) : Money
  subtract(Money) : Money
  multiply(Factor) : Money
  appliquerRemise(Percentage) : Money
  
  Invariant : Devise identique pour opérations
  Invariant : Précision décimale préservée
}
```

**Risques évités :** En encapsulant la logique monétaire, on évite les erreurs d'arrondis, les incohérences de devise et les calculs financiers incorrects qui pourraient affecter la facturation.

### Créneau (Période Temporelle)

**Justification Value Object :** Un créneau temporel est défini uniquement par ses bornes. Deux créneaux identiques sont interchangeables.

**Invariants protégés :**
- DateFin doit être postérieure à DateDébut
- La durée doit respecter les minimums métier (30 min)
- La durée ne peut excéder les maximums selon le type d'espace
- Les créneaux doivent être alignés sur les horaires d'ouverture

**Implémentation conceptuelle :**
```
Value Object Créneau {
  DateTimeDebut : DateTime
  DateTimeFin : DateTime
  
  calculerDuree() : Duration
  chevaucheAvec(Créneau) : Boolean
  estDansHoraires(HorairesOuverture) : Boolean
  diviserEn(NombreParties) : Collection<Créneau>
  
  Invariant : DateTimeFin > DateTimeDebut
  Invariant : Durée >= DureeMinimale
}
```

### Crédit

**Justification Value Object :** Un crédit individuel est immuable une fois créé. Sa valeur, son type et son expiration sont fixes.

**Invariants protégés :**
- Le montant du crédit ne peut être négatif
- La date d'expiration doit être future à la création
- Le type de crédit doit être cohérent avec le montant
- Un crédit expiré ne peut être consommé

**Implémentation conceptuelle :**
```
Value Object Crédit {
  Valeur : Money
  Type : TypeCrédit (Heures, Euros, Points)
  DateExpiration : Date
  
  estValide(DateCourante) : Boolean
  peutCouvrir(Montant) : Boolean
  calculerReliquat(Consommation) : Crédit
  
  Invariant : DateExpiration > DateCréation
  Invariant : Valeur.Montant >= 0
}
```

### Adresse

**Justification Value Object :** L'adresse d'un site n'a pas d'identité propre. Elle est définie par ses composants géographiques.

**Invariants protégés :**
- Le code postal doit être valide pour le pays
- La cohérence ville/code postal doit être respectée
- L'adresse doit être géocodable pour les services de localisation

**Implémentation conceptuelle :**
```
Value Object Adresse {
  Rue : String
  Ville : String
  CodePostal : String
  Pays : String
  
  formaterPourAffichage() : String
  calculerDistanceVers(Adresse) : Distance
  valider() : Boolean
  
  Invariant : CodePostal valide pour Pays
  Invariant : Tous les champs obligatoires renseignés
}
```

## 3.3 Analyse Critique : Pourquoi un Montant avec un Simple Double est Dangereux

La question posée dans le TP illustre parfaitement pourquoi les Value Objects sont cruciaux en DDD.

### Risques Techniques du Simple Double

1. **Erreurs d'Arrondis Cumulées**
   ```
   Exemple : 0.1 + 0.2 = 0.30000000000000004 en double
   Impact : Sur 10 000 transactions, erreur cumulée significative
   ```

2. **Précision Flottante Inadaptée**
   - Les doubles utilisent une représentation binaire
   - Les décimales monétaires (0.01, 0.10) ne sont pas représentables exactement
   - Les calculs répétés amplifient les imprécisions

3. **Incohérences de Devise Non Contrôlées**
   ```java
   double montantEUR = 100.50;
   double montantUSD = 85.75;
   double total = montantEUR + montantUSD; // Erreur métier silencieuse !
   ```

### Risques Métier Concrets

1. **Erreurs de Facturation**
   - Remboursements incorrects impactant la satisfaction client
   - Calculs de remises d'abonnement erronés
   - Écarts comptables difficiles à identifier

2. **Problèmes Réglementaires**
   - Non-conformité aux standards comptables
   - Audits financiers compromise
   - Problèmes de réconciliation bancaire

3. **Évolutivité Compromise**
   - Impossible d'ajouter facilement de nouvelles devises
   - Logiques de conversion dispersées dans le code
   - Tests unitaires monétaires fragiles

### Architecture Value Object Money

Le Value Object Money résout ces problèmes :

```java
public class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }
    
    public Money applyDiscount(Percentage discount) {
        BigDecimal factor = BigDecimal.ONE.subtract(discount.toDecimal());
        return new Money(amount.multiply(factor), currency);
    }
}
```

**Avantages :**
- **Précision garantie** via BigDecimal
- **Immutabilité** empêchant les mutations accidentelles  
- **Invariants** protégés (devise, précision)
- **Expressivité** métier dans le code
- **Évolutivité** pour nouvelles règles monétaires

## 3.4 Services de Domaine : Logique Métier Transversale

Les services de domaine encapsulent la **logique métier** qui ne relève naturellement d'aucune entité spécifique, tout en restant dans le domaine métier (pas applicatif).

### PricingService (Service Central du Core Domain)

**Justification :** Le calcul de prix croise plusieurs entités (Espace, Abonnement, Membre) et porte la logique différenciante de WorkSpace+. Cette complexité mérite un service dédié.

**Responsabilités métier :**
- Calcul du prix de base selon type d'espace et durée
- Application des remises d'abonnement
- Gestion des heures de pointe et tarifications spéciales
- Application des politiques promotionnelles

**Interface métier :**
```
Service PricingService {
  calculerPrixReservation(
    Espace : Espace,
    Créneau : Créneau,
    Membre : Membre
  ) : Money
  
  calculerRemiseApplicable(
    Abonnement : Abonnement,
    TypeReservation : TypeReservation
  ) : Percentage
  
  appliquerPromotionSiEligible(
    Membre : Membre,
    MontantBase : Money
  ) : Money
}
```

**Règles métier encapsulées :**
- Heures de pointe : +20% entre 9h-17h en semaine
- Remise Premium : -30% sur tous les espaces
- Remise Standard : -15% sur espaces partagés uniquement
- Tarif dégressif : -10% au-delà de 4h de réservation

### AvailabilityCalculator (Service de Cohérence)

**Justification :** Le calcul de disponibilité nécessite de croiser les données de plusieurs agrégats (Espaces, Réservations, Site) tout en garantissant la cohérence temps réel.

**Responsabilités métier :**
- Calcul de disponibilité dynamique pour un espace donné
- Prise en compte des horaires d'ouverture du site
- Gestion des périodes de maintenance programmée
- Optimisation par cache avec invalidation intelligente

**Interface métier :**
```
Service AvailabilityCalculator {
  estDisponible(
    Espace : Espace,
    Créneau : Créneau
  ) : Boolean
  
  obtenirCreneauxLibres(
    Espace : Espace,
    Période : Période
  ) : Collection<Créneau>
  
  calculerTauxOccupation(
    Espace : Espace,
    Période : Période
  ) : Percentage
}
```

### RefundCalculationService (Service de Politique)

**Justification :** Les règles de remboursement sont complexes et évolutives selon les politiques commerciales. Elles méritent un service spécialisé.

**Responsabilités métier :**
- Calcul du montant de remboursement selon délai d'annulation
- Application des politiques différentielles selon type d'abonnement
- Gestion des frais d'annulation
- Prise en compte des conditions spéciales (force majeure, etc.)

**Interface métier :**
```
Service RefundCalculationService {
  calculerMontantRemboursement(
    Réservation : Réservation,
    DateAnnulation : DateTime,
    MotifAnnulation : MotifAnnulation
  ) : Money
  
  determinerEligibilitéRemboursement(
    Réservation : Réservation,
    DateAnnulation : DateTime
  ) : Boolean
}
```

## Conclusion Intermédiaire

Cette analyse des Entités, Value Objects et Services de Domaine établit la **structure tactique** de l'architecture DDD WorkSpace+. 

La section suivante (3.4) abordera les **Agrégats** : comment organiser ces concepts en frontières de cohérence pour protéger les invariants métier critiques.

Chaque décision architecturale prise ici **s'aligne avec le Core Domain** identifié et **respecte l'Ubiquitous Language** établi, garantissant une implémentation fidèle à la réalité métier de WorkSpace+.