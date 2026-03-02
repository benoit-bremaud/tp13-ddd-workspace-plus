# Guidelines de Workflow - TP13 DDD WorkSpace+

## Organisation du Travail

### Principe Fondamental
**Une phase = Une branche = Un livrable = Une PR**

### Structure de Travail par Phase

#### Phase 0 ✅ - Setup Projet
- **Branche** : `main` (initialisation)
- **Livrables** : Structure projet, configuration Git/GitHub
- **Statut** : ✅ Terminée

#### Phase 1 ✅ - Analyse Stratégique  
- **Branche** : `feat/phase-1-core-domain-analysis`
- **Livrables** : 
  - `analysis/phase-1-core-domain.md` (analyse argumentée)
  - `diagrams/domain-overview.mmd` (diagramme stratégique)
- **PR** : #1 - Mergée
- **Statut** : ✅ Terminée

#### Phase 2 - Ubiquitous Language
- **Branche** : `feat/phase-2-ubiquitous-language`  
- **Livrables** :
  - `analysis/phase-2-ubiquitous-language.md`
  - `docs/glossary.md` (dictionnaire des termes)
- **PR** : #2 - Mergée
- **Statut** : ✅ Terminée

#### Phase 3 - Modélisation Tactique
- **Branche** : `feat/phase-3-tactical-modeling`
- **Livrables** :
  - `analysis/phase-3-tactical-modeling.md`
  - `diagrams/entities-value-objects.mmd`
  - `diagrams/aggregates.mmd`
  - `diagrams/domain-services.mmd`
- **PR** : #3 - Mergée
- **Statut** : ✅ Terminée

#### Phase 4 - Bounded Contexts
- **Branche** : `feat/phase-4-bounded-contexts`
- **Livrables** :
  - `analysis/phase-4-bounded-contexts.md`
  - `diagrams/context-map.mmd`
- **PR** : #4 - Mergée
- **Statut** : ✅ Terminée

#### Phase 5 - Validation Cas d'Usage
- **Branche** : `feat/phase-5-use-cases-validation`
- **Livrables** :
  - `analysis/phase-5-use-cases.md`
  - `diagrams/use-cases-flow.mmd`
- **PR** : #5 - Mergée
- **Statut** : ✅ Terminée

#### Phase 6 - Livrable Final
- **Branche** : `feat/phase-6-final-deliverable`
- **Livrables** :
  - `analysis/phase-6-synthesis.md`
  - `docs/architecture-document.md` (document final)
- **PR** : #6 - En préparation
- **Statut** : 🔄 En cours

## Convention de Commits

### Format Standard (Angular)
```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Types Utilisés
- `feat` : Nouvelle fonctionnalité/analyse
- `docs` : Documentation uniquement  
- `fix` : Correction d'erreur
- `refactor` : Refactorisation sans changement fonctionnel
- `chore` : Tâches de maintenance

### Scopes par Phase
- `setup` : Configuration initiale
- `analysis` : Analyses métier et strategiques
- `diagrams` : Schémas et visualisations
- `docs` : Documentation générale

### Exemples de Messages
```bash
feat(analysis): identify core domain and strategic subdomains
feat(diagrams): add domain overview with investment priorities  
docs(workflow): establish phase-by-phase organization guidelines
chore(setup): initialize project structure and GitHub integration
```

## Workflow de Validation

### Processus par Phase
1. **Développement** sur branche dédiée
2. **Commits fréquents** avec messages explicites
3. **Push réguliers** pour sauvegarde 
4. **PR détaillée** à la fin de phase
5. **Validation manuelle** par le responsable
6. **Merge** une fois approuvée
7. **Création nouvelle branche** pour phase suivante

### Template PR Obligatoire
- Objectif clair de la phase
- Liste des changements
- Type de modification
- Checklist de validation
- Description détaillée
- Tests effectués

## Règles d'Organisation

### 📁 Structure des Fichiers
```
├── analysis/           # Analyses par phase
├── diagrams/          # Schémas Mermaid
├── docs/              # Documentation projet
├── .github/           # Templates PR/Issues
└── README.md          # Vue d'ensemble
```

### 🔄 Synchronisation
- **Push fréquent** pendant développement
- **Branches à jour** avant création PR
- **Main protégée** - Merge via PR uniquement

### 📋 Traçabilité
- Chaque décision DDD **justifiée et documentée**
- Liens entre phases **explicites**  
- Evolution des concepts **tracée**

### 🎯 Focus Qualité
- **Argumentation rigoureuse** pour chaque concept DDD
- **Cohérence** entre analyse et diagrammes
- **Validation métier** avant passage phase suivante

## Points d'Attention

### ⚠️ Erreurs à Éviter
- Mélanger plusieurs phases dans une PR
- Commits sans message explicite
- Documentation insuffisante des choix DDD
- Passage phase suivante sans validation précédente

### ✅ Bonnes Pratiques
- Une seule responsabilité par commit
- Messages de commit en français (contexte académique)
- Diagrammes synchronisés avec analyses textuelles
- Validation systématique avant progression

---

**Cette organisation garantit la qualité, la traçabilité et la progression maîtrisée du projet DDD WorkSpace+.**