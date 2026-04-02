# 🚨 DEVLABCYB — Simulation Incident P1 AWS Lambda
### Rôle simulé : System Administrator · Environnement : AWS Lambda · Kinesis · DynamoDB · Datadog

---

> 🇫🇷 **Français** | 🇬🇧 [English below](#-devlabcyb--aws-lambda-p1-incident-simulation)

---

## 📋 Contexte / Context

**Scénario :** 14h30. Le dev Kotlin de la Squad signale que l'API de scoring `tokenybet-scoring-api` ne répond plus depuis 10 minutes. Les paris entrants sont bloqués.

**Rôle :** System Administrator — diagnostiquer, résoudre, documenter, monitorer.

**Environnement simulé :** Dashboard interactif Claude (sans compte AWS requis).

---

## 🏗️ Architecture simulée

```
Parieurs
   │
   ▼
Kinesis Stream (tokenybet-bets-stream)
   │  flux de paris en temps réel
   ▼
AWS Lambda (tokenybet-scoring-api)
   │  traitement du scoring
   ▼
DynamoDB (stockage des résultats)
   │
   ▼
ALB (Application Load Balancer)
   │
   ▼
Utilisateurs finaux
```

---

## 🔴 Phase 1 — Triage initial (5 min)

### Actions réalisées

| Outil | Métrique | Résultat |
|---|---|---|
| Lambda Console → Monitor | Throttling | **997 / 1000** exécutions simultanées |
| CloudWatch Logs | Erreurs | `NullPointerException` non catchée · cold start +4200ms |
| ALB → Health checks | Targets | **0 / 2** targets healthy · HTTP 504 |

### 🔍 Diagnostic

- **Throttling confirmé** : Lambda a atteint sa limite de concurrence (1000 exécutions simultanées)
- **Analogie** : Le parking est plein (997/1000 places). Les nouvelles requêtes reçoivent une erreur **HTTP 429 Too Many Requests** et sont rejetées.
- **Conséquence** : Les paris entrants ne sont plus traités.

### ✅ Action immédiate

> Augmentation de la **concurrence réservée Lambda** : `1000 → 1500`

**Pourquoi 1500 ?**
- 1200 : trop proche du seuil, risque de re-saturation rapide
- 1500 : augmentation de 50%, absorbe le pic sans surcharger DynamoDB
- 3000 : trop brutal, risque de facture AWS explosive et surcharge en aval

---

## 🟡 Phase 2 — Diagnostic réseau (5 min)

### Kinesis Stream — Lag détecté

| Métrique | Valeur | Seuil acceptable |
|---|---|---|
| `GetRecords.IteratorAgeMilliseconds` | **82 000 ms** | < 5 000 ms |

**Chaîne causale identifiée :**

```
Pic de paris à 14h20 (IncomingRecords ↑)
        ↓
Kinesis accumule → lag 82 secondes
        ↓
Lambda reçoit un burst massif → saturation 997/1000
        ↓
Throttling → paris bloqués
```

### VPC Security Groups — Faille détectée

| Règle | Type | Port | Destination | Statut |
|---|---|---|---|---|
| Sortante | HTTPS | 443 | `0.0.0.0/0` | ⚠️ Trop permissive |

**Problème :** `0.0.0.0/0` autorise tout internet comme destination.

**Solution recommandée :** Restreindre au **VPC Endpoint DynamoDB** (`pl-xxxxxxx`) — communication privée sans transiter par internet.

```bash
# Vérification via AWS CLI
aws ec2 describe-security-groups \
  --group-ids sg-0abc123 \
  | grep -i "0.0.0.0"
```

---

## 📝 Phase 3 — Ticket d'incident

```
INC-BETCLIC-001 | Sévérité : P1
────────────────────────────────────────────
Service      : tokenybet-scoring-api (Lambda)
Détecté      : 14h20
Déclaré      : 14h30 (dev Kotlin Squad)
────────────────────────────────────────────
Symptôme     : Pic de paris à 14h20 — paris entrants bloqués
Cause racine : Lag Kinesis 82s → Lambda saturé 997/1000
────────────────────────────────────────────
Action réalisée :
  • Concurrence Lambda augmentée 1000 → 1500
  • Règle 0.0.0.0/0 signalée — VPC Endpoint à configurer
  • P1 ouvert · Dev Kotlin notifié via Slack #tokenybet-alerts
────────────────────────────────────────────
Action restante :
  • NullPointerException non catchée → patch dev Kotlin
  • Security Group → restreindre au VPC Endpoint DynamoDB
```

### Message Slack au dev Kotlin

> *"Hey, P1 ouvert sur `tokenybet-scoring`. Throttling Lambda confirmé 997/1000, lag Kinesis à 82s. J'ai monté la concurrence réservée à 1500 — ça devrait se résorber. De ton côté tu peux regarder la `NullPointerException` non catchée dans les logs CloudWatch ? Elle aggrave la saturation. Dis-moi quand c'est patché."*

---

## 📊 Phase 4 — Monitoring préventif Datadog

### Monitors configurés

| Métrique | Seuil Warning (70%) | Seuil Critique (90%) | Notification |
|---|---|---|---|
| `aws.lambda.concurrent_executions` | > **1050** | > **1350** | `@slack-tokenybet-alerts` |
| `aws.lambda.errors` | > **5** / 5 min | > **10** / 5 min | `@slack-tokenybet-alerts` |

**Logique des seuils :**
- **70%** → alerte warning : tu surveilles, tu anticipes
- **90%** → alerte critique : tu interviens immédiatement
- **100%** → trop tard, c'est l'incident de ce soir

### Tags Datadog

```
env:prod
service:tokenybet
team:scoring
region:eu-west-1
```

### Escalade

```
Alerte Datadog → #tokenybet-alerts (Slack)
        │
        ▼ si non résolu après 15 min
@pagerduty-oncall (astreinte)
```

---

## 🧠 Concepts clés abordés

| Concept | Définition rapide |
|---|---|
| **Throttling** | Limitation volontaire d'AWS — HTTP 429 quand la concurrence max est atteinte |
| **Concurrence réservée** | Nombre max d'exécutions Lambda simultanées |
| **IteratorAgeMilliseconds** | Lag entre l'entrée dans Kinesis et la consommation par Lambda |
| **Security Group** | Règles réseau par ressource (≈ ACL Cisco par appareil) |
| **VPC Endpoint** | Connexion privée AWS sans passer par internet |
| **KPI** | Indicateur de performance business (ex: latence < 200ms) |
| **CLI** | Interface en ligne de commande — AWS CLI = terminal AWS |
| **Stream** | Flux de données continu en temps réel |
| **INC** | Incident — nomenclature ITIL standard |
| **SLA** | Service Level Agreement — contrat de niveau de service |

---

## 🛠️ Environnement de simulation

> Ce lab a été réalisé sans compte AWS via un **dashboard interactif** généré par Claude (Anthropic).
>
> ✅ Accessible sans abonnement AWS
> ✅ Reproductible par tout étudiant
> ✅ Idéal pour pratiquer le diagnostic d'incident cloud

**Pour pratiquer avec un vrai compte AWS :**
- [AWS Free Tier](https://aws.amazon.com/free/) — 1M requêtes Lambda/mois gratuites
- [LocalStack](https://localstack.cloud/) — émulation AWS en local via Docker
- [AWS Skill Builder](https://skillbuilder.aws/) — labs guidés sans carte bancaire

---

## 🔗 Liens DEVLABCYB

| Repo | Description |
|---|---|
| [cgi-lbp-devsecops-lab](https://github.com/Li-Lise/cgi-lbp-devsecops-lab) | Lab principal — Active Directory · ITSM · PowerShell · Docker |

---

## 👩‍💻 Profil

**Li-Lise** — Étudiante Bachelor Systèmes & Réseaux, Cloud & Cybersécurité · Sup de Vinci Bordeaux

Recherche alternance septembre 2026 · Bordeaux · FinTech · Banking · ESN

[![LinkedIn](https://img.shields.io/badge/LinkedIn-blue?style=flat&logo=linkedin)](https://linkedin.com/in/li-lise)

---

---

# 🇬🇧 DEVLABCYB — AWS Lambda P1 Incident Simulation
### Simulated role: System Administrator · Stack: AWS Lambda · Kinesis · DynamoDB · Datadog

---

## 📋 Context

**Scenario:** 2:30 PM. The Kotlin developer from the Squad reports that the scoring API `tokenybet-scoring-api` has been unresponsive for 10 minutes. Incoming bets are blocked.

**Role:** System Administrator — diagnose, resolve, document, monitor.

**Simulated environment:** Interactive Claude dashboard (no AWS account required).

---

## 🔴 Phase 1 — Initial Triage (5 min)

| Tool | Metric | Result |
|---|---|---|
| Lambda Console → Monitor | Throttling | **997 / 1000** concurrent executions |
| CloudWatch Logs | Errors | Uncaught `NullPointerException` · cold start +4200ms |
| ALB → Health checks | Targets | **0 / 2** healthy · HTTP 504 |

**Root cause:** Lambda reached its concurrency limit. New requests receive **HTTP 429 Too Many Requests**.

**Action:** Reserved concurrency increased `1000 → 1500`

---

## 🟡 Phase 2 — Network Diagnosis (5 min)

**Kinesis lag:** `GetRecords.IteratorAgeMilliseconds` = **82,000 ms** (threshold: < 5,000 ms)

**Causal chain:**
```
Bet spike at 2:20 PM → Kinesis lag 82s → Lambda burst → Throttling 997/1000 → Bets blocked
```

**Security Group issue:** Outbound rule `0.0.0.0/0` on port 443 — too permissive.

**Fix:** Restrict to DynamoDB VPC Endpoint (`pl-xxxxxxx`).

---

## 📝 Phase 3 — Incident Ticket

```
INC-BETCLIC-001 | Severity: P1
Service      : tokenybet-scoring-api (Lambda)
Detected     : 2:20 PM | Reported: 2:30 PM
Symptom      : Bet spike → bets blocked
Root cause   : Kinesis lag 82s → Lambda saturated 997/1000
Action taken : Lambda concurrency 1000 → 1500 · Security Group flagged
Remaining    : Uncaught NullPointerException → Kotlin dev patch
```

---

## 📊 Phase 4 — Preventive Monitoring (Datadog)

| Metric | Warning (70%) | Critical (90%) | Notify |
|---|---|---|---|
| `aws.lambda.concurrent_executions` | > 1050 | > 1350 | `@slack-tokenybet-alerts` |
| `aws.lambda.errors` | > 5 / 5min | > 10 / 5min | `@slack-tokenybet-alerts` |

**Tags:** `env:prod` · `service:tokenybet` · `team:scoring` · `region:eu-west-1`

---

## 🛠️ Simulation Environment

> This lab was completed without an AWS account using an **interactive dashboard** built with Claude (Anthropic).
>
> ✅ No AWS subscription needed
> ✅ Reproducible by any student
> ✅ Great for practicing cloud incident response

---

*DEVLABCYB — Simulated IT subcontractor lab · CGI / La Banque Postale context · Aerospace sector*
