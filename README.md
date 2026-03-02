# TP13 - Application des Concepts du Domain-Driven Design (DDD)

## Contexte

Ce projet présente l'application des concepts du Domain-Driven Design (DDD) pour la conception de l'architecture métier d'une plateforme SaaS de gestion d'espaces de coworking : **WorkSpace+**.

## Objectifs

L'entreprise WorkSpace+ souhaite développer une plateforme capable de :
- Gérer les espaces et leurs caractéristiques
- Permettre les réservations horaires et journalières
- Gérer des abonnements mensuels
- Appliquer des politiques tarifaires dynamiques
- Gérer paiements et remboursements
- Éviter les conflits de réservation
- Permettre à terme l'ouverture à des partenaires franchisés

## Approche

Ce travail adopte une posture d'architecte logiciel, en privilégiant la compréhension métier avant la modélisation technique.

**Citation du CTO** : *"Nous voulons un système qui reflète notre métier, pas un simple CRUD autour d'une base SQL."*

## Structure du Projet

```
├── docs/           # Documentation d'architecture
├── diagrams/       # Schémas et diagrammes
├── analysis/       # Analyses détaillées par phase
└── README.md       # Ce fichier
```

### Livrables principaux

- `analysis/phase-1-core-domain.md` à `analysis/phase-6-synthesis.md`
- `docs/architecture-document.md` (synthèse finale professionnelle)
- Diagrammes Mermaid :
  - `diagrams/domain-overview.mmd`
  - `diagrams/entities-value-objects.mmd`
  - `diagrams/aggregates.mmd`
  - `diagrams/context-map.mmd`
  - `diagrams/use-cases-flow.mmd`

## Phases de Travail

1. **Analyse Stratégique** - Identification du Core Domain
2. **Ubiquitous Language** - Construction du langage métier
3. **Modélisation Tactique** - Entités, Value Objects, Agrégats, Services
4. **Bounded Contexts** - Architecture distribuée
5. **Validation** - Tests par cas d'usage
6. **Synthèse** - Livrable final

## Workflow Git

- Une branche par phase de travail
- Commits fréquents avec convention Angular
- Pull requests détaillées pour chaque phase
- Validation par review avant merge

## État d'avancement

- ✅ Phases 1 à 6 réalisées
- ✅ Validation par cas d'usage critiques effectuée
- ✅ Document d'architecture final livré

---

**Auteur** : Benoit BREMAUD  
**Cours** : Architecture Logicielle - M1 Ynov  
**Date** : Mars 2026