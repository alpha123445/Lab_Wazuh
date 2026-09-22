# Lab Wazuh 

Ce lab c'est pour me préparer à un poste d'analyste SOC Junior


Je me suis aidé de l'IA pour créer les exercices mais la pratique derrière ainsi que la compréhension c'est moi, et voici donc le prompt que j'ai fais à claude.

**Prompt**

Agis en tant qu'Expert en Cyber-Réponse, Responsable de SOC (Security Operations Center) et Ingénieur Pédagogique chevronné. 

J'ai terminé les parcours théoriques SOC Level 1 et 2 sur TryHackMe. Pour casser l'effet "trop théorique" et me préparer concrètement au quotidien d'un Analyste SOC en entreprise, je veux mettre en place un projet de Lab Pratique Réaliste d'une semaine. L'objectif est de simuler un incident d'entreprise complexe de bout en bout, de configurer la télémétrie, de créer des règles de détection (IDS), de configurer le blocage automatique (IPS) et de documenter le tout pour mon CV et mon GitHub afin de décrocher un poste d'Analyste SOC Junior.

Voici mon infrastructure actuelle (tous les agents Wazuh sont déjà installés, connectés et fonctionnels) :
1. Ubuntu : Serveur SIEM Wazuh (Manager).
2. Kali Linux : Machine de l'attaquant.
3. Windows (Client) : Poste de travail d'un employé de l'entreprise.
4. Debian : Serveur Web de l'entreprise (Apache + SSH).

Conçois-moi un guide de projet jour par jour, ultra-technique, concret et orienté "métier", structuré exactement ainsi :

1. ENRICHISSEMENT DE LA TÉLÉMÉTRIE SOC (Jour 1-2) :
   - Explique-moi comment configurer finement la collecte sur Windows : déploiement de Sysmon (avec une configuration de référence comme celle de SwiftOnSecurity) et modification du fichier 'ossec.conf' de l'agent pour remonter ces Event IDs Sysmon spécifiques vers le manager.
   - Explique-moi comment activer la surveillance sur Debian : configuration des logs d'audit Linux (auditd pour tracer les exécutions de commandes suspectes), logs Apache (access.log/error.log) et logs d'authentification SSH, et comment les intégrer dans le fichier 'ossec.conf' de l'agent.

2. LE SCÉNARIO D'ATTAQUE RÉALISTE - "De l'accès initial au Pivot" (Jour 3-4) :
   - Propose un scénario d'attaque d'entreprise classique aligné sur le framework MITRE ATT&CK. 
     * Étape 1 (Windows) : Exécution d'un script ou binaire malveillant -> Établissement d'une persistance -> Vol d'identifiants (mémoire ou fichiers).
     * Étape 2 (Mouvement latéral) : Utilisation des identifiants volés pour se connecter à distance (SSH) sur le serveur Debian.
     * Étape 3 (Debian) : Escalade de privilèges (via une mauvaise configuration sudo ou SUID) -> Déploiement d'un webshell ou action malveillante sur Apache.
   - Donne-moi les commandes et outils exacts (PowerShell, Bash, outils Kali) à exécuter pour jouer ce scénario côté attaquant.

3. DÉTECTION AVANCÉE ET CRÉATION DE RÈGLES XML / IDS (Jour 5-6) :
   - Comment un analyste SOC doit investiguer ces étapes dans les dashboards de Wazuh (quels indicateurs chercher, quels Event IDs Sysmon analyser).
   - Fournis-moi le code XML complet d'une RÈGLE DE DÉTECTION PERSONNALISÉE (Custom Rule) à ajouter sur le manager Wazuh pour détecter un comportement spécifique et furtif de ce scénario (qui ne déclencherait pas d'alerte critique par défaut). Explique-moi comment la tester pour qu'elle lève une alerte de niveau critique (Level 10+).

4. RÉPONSE ACTIVE / IPS ET RAPPORT D'INCIDENT (Jour 7) :
   - Explique-moi étape par étape comment configurer l' "Active Response" dans Wazuh pour lier ma règle personnalisée à un blocage automatisé en temps réel (ex: isolation réseau automatique du poste Windows ou bannissement immédiat de l'IP de la Kali via le pare-feu de la Debian).
   - Fournis-moi un modèle de rapport d'incident (Incident Report) au format professionnel de type SOC (Résumé exécutif, Chronologie/Timeline de l'attaque, Analyse technique, Recommandations de remédiation).

Pour finir, donne-moi des conseils précis pour packager et documenter ce projet sur GitHub de manière professionnelle, ainsi que des exemples de lignes d'impact percutantes à intégrer directement dans la section "Projets" ou "Expériences" de mon CV pour valoriser ce lab auprès des recruteurs. Donne-moi des instructions très concrètes et techniques.
