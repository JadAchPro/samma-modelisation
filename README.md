# Samma, Modélisation membership

Dashboard interactif (une seule page HTML, aucune dépendance) qui modélise l'économie mensuelle du centre Samma : clients qui paient à la séance vs membres annuels. Objectif : trouver le bon mix et le bon pricing pour la pérennité du projet.

- **Fichier unique** : `index.html` (HTML + CSS + JS vanilla, autonome, fonctionne hors ligne)
- **Langue du dashboard** : anglais (consultable par Lucas et les fondatrices)
- **Devise** : CHF
- **Mot de passe d'accès** : `harmonik` (minuscules)

## Contexte et question de fond

Les fondatrices de Samma envisagent deux façons de payer :

1. **À l'unité** : le client paie sa séance (prix de référence relevé sur le site officiel : ~CHF 160 la séance praticien).
2. **Membership annuel** (~CHF 2'000 à 4'000/an) : donne accès à N séances par mois. Les séances mensuelles non utilisées sont **perdues** (pas de report).

Le prix effectif par séance d'un membre est plus bas que le prix à l'unité (c'est l'intérêt du membership pour le client). Mais chaque séance réservée à un membre est un créneau retiré aux clients ponctuels, qui rapportent plus par séance. Le dashboard sert à visualiser cet arbitrage et à trouver l'équilibre.

## Le modèle

Photo mensuelle en régime stable (pas de rampe de montée en charge, pas d'attrition, choix validés par Jad). Séance "moyenne" unique (pas de distinction par pratique).

### Paramètres (tous des curseurs)

Regroupés en quatre sections : Costs, Capacity, Membership, Single session.

| Section | Paramètre | Défaut | Plage |
|---|---|---|---|
| Costs | Coûts fixes mensuels | CHF 25'000 | 5'000 à 80'000 |
| Costs | Coût variable par séance | CHF 20 | 0 à 120 |
| Capacity | Nombre de praticiens | 4 | 1 à 10 |
| Capacity | Séances par jour par praticien | 6 | 1 à 12 |
| Membership | Prix membership annuel | CHF 3'000 | 1'000 à 6'000 |
| Membership | Séances incluses par mois | 4 | 1 à 10 |
| Membership | Taux d'utilisation des membres | 75% | 0 à 100% |
| Membership | Nombre de membres | 50 | 0 à 300 |
| Single session | Prix séance à l'unité | CHF 160 | 60 à 250 |
| Single session | Taux de remplissage des single sessions disponibles | 60% | 0 à 100% |

« Part de membres » n'est plus un curseur : c'est un **indicateur calculé** (séances membres utilisées / capacité), affiché dans la section Membership et dans le tableau détail.

Constante : **22 jours travaillés par mois** par praticien (`WORKING_DAYS` dans le JS).

### Formules (fonction `compute()` dans index.html)

```
capacité              = praticiens × séances_par_jour × 22

# Le nombre de membres est un input (curseur), plafonné par la capacité :
# les memberships que le planning ne peut pas honorer sont refusés et ne
# génèrent aucun revenu.
membres_max           = capacité / (séances_incluses × taux_utilisation)
membres_servis        = min(membres_demandés, membres_max)
séances_membres       = membres_servis × séances_incluses × taux_utilisation
part_membres          = séances_membres / capacité              # indicateur calculé

# Les single sessions se remplissent sur la capacité restante.
single_disponibles    = capacité - séances_membres
single_remplies       = taux_remplissage × single_disponibles

revenu_membres        = membres_servis × prix_annuel / 12       # indépendant de l'utilisation
revenu_single         = single_remplies × prix_unité
coûts_variables       = (séances_membres + single_remplies) × coût_variable
bénéfice mensuel      = revenus - coûts_fixes - coûts_variables
```

Hypothèses structurantes : **les membres sont prioritaires**, leurs séances utilisées sont bookées avant toute single session, et **la base de membres est plafonnée par la capacité** : au-delà de `membres_max`, les membres supplémentaires sont refusés et ne comptent ni dans les volumes ni dans les revenus (pas de revenu sur un service qu'on ne peut pas rendre). Invariants du modèle (validés par Jad le 11/08/2026) : bouger « taux d'utilisation » ou « séances incluses » change les **séances membres utilisées** et les single sessions disponibles, jamais le nombre de membres servis (hors plafonnement) ni le revenu membership (les membres paient le même prix qu'ils utilisent 0 ou 100% de leurs séances). Une alerte se déclenche si les membres saturent la capacité (peu ou zéro créneaux single), une autre si le curseur membres dépasse ce que la capacité peut honorer (membres refusés).

Point mort : le profit est linéaire en taux de remplissage single. Cas particulier géré : si le prix single est inférieur au coût variable, le profit décroît avec le remplissage ; le KPI affiche alors un **plafond** de remplissage ("X% fill max") au lieu d'un minimum, avec les couleurs cohérentes. Cas "any fill" (rentable partout) et "unreachable" (rentable nulle part) également gérés.

### Indicateurs affichés

- **Bénéfice mensuel** (+ marge en %)
- **Point mort** : taux de remplissage des single sessions nécessaire pour être à l'équilibre, aux réglages courants (balayage numérique 0 à 100%)
- **Taux d'occupation** de la capacité
- **Revenu mensuel** (ventilé membres / single)
- **Encadré insight** : prix effectif par séance membre (mensualité ÷ séances réellement utilisées) comparé au prix à l'unité, et marge par créneau membre vs single
- **Graphique 1** : bénéfice en fonction du nombre de membres, point courant marqué. Répond à "quel est le meilleur mix".
- **Graphique 2** : bénéfice en fonction du taux de remplissage des single sessions (0 à 100%), ligne verticale au point mort.
- **Tableau détail** : tous les volumes et flux du mois.

### Le levier caché : le "breakage"

Le taux d'utilisation des membres est un levier de rentabilité majeur : un membre qui paie CHF 250/mois et n'utilise que 3 séances sur 4 revient à CHF 83 la séance servie au lieu de CHF 62. Les séances perdues (non reportables) sont du revenu sans coût variable ni créneau consommé. C'est pour cela que ce taux a son propre curseur.

## Mot de passe

Barrière côté client uniquement (comparaison en base64 dans le JS, stockage `sessionStorage`). C'est un **filtre de confidentialité léger**, pas une vraie sécurité : quelqu'un qui lit le code source peut le contourner. Suffisant pour un outil interne non référencé (`noindex` posé). Pour changer le mot de passe : remplacer la constante `EXPECTED` dans `index.html` par `btoa("nouveau_mot_de_passe")`.

## Déploiement GitHub Pages (à faire par la session Claude qui a l'accès GitHub)

Le VPS de cette session n'a ni `gh` ni clé SSH GitHub. Marche à suivre :

1. Créer un repo GitHub nommé **`samma-modelisation`** (public, GitHub Pages gratuit ne fonctionne pas sur un repo privé en plan Free).
2. Y pousser le contenu de ce dossier (`index.html` + `README.md`) sur la branche `main`.
3. Activer GitHub Pages : Settings → Pages → Source : "Deploy from a branch" → branche `main`, dossier `/ (root)`.
4. L'URL sera `https://<compte>.github.io/samma-modelisation/`.
5. Transmettre l'URL et le mot de passe (`harmonik`) à Jad, qui les partagera avec Lucas et les fondatrices.

Aucun build, aucune dépendance, aucun secret dans le repo : le fichier est servable tel quel.

## Historique et reprise du travail

- **2026-08-11** : création du modèle par Claude Code (session VPS, dossier source : `/home/samma/jad_workspace/samma/business/samma-modelisation/`). Paramètres et hypothèses validés avec Jad (séance moyenne unique, membership annuel seulement, séances mensuelles perdues si non utilisées, pas de rampe ni d'attrition, KPI = bénéfice mensuel + point mort).
- **2026-08-11** : refonte des sections de curseurs (demande de Jad). Split de « Pricing & membership » en deux sections « Membership » et « Single session ». Suppression de la section « Demand » (curseurs « Clients per month » et « Sessions per pay-per-use client » retirés). « Share of members » déplacé dans Membership et redéfini comme part de la capacité. Nouveau curseur « Available single session fill rate ». Point mort et graphique 2 recalés sur le taux de remplissage. Déployé sur GitHub Pages.
- **2026-08-11 (correctif cohérence)** : la variante « share = % capacité en input » créait une incohérence relevée par Jad : bouger utilisation ou séances incluses faisait varier le nombre de membres et le revenu membership au lieu des séances utilisées. Correctif : « Number of members » devient le curseur (input), « Share of members » devient un indicateur calculé (séances membres / capacité). Invariants garantis : utilisation et séances incluses ne modifient que les séances membres utilisées et les single sessions disponibles. Graphique 1 = bénéfice vs nombre de membres. Alerte « promesse intenable » rétablie. Redéployé.
- **2026-08-15 (audit pré-partage + correctifs, go de Jad)** : audit de cohérence (session principale + sous-agent indépendant), 10 findings, tous corrigés. (1) Base de membres plafonnée par la capacité : les membres au-delà de `membres_max` sont refusés et ne génèrent plus de revenu (supprime la courbe en V trompeuse du graphique 1). (2) Point mort géré dans le cas prix single < coût variable (profit décroissant : affichage "fill max", plus de KPI vert avec profit rouge). (3-5) Encadré insight : cas utilisation 0% (breakage pur), vente à perte des singles signalée, marge membre négative formulée clairement. (6) Point courant des graphiques interpolé exactement sur la valeur du curseur. (7) Tableau détail : volumes fractionnaires affichés à 1 décimale pour que les multiplications recoupent les montants. (8) Coquille README. (9) Terminologie unifiée (single session), caption graphique 2, id canvas renommé. (10) Alerte créneaux restants corrigée (seuil et pluriel). Invariants revérifiés numériquement après correctifs. Redéployé.
- Prix par défaut tirés du site officiel (source de vérité) : séance praticien CHF 160-170, machines CHF 60-130, groupe CHF 20-40.

### Pistes d'évolution possibles (non demandées à ce stade)

- Distinguer 2-3 catégories de séances (praticien / machine / groupe) avec capacités séparées
- Ajouter une dimension temporelle (rampe de montée en charge, attrition des membres)
- Presets de scénarios (pessimiste / réaliste / optimiste)
- Membership mensuel en plus de l'annuel

### Conventions Samma à respecter si le contenu évolue

- Pas de "clinic"/"clinique" : dire "centre"
- Pas de "Functional Medicine" : dire "Integrative Health"
- Pas de tiret cadratin (em dash U+2014) dans les contenus
- Adresse : 2-4 Place du Molard, 3e étage, 1204 Genève
- Tarifs et faits : toujours vérifier sur le site officiel (`website/current/V_new_00.2/src/pages/fr/practices/`), jamais inventer
