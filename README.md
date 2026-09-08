# Wazuh SOC Junior Labs

Une collection de labs pratiques destinés aux personnes qui souhaitent développer leurs compétences en **SOC / Blue Team** en utilisant **Wazuh**.

L'objectif de ces exercices est de reproduire différents scénarios de sécurité dans un environnement de laboratoire et de travailler le cycle :

**Détection → Investigation → Qualification → Réponse**

---

## Objectif

Ces labs sont conçus pour un **Analyste SOC Junior** qui souhaite pratiquer Wazuh sur des scénarios concrets :

- attaques par force brute ;
- élévation de privilèges ;
- compromission de comptes ;
- persistance Windows ;
- PowerShell ;
- accès à LSASS ;
- reconnaissance réseau ;
- fichiers suspects ;
- surveillance de l'intégrité des fichiers ;
- etc.

L'objectif n'est pas uniquement de faire apparaître une alerte dans Wazuh.

Pour chaque scénario, il faut être capable de comprendre :

> **ce qui s'est passé, pourquoi Wazuh a généré l'alerte, si l'activité est réellement malveillante et quelle réponse doit être apportée.**

---

# Pré-requis

Ces labs supposent que tu possèdes déjà les **bases de Wazuh**.

Tu dois notamment être capable d'installer, configurer et utiliser un environnement Wazuh avant de commencer les exercices.

## Wazuh

Tu dois connaître les bases de :

- installation de Wazuh ;
- architecture Wazuh ;
- Wazuh Manager ;
- Wazuh Indexer ;
- Wazuh Dashboard ;
- agents Wazuh ;
- groupes d'agents ;
- configuration des agents ;
- collecte de logs ;
- règles et décodeurs ;
- niveaux de sévérité ;
- recherche d'alertes ;
- `wazuh-logtest` ;
- Active Response ;
- File Integrity Monitoring (FIM / `syscheck`).

---

# Environnement recommandé

Pour réaliser les labs, un environnement de virtualisation est fortement recommandé.

### Infrastructure minimale

```text
                 ┌──────────────────┐
                 │   Wazuh Manager  │
                 │     + Indexer    │
                 │    + Dashboard   │
                 └────────┬─────────┘
                          │
             ┌────────────┼────────────┐
             │                         │
      ┌──────▼──────┐          ┌──────▼──────┐
      │ Linux Agent │          │Windows Agent│
      │   Debian    │          │ Windows 10/11│
      └─────────────┘          └─────────────┘
             │                         │
             └────────────┬────────────┘
                          │
                   ┌──────▼──────┐
                   │ Kali Linux  │
                   │   Attacker  │
                   └─────────────┘
```

---

# Configuration nécessaire

Avant de commencer les labs, ton environnement doit au minimum permettre :

### Linux

La collecte des événements d'authentification :

```text
/var/log/auth.log
```

ou les journaux équivalents de ta distribution.

### Windows

La collecte des journaux :

```text
Windows Security
System
Application
```

Pour les labs Windows avancés, **Sysmon** est recommandé afin de disposer de logs plus détaillés sur les processus, fichiers et activités système.

Les événements Sysmon particulièrement utilisés dans ces labs comprennent notamment :

```text
Event ID 1  → Process Creation
Event ID 10 → Process Access
Event ID 11 → File Create
```

---

# Fonctionnalités Wazuh utilisées

Les exercices vont progressivement utiliser différentes fonctionnalités de Wazuh :

```text
Agents
   ↓
Log collection
   ↓
Decoders
   ↓
Rules
   ↓
Alerts
   ↓
Correlation
   ↓
Investigation
   ↓
Active Response
```

Certains labs utilisent également :

```text
Sysmon
Suricata
File Integrity Monitoring
VirusTotal
Kali Linux
```

---

# Prérequis techniques

Il est recommandé d'avoir les bases suivantes avant de commencer :

### Linux

- commandes Linux ;
- SSH ;
- utilisateurs et groupes ;
- permissions ;
- `sudo` ;
- processus ;
- logs ;
- `iptables` / `nftables` ;
- réseau de base.

### Windows

- utilisateurs et groupes ;
- comptes locaux ;
- Event Viewer ;
- Windows Event Logs ;
- PowerShell ;
- processus Windows ;
- services ;
- groupes Administrateurs.

### Réseau

- TCP/IP ;
- ports ;
- services ;
- adresses IP ;
- DNS ;
- notions de firewall ;
- bases de Nmap.

### Sécurité

- authentification ;
- brute force ;
- élévation de privilèges ;
- persistance ;
- reconnaissance ;
- IOC ;
- True Positive / False Positive ;
- triage d'alertes.

---

# Important

Tous les exercices doivent être réalisés **uniquement dans un environnement de laboratoire que tu contrôles ou pour lequel tu as une autorisation explicite**.

Les attaques simulées dans ces labs ont pour objectif de générer des événements de sécurité afin de pratiquer leur détection et leur investigation avec Wazuh.

---


## Objectif final

À la fin de ces exercices, tu dois être capable de prendre une alerte Wazuh et de répondre à cinq questions :

> **Qu'est-ce qui s'est passé ?**  
> **Est-ce normal ?**  
> **Est-ce malveillant ?**  
> **Quel est le risque ?**  
> **Que doit faire le SOC ?**


**BAH Alpha**  
*Analyste SOC Junior | Cybersécurité | Blue Team*
