# 🚨 DEVLABCYB — Simulation Incident P1 AWS Lambda

> 🇫🇷 Simulation d'un incident de production · Rôle : System Administrator
> 🇬🇧 Production incident simulation · Role: System Administrator

---

## ⚡ Scénario / Scenario

🇫🇷 14h30. L'API de scoring `tokenybet-scoring-api` ne répond plus. Les paris sont bloqués.

🇬🇧 2:30 PM. The scoring API `tokenybet-scoring-api` is down. Incoming bets are blocked.

---

## 🔍 Stack simulée / Simulated stack

`AWS Lambda` · `Kinesis` · `DynamoDB` · `CloudWatch` · `Datadog` · `VPC Security Groups`

---

## 📋 Les 4 phases / The 4 phases

| # | 🇫🇷 | 🇬🇧 |
|---|---|---|
| 1 | Triage — Throttling 997/1000 détecté | Triage — Throttling 997/1000 detected |
| 2 | Réseau — Lag Kinesis 82s · Faille 0.0.0.0/0 | Network — Kinesis lag 82s · Security flaw |
| 3 | Ticket INC-BETCLIC-001 rédigé | Incident ticket INC-BETCLIC-001 written |
| 4 | Monitoring Datadog configuré | Datadog monitors configured |

---

## 🛠️ Sans compte AWS / No AWS account needed

🇫🇷 Ce lab a été simulé via un **dashboard interactif** — reproductible par tout étudiant.

🇬🇧 This lab was simulated via an **interactive dashboard** — reproducible by any student.

---

## 📖 Documentation complète / Full documentation

➡️ [README_Tok.md](./README_Tok.md)

---

## 👩‍💻 Li-Lise

Bachelor Systèmes & Réseaux, Cloud & Cybersécurité · Sup de Vinci Bordeaux

Alternance septembre 2026 · Bordeaux · FinTech · Banking · ESN

[![LinkedIn](https://img.shields.io/badge/LinkedIn-blue?style=flat&logo=linkedin)](https://linkedin.com/in/li-lise)
