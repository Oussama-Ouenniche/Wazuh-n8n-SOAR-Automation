# Wazuh-n8n-SOAR-Automation

Automatisation SOAR (Security Orchestration, Automation and Response) intégrant **Wazuh** et **n8n** : une attaque par force brute SSH détectée par Wazuh déclenche automatiquement, via une intégration Python et un webhook n8n, l'envoi d'une alerte email formatée — sans aucune intervention manuelle.

## 📋 Contexte

Ce projet illustre comment connecter un SIEM open-source (Wazuh) à une plateforme d'automatisation de workflows (n8n) afin de réduire le temps de réponse d'un SOC face à un incident de sécurité, sans dépendre d'une solution SOAR commerciale.

Il s'inscrit dans la continuité du projet [SOC-AI](https://github.com/Oussama-Ouenniche/soc-ai-active-directory), en se concentrant cette fois sur l'orchestration et la réponse automatisée plutôt que sur la détection par machine learning.

## 🎯 Cas d'usage

Un attaquant multiplie les tentatives de connexion SSH échouées sur une machine Ubuntu surveillée par un agent Wazuh. La règle Wazuh **5551 — PAM: Multiple failed logins in a small period of time** (niveau 10) se déclenche, ce qui lance toute la chaîne d'automatisation jusqu'à la notification email.

## 🏗️ Architecture

```
Wazuh SIEM  →  Custom Integration (Python)  →  n8n Webhook  →  Gmail Notification
```

1. **Wazuh SIEM** détecte l'anomalie (ex. règle 5551, échecs de connexion SSH répétés) et génère une alerte JSON.
2. **Custom Integration** : un script Python (`wazuh/integrations/custom-n8n`), appelé automatiquement par le Wazuh Manager, lit le fichier d'alerte et le transmet en `POST` à l'URL du webhook n8n.
3. **n8n Webhook** reçoit l'alerte JSON complète (`rule`, `agent`, `timestamp`, `full_log`, etc.).
4. **Gmail Notification** : le node Gmail formate l'alerte (type d'événement, ID de règle, sévérité, agent, horodatage, log complet) et envoie un email HTML au SOC.

Diagramme complet disponible dans `docs/architecture_wazuh_n8n_gmail.png`.

## 📁 Structure du repo

```
Wazuh-n8n-SOAR-Automation/
├── docs/
│   └── architecture_wazuh_n8n_gmail.png   # Schéma d'architecture
├── n8n/
│   └── workflow.json                       # Workflow n8n exportable (Webhook → Gmail)
├── screenshots/
│   ├── Wazuh ALERT SOAR .png                # Exemple d'email d'alerte reçu
│   ├── capture soar .png
│   ├── n8n workflow.png                    # Vue du workflow dans n8n
│   └── script python de soar.png
└── wazuh/
    ├── integrations/
    │   └── custom-n8n                      # Script Python d'intégration Wazuh → n8n
    └── ossec.conf.example                  # Exemple de configuration de l'intégration
```

## ⚙️ Fonctionnement du workflow n8n

Le workflow (`n8n/workflow.json`) contient deux nodes :

- **Webhook** (`n8n-nodes-base.webhook`) : endpoint `POST /wazuh-alert-webhook` qui reçoit l'alerte brute envoyée par le script Python.
- **Send a message** (`n8n-nodes-base.gmail`) : construit et envoie un email HTML reprenant les champs clés de l'alerte (`{{$json.body.rule.id}}`, `{{$json.body.rule.level}}`, `{{$json.body.agent.name}}`, `{{$json.body.full_log}}`, etc.) à l'adresse du SOC.

## 🛠️ Stack technique

| Composant | Rôle |
|---|---|
| **Wazuh** | SIEM — collecte des logs, détection (règle PAM 5551) et génération d'alertes |
| **Python** | Script d'intégration `custom-n8n` exécuté par Wazuh, POST de l'alerte vers n8n |
| **n8n** | Orchestrateur du workflow (Webhook → Gmail) |
| **Gmail** | Canal de notification du SOC |

## 🚀 Installation et utilisation

### Prérequis

- Une instance Wazuh (Manager + Agent) fonctionnelle
- Une instance n8n (self-hosted ou cloud) avec des credentials Gmail configurés
- Python 3 avec le module `requests` sur le Wazuh Manager

### Étapes

1. **Cloner le repo**
   ```bash
   git clone git@github.com:Oussama-Ouenniche/Wazuh-n8n-SOAR-Automation.git
   cd Wazuh-n8n-SOAR-Automation
   ```

2. **Déployer le script d'intégration**
   ```bash
   cp wazuh/integrations/custom-n8n /var/ossec/integrations/custom-n8n
   chmod 750 /var/ossec/integrations/custom-n8n
   chown root:wazuh /var/ossec/integrations/custom-n8n
   ```

3. **Configurer l'intégration dans `ossec.conf`** (voir `wazuh/ossec.conf.example`)
   ```xml
   <integration>
     <name>custom-n8n</name>
     <hook_url>https://<votre-instance-n8n>/webhook/wazuh-alert-webhook</hook_url>
     <level>7</level>
     <alert_format>json</alert_format>
   </integration>
   ```
   puis redémarrer le manager : `systemctl restart wazuh-manager`

4. **Importer le workflow n8n**
   - Dans n8n : `Import from File` → `n8n/workflow.json`
   - Configurer les credentials Gmail sur le node **Send a message**
   - Remplacer `SOC-EMAIL@gmail.com` par l'adresse réelle du SOC
   - Activer le workflow

5. **Simuler l'attaque**
   - Générer plusieurs échecs d'authentification SSH sur une machine avec agent Wazuh (déclenche la règle 5551, niveau 10)

6. **Vérifier**
   - L'alerte apparaît dans le Wazuh Dashboard
   - Le webhook n8n s'exécute (visible dans l'historique d'exécutions n8n)
   - L'email d'alerte est reçu automatiquement (cf. `screenshots/Wazuh ALERT SOAR .png`)

## 📌 Améliorations possibles

- Filtrage/dédoublonnage des alertes directement dans n8n avant notification
- Ajout d'un blocage automatique de l'IP source (réponse active, ex. `active-response`)
- Enrichissement des alertes via une API de threat intelligence (VirusTotal, AbuseIPDB)
- Mapping MITRE ATT&CK des alertes traitées
- Notification multi-canal (Slack, Telegram) en plus de Gmail

## 👤 Auteur

**Oussama Ouenniche**
Licence en Télécommunications — ISITCOM, Université de Sousse
Projet réalisé dans le cadre d'un stage d'été, en complément du projet [SOC-AI](https://github.com/Oussama-Ouenniche/soc-ai-active-directory).

