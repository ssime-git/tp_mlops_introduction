# Correction TP MLOps - Architecture Évolutive

## ÉTAPE 1 – Prototype local Data Scientist

### Nouveau besoin identifié
Un collègue métier (technicien maintenance) doit pouvoir tester le modèle de détection d'anomalies sans installer Python ni manipuler du code.

### Architecture proposée

```mermaid
graph TD
    USER[Utilisateur Métier]
    STREAMLIT[Interface Streamlit]
    MODEL[model.pkl]
    DATA[Données CSV upload]
    
    USER --> STREAMLIT
    STREAMLIT --> MODEL
    DATA --> STREAMLIT
    STREAMLIT --> RESULTS[Résultats + Graphiques]
```

### Justifications techniques

**Choix Streamlit** : Interface web simple sans développement frontend complexe. Le technicien peut uploader ses fichiers CSV et voir les résultats visuellement.

**Pattern utilisé** : Application web intégrée (selon masterclass - modèle inclus dans l'application)

**Isolation environnement** : 
```bash
python -m venv venv_anomalies
pip install streamlit scikit-learn pandas
```

### Code conceptuel
```python
# app_streamlit.py
import streamlit as st
import joblib
import pandas as pd

model = joblib.load('anomaly_detector.pkl')

uploaded_file = st.file_uploader("Données capteurs", type=['csv'])
if uploaded_file:
    data = pd.read_csv(uploaded_file)
    predictions = model.predict(data)
    st.write("Anomalies détectées:", predictions.sum())
```

## ÉTAPE 2 – Outil interne partagé

### Nouveaux besoins identifiés
- Techniciens sur 3 sites différents (Paris, Lyon, Toulouse)
- Éviter les problèmes "ça marche sur mon PC"
- Accès simultané sans installation locale

### Architecture proposée

```mermaid
graph TD
    SITE1[Site Paris]
    SITE2[Site Lyon] 
    SITE3[Site Toulouse]
    
    SITE1 --> API[API FastAPI]
    SITE2 --> API
    SITE3 --> API
    
    API --> DOCKER[Container Docker]
    DOCKER --> MODEL[model.pkl]
    DOCKER --> LOGS[logs.txt]
    
    SERVEUR[Serveur interne] --> DOCKER
```

### Justifications techniques

**Migration vers API dédiée** : Séparation logique métier/ML, accès concurrent, meilleure robustesse.

**Protocole HTTP/REST** : Standard, firewall-friendly, documentation automatique avec FastAPI.

**Containerisation Docker** : 
- Reproductibilité garantie (même environnement partout)
- Isolation des dépendances
- Déploiement simplifié

**Logs basiques** : Traçabilité minimale pour debugging
```python
logging.basicConfig(filename='predictions.log', level=logging.INFO)
logging.info(f"Prediction for {user_id}: {result}")
```

### Pattern utilisé
Architecture dédiée avec API fonctionnelle (transformation des données capteurs en prédictions d'anomalies)


## ÉTAPE 3 – Démarrage industriel (pré-production)

### Nouveaux besoins identifiés
- Traitement en continu des données de 50 équipements
- Historisation pour analyse des tendances
- Tests de nouvelles versions modèle
- Monitoring performance système

### Architecture proposée

```mermaid
graph TD
    EQUIPEMENTS[50 Équipements industriels]
    
    EQUIPEMENTS --> BATCH[Pipeline Batch Quotidien]
    EQUIPEMENTS --> STREAMING[Flux Temps Réel]
    
    BATCH --> PREPROCESS[Feature Engineering]
    STREAMING --> PREPROCESS
    
    PREPROCESS --> API_V1[API ML v1.2.3]
    API_V1 --> DB[(Base Prédictions)]
    
    GIT[Git Repository] --> DOCKER_BUILD[Docker Build]
    DOCKER_BUILD --> API_V1
    
    MONITORING[Logs + Métriques] --> API_V1
    DB --> DASHBOARD[Dashboard Grafana]
```

### Justifications techniques

**Pipeline de données structuré** :
- Batch pour analyses historiques (nuit, données complètes)
- Streaming pour alertes temps réel (seuils critiques)

**Versioning sémantique** : v1.2.3 (major.minor.patch)
- Git tags pour traçabilité
- Dockerfile avec version figée des dépendances

**Base de données** : PostgreSQL pour stockage prédictions
```sql
CREATE TABLE predictions (
    id SERIAL PRIMARY KEY,
    equipment_id VARCHAR(50),
    timestamp TIMESTAMP,
    model_version VARCHAR(20),
    anomaly_score FLOAT,
    is_anomaly BOOLEAN
);
```

**Monitoring technique** :
- Latence API (< 100ms cible)
- Taux d'erreur (< 1%)
- Utilisation mémoire/CPU

### Pattern utilisé
Architecture dédiée évoluée avec pipeline ETL et versioning


## ÉTAPE 4 – Mise en production industrielle

### Nouveaux besoins identifiés
- Certification industrielle (traçabilité complète)
- Zéro interruption de service
- Détection automatique de dérive modèle
- Sécurité renforcée (authentification, audit)

### Architecture proposée

```mermaid
graph TD
    subgraph "Zone DMZ"
        LB[Load Balancer + SSL]
        AUTH[Service Authentification]
    end
    
    subgraph "Zone Production"
        API_BLUE[API v2.1.0 - BLUE]
        API_GREEN[API v2.0.5 - GREEN]
        MODEL_REGISTRY[Registry Modèles]
    end
    
    subgraph "Données"
        DB_PROD[(DB Production)]
        DB_AUDIT[(DB Audit)]
        FEATURE_STORE[Feature Store]
    end
    
    subgraph "Monitoring"
        PROMETHEUS[Prometheus Metrics]
        ALERTS[Alerting]
        DRIFT_DETECTION[Monitoring Dérive]
    end
    
    subgraph "CI/CD"
        GITHUB[GitHub Actions]
        TESTS[Tests Auto]
        DEPLOY[Déploiement Blue/Green]
    end
    
    USERS[Utilisateurs] --> LB
    LB --> AUTH
    AUTH --> API_BLUE
    AUTH --> API_GREEN
    
    API_BLUE --> MODEL_REGISTRY
    API_GREEN --> MODEL_REGISTRY
    
    API_BLUE --> DB_PROD
    API_GREEN --> DB_PROD
    
    GITHUB --> TESTS --> DEPLOY
    DEPLOY --> API_GREEN
    
    PROMETHEUS --> DRIFT_DETECTION
    DRIFT_DETECTION --> ALERTS
```

### Justifications techniques

**Authentification granulaire** :
- JWT tokens avec rôles (lecteur/admin)
- Audit trail de tous les accès
```python
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time.time()
    response = await call_next(request)
    # Log: user, endpoint, duration, response
    return response
```

**Blue/Green Deployment** :
- Zéro downtime lors des mises à jour
- Test en conditions réelles sur GREEN
- Bascule instantanée si validation OK

**Monitoring métier** :
- Dérive détectée par comparaison distributions
- Alertes automatiques si performance < seuil
- Métriques business (taux détection, faux positifs)

**CI/CD Pipeline** :
```yaml
# .github/workflows/deploy.yml
on: push
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Run ML Tests
        run: pytest tests/
  
  deploy:
    needs: test
    steps:
      - name: Build Docker
      - name: Deploy Green
      - name: Health Check
      - name: Switch Traffic
```

**Sécurité renforcée** :
- Chiffrement TLS obligatoire
- Validation stricte inputs (Pydantic)
- Secrets management (variables d'environnement)
- Logs d'audit horodatés et signés

### Pattern utilisé
Architecture microservices avec orchestration complète


## Évolution architecturale - Vue d'ensemble

### Progression logique

1. **Étape 1** : Application monolithique locale (Streamlit intégré)
2. **Étape 2** : Séparation API/Interface (architecture dédiée)
3. **Étape 3** : Pipeline de données + versioning (pré-industriel)
4. **Étape 4** : Microservices + automatisation (production industrielle)

### Transitions justifiées

**1→2** : Besoin de partage = nécessité d'une API réseau

**2→3** : Volume/criticité = pipeline robuste + traçabilité

**3→4** : Industrialisation = automatisation + sécurité + fiabilité

### Technologies utilisées (conforme masterclass)

- **APIs** : FastAPI (performance + documentation)
- **Containerisation** : Docker (reproductibilité)
- **Versioning** : Git + tags sémantiques
- **Monitoring** : Logs + Prometheus (métriques)
- **CI/CD** : GitHub Actions
- **Base données** : PostgreSQL (audit + performance)

### Critères MLOps respectés

* **Reproductibilité** : Docker + versioning Git
* **Scalabilité** : Load balancer + architecture stateless
* **Monitoring** : Métriques techniques + métier
* **Sécurité** : Authentification + chiffrement + audit
* **Automatisation** : CI/CD complet
* **Robustesse** : Tests automatisés + blue/green deployment

### Points critiques identifiés

* **Data drift** : Surveillance automatique distribution features
* **Model decay** : Monitoring performance métier en continu
* **Security** : Principe moindre privilège + logs d'audit
* **Reliability** : SLA 99.9% avec mécanismes failover

Cette évolution respecte la philosophie MLOps : partir simple et complexifier seulement quand le besoin métier le justifie, en maintenant toujours reproductibilité et observabilité.