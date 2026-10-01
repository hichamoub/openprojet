# Plan de mise en place : Odoo + OpenProject pour la gestion de projets informatiques

## 1. Objectif et périmètre

Mettre en place une plateforme intégrée où :
- **Odoo** gère le volet *commercial, financier et RH* : clients, devis/commandes, facturation, achats, temps facturable, comptabilité analytique, paie/congés.
- **OpenProject** gère le volet *pilotage de projet* : planification (Gantt), backlog et sprints (agile), work packages, jalons, suivi d'avancement, gestion des risques, documentation.
- Une **intégration** synchronise les données communes (clients, projets, temps passés, statuts) pour éviter la double saisie.

**Principe directeur : une seule source de vérité par donnée.**

| Donnée | Source de vérité | Consommateur |
|---|---|---|
| Clients / contacts | Odoo (CRM/Contacts) | OpenProject (lecture) |
| Devis, commandes, facturation | Odoo | - |
| Projet (fiche, budget vendu) | Odoo (commande confirmée) | OpenProject (création auto) |
| Planning, tâches, sprints, jalons | OpenProject | Odoo (avancement) |
| Temps passé | OpenProject (saisie) | Odoo (facturation, analytique) |
| Utilisateurs / SSO | Annuaire (Keycloak / LDAP / Azure AD) | Odoo, OpenProject |

## 2. Hypothèses à valider (questions ouvertes)

1. Taille de l'équipe et nombre d'utilisateurs (Odoo vs OpenProject) ?
2. Méthode dominante : cascade, agile/Scrum, hybride ?
3. Modèle économique : régie (temps passé), forfait, TMA/support, mixte ?
4. Édition : Odoo Community ou Enterprise ? OpenProject Community ou Enterprise (Gantt avancé, board agile, SSO, etc.) ?
5. Hébergement : cloud (SaaS), serveur dédié, on-premise ? Contraintes RGPD / souveraineté ?
6. Outils existants à migrer (Jira, Excel, Redmine, autre ERP) ?
7. Besoin de facturation électronique (conformité locale) et de comptabilité complète dans Odoo ?

## 3. Architecture cible

```
           ┌────────────── SSO (Keycloak / OIDC) ──────────────┐
           │                                                   │
   ┌───────▼────────┐    API REST / webhooks     ┌─────────────▼────────┐
   │      Odoo      │◄──────────────────────────►│     OpenProject      │
   │ CRM, Ventes,   │   (connecteur / middleware) │ Gantt, Agile, WP,    │
   │ Facturation,   │                            │ Temps, Jalons        │
   │ Compta, RH     │                            └─────────────┬────────┘
   └───────┬────────┘                                          │
           │ PostgreSQL                                        │ PostgreSQL
   ┌───────▼────────┐                            ┌─────────────▼────────┐
   │  DB Odoo       │                            │  DB OpenProject      │
   └────────────────┘                            └──────────────────────┘
        Reverse proxy (Traefik/Nginx + TLS) · Sauvegardes · Supervision
```

- Déploiement **Docker Compose** (puis Kubernetes si la charge le justifie), une base PostgreSQL par application.
- Reverse proxy HTTPS (Let's Encrypt), sauvegardes chiffrées quotidiennes, supervision (Prometheus/Grafana ou équivalent).
- Environnements : **dev → recette → production**.

## 4. Stratégie d'intégration (point clé)

| Option | Avantages | Inconvénients | Recommandation |
|---|---|---|---|
| A. Sans intégration (2 outils séparés) | Rapide | Double saisie, erreurs | Non |
| B. Connecteur/module existant (OCA, marketplace) | Peu de dev | Couverture variable, maintenance tierce | À évaluer en PoC |
| C. Middleware sur mesure (API REST OpenProject + JSON-RPC/REST Odoo, webhooks) | Contrôle total | Développement + maintenance | **Recommandé si B insuffisant** |
| D. Tout dans Odoo (module Projet/Timesheet) | Un seul outil | Planification/agile moins riches | Alternative si besoin simple |

**Flux à synchroniser (v1) :**
1. Commande Odoo confirmée → création du projet OpenProject (modèle de projet selon le type) + work packages initiaux.
2. Clients/contacts Odoo → références côté OpenProject.
3. Temps saisis dans OpenProject → feuilles de temps Odoo (analytique + facturation régie).
4. Avancement / jalons OpenProject → mise à jour du statut projet et déclenchement de facturation par jalon dans Odoo.
5. Budget : heures vendues (Odoo) vs consommées (OpenProject), alertes de dépassement.

Règles techniques : idempotence (clé d'identifiant externe), file de réessai, journalisation, compte de service avec droits minimaux.

## 5. Phasage et planning indicatif (~20 semaines)

| Phase | Durée | Contenu | Livrables |
|---|---|---|---|
| 0. Cadrage | S1–S2 | Ateliers, processus cibles, choix d'éditions/hébergement, matrice RACI, risques | Cahier des charges, architecture validée |
| 1. Infrastructure | S3–S4 | Docker/serveurs, PostgreSQL, TLS, SSO, sauvegardes, environnements | Plateformes dev + recette opérationnelles |
| 2. Paramétrage Odoo | S5–S8 | CRM, Ventes, Facturation, Compta analytique, Timesheets, RH ; modèles de devis | Odoo paramétré, jeux de données de test |
| 3. Paramétrage OpenProject | S5–S8 | Types de work packages, workflows, rôles, modèles de projets (cascade/Scrum), champs personnalisés, tableaux de bord | OpenProject paramétré |
| 4. Intégration | S9–S13 | PoC (S9), développement des flux §4, tests de bout en bout, gestion d'erreurs | Connecteur + documentation technique |
| 5. Migration de données | S12–S14 | Nettoyage, import clients/projets/historique, rapprochement | Données migrées et contrôlées |
| 6. Recette (UAT) | S14–S16 | Scénarios métier, correction des anomalies, tests de charge/sécurité | PV de recette |
| 7. Formation | S15–S17 | Sessions par profil (direction, chefs de projet, développeurs, compta), guides | Supports + vidéos courtes |
| 8. Mise en production | S18 | Bascule (go/no-go), support renforcé | Production ouverte |
| 9. Hypercare & amélioration | S19–S20+ | Suivi, ajustements, KPI, backlog d'évolutions | Bilan et feuille de route |

Stratégie de déploiement recommandée : **pilote sur 1–2 projets** (S14–S16), puis généralisation.

## 6. Configuration fonctionnelle de référence

**OpenProject**
- Types : Epic, User Story, Tâche, Bug, Jalon, Risque, Livrable.
- Workflows de statuts par type ; rôles : Chef de projet, Développeur, Client (lecture/commentaire), PMO.
- Modèles de projet : *Forfait cascade*, *Scrum*, *TMA/support*.
- Modules : Gantt, Boards, Sprints, Suivi du temps, Budgets, Wiki, Réunions, Documents.
- Tableaux de bord : avancement, charge par personne, burndown, risques ouverts.

**Odoo**
- Applications : Contacts/CRM, Ventes, Facturation/Comptabilité, Projet + Feuilles de temps (minimal), Achats, Congés/RH.
- Comptabilité analytique : un compte analytique par projet.
- Produits de service : régie (à l'heure), forfait (par jalon), abonnement (TMA).

## 7. Sécurité, conformité, exploitation

- SSO (OIDC/SAML) + MFA ; principe du moindre privilège ; journalisation des accès.
- Sauvegardes quotidiennes + test de restauration trimestriel ; RPO ≤ 24 h, RTO ≤ 4 h (à confirmer).
- RGPD : registre des traitements, durées de conservation, hébergement dans l'UE.
- Mises à jour : politique de versions (LTS Odoo, versions stables OpenProject), test en recette avant production.
- Supervision et alertes (disponibilité, espace disque, files de synchronisation).

## 8. Gouvernance et équipe

| Rôle | Responsabilité |
|---|---|
| Sponsor | Arbitrage, budget |
| Chef de projet | Planning, risques, communication |
| Référent métier (x2) | Processus, recette |
| Consultant/intégrateur Odoo | Paramétrage Odoo |
| Administrateur OpenProject | Paramétrage OpenProject |
| Développeur intégration | Connecteur/API |
| DevOps | Infra, sauvegardes, supervision |

Comité de pilotage mensuel ; point d'avancement hebdomadaire.

## 9. Risques principaux

| Risque | Impact | Mitigation |
|---|---|---|
| Périmètre qui dérive | Retard, coût | Périmètre v1 figé, backlog v2 |
| Adoption faible | Échec du projet | Implication des utilisateurs clés, formation, pilote |
| Qualité des données migrées | Erreurs de facturation | Nettoyage + rapprochement avant import |
| Connecteur fragile / maintenance | Désynchronisation | Tests automatisés, supervision, version API épinglée |
| Double saisie persistante | Perte de gain | Règles de source de vérité claires |
| Licences/éditions sous-estimées | Dépassement budget | Validation des fonctionnalités Enterprise dès le cadrage |

## 10. Indicateurs de succès

- Zéro double saisie des temps et des clients.
- ≥ 95 % des temps saisis à J+2.
- Délai commande → projet créé < 1 jour ouvré.
- Écart budget vendu / réalisé visible en temps réel par projet.
- Taux d'adoption > 90 % à 3 mois ; satisfaction utilisateurs ≥ 4/5.

## 11. Prochaines étapes immédiates

1. Valider les hypothèses du §2 avec le sponsor.
2. Lancer un **PoC d'intégration** (commande Odoo → projet OpenProject + retour des temps) sur environnement de dev.
3. Chiffrer le budget (licences, infra, prestations, formation) sur la base du phasage §5.
4. Planifier l'atelier de cadrage (Phase 0).
