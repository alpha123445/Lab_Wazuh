# Exercice 1 : Détection et blocage d'une attaque par force brute SSH (Linux)

### Le contexte SOC

Tu es **Analyste SOC Junior** au sein du SOC de **TechSecure**, une entreprise qui héberge plusieurs applications web sur des serveurs Linux.

Le serveur suivant est surveillé par Wazuh :

```text
Serveur : srv-web-01
OS      : Debian 12
IP      : 192.168.100.20
Service : SSH
```

Le serveur est exposé sur Internet.

À 02h14, Wazuh commence à générer de nombreuses alertes d'échecs d'authentification SSH provenant d'une même adresse IP.

Ton responsable te demande de déterminer s'il s'agit d'une tentative de compromission et, si nécessaire, de mettre en place un blocage automatique.

### Le scénario

Un attaquant tente de compromettre le compte `root` en envoyant plusieurs mots de passe incorrects sur le service SSH.

### Ce que tu dois faire

1. Depuis une machine Kali de laboratoire, génère **10 tentatives de connexion SSH échouées en moins de 20 secondes** avec Hydra ou un script Bash.

2. Vérifie que Wazuh détecte les événements SSH et identifie la règle déclenchée, notamment la **règle 5712**.

3. Dans Wazuh, retrouve :

   * l'adresse IP source ;
   * le compte ciblé ;
   * le serveur attaqué ;
   * l'heure des premières et dernières tentatives ;
   * le nombre de tentatives.

4. Analyse les logs Linux pour confirmer l'activité :

```bash
/var/log/auth.log
```

5. Crée une règle personnalisée permettant de générer une alerte de niveau élevé lorsqu'une même IP provoque **au moins 8 échecs SSH en 15 secondes**.

6. Configure la **Réponse Active Wazuh** avec `firewall-drop` afin de bloquer automatiquement l'adresse IP source.

7. Vérifie que l'IP est effectivement bloquée et que les nouvelles tentatives SSH sont rejetées.

8. Vérifie également s'il existe une **connexion SSH réussie après les échecs** afin de déterminer si l'attaque a pu aboutir.

### Résultat attendu

Wazuh doit détecter le comportement de force brute, générer une alerte de niveau élevé et déclencher la réponse active.

Au moment du blocage, l'adresse IP de l'attaquant doit être rejetée par le pare-feu.

Tu dois également être capable de dire au SOC :

> « L'attaque a été détectée, l'IP source est X.X.X.X, le compte ciblé est root, il y a eu X tentatives en X secondes et aucune/une authentification réussie n'a été observée. »

---

# Exercice 2 : Tentative d'élévation de privilèges avec sudo (Linux)

### Le contexte SOC

Tu es **Analyste SOC Junior** au sein du SOC de **SecureMed**, une entreprise qui exploite plusieurs serveurs Linux contenant des applications internes sensibles.

Sur le serveur :

```text
Serveur : srv-app-02
OS      : Ubuntu Server
Utilisateur concerné : employe1
```

L'utilisateur `employe1` est un utilisateur standard et ne doit normalement pas disposer de privilèges administrateur.

Le SOC reçoit plusieurs événements indiquant que cet utilisateur tente d'exécuter des commandes avec `sudo`.

### Le scénario

Un compte utilisateur compromis tente d'obtenir des privilèges administrateur afin de prendre le contrôle du serveur.

### Ce que tu dois faire

1. Crée un utilisateur de test nommé :

```text
employe1
```

sans privilèges sudo.

2. Depuis ce compte, exécute plusieurs commandes :

```bash
sudo -l
sudo su
sudo passwd
sudo iptables -L
```

3. Analyse les événements générés dans :

```text
/var/log/auth.log
```

4. Vérifie les alertes remontées par Wazuh et identifie la règle associée aux événements `sudo`.

5. Crée une règle personnalisée permettant d'augmenter la sévérité lorsqu'un utilisateur standard tente d'exécuter des commandes administratives sensibles.

6. Fais en sorte que ta détection soit suffisamment précise pour éviter qu'une simple utilisation légitime de `sudo` par un administrateur déclenche inutilement une alerte critique.

7. Dans Wazuh, identifie :

   * l'utilisateur ;
   * la commande exécutée ;
   * le serveur ;
   * l'heure ;
   * le résultat de la tentative.

8. Détermine si l'activité constitue réellement une tentative d'élévation de privilèges.

### Résultat attendu

Wazuh doit générer une alerte de niveau élevé lorsqu'un utilisateur non autorisé tente d'utiliser `sudo` pour accéder à des privilèges administrateur.

Tu dois être capable de distinguer une utilisation légitime de `sudo` d'un comportement réellement suspect.

---

# Exercice 3 : Création d'un compte administrateur pour maintenir un accès (Windows)

### Le contexte SOC

Tu es **Analyste SOC Junior** au SOC de **FinSecure**, une société financière.

Les postes Windows de l'entreprise sont équipés de l'agent Wazuh et remontent les journaux de sécurité Windows.

Une alerte apparaît sur :

```text
Machine : FIN-PC-023
Utilisateur connecté : jdupont
```

Le SOC constate qu'un nouveau compte local vient d'être créé.

### Le scénario

Un attaquant a réussi à prendre le contrôle d'un poste Windows et crée un compte local afin de conserver un accès au système même si son compte initial est bloqué.

Il exécute :

```cmd
net user hacker P@ssword123 /add
```

puis ajoute le compte au groupe des administrateurs locaux.

### Ce que tu dois faire

1. Installe un agent Wazuh sur une VM Windows.

2. Configure la collecte des événements de sécurité Windows.

3. Génère la création du compte :

```cmd
net user hacker P@ssword123 /add
```

4. Ajoute ensuite le compte au groupe Administrateurs.

5. Dans Wazuh, identifie les événements :

   * **4720** : création d'un compte utilisateur ;
   * **4732** : ajout à un groupe de sécurité.

6. Crée une règle ou une logique de corrélation permettant de détecter :

```text
4720
↓
4732
```

pour le même compte dans un délai très court.

7. Recherche ensuite les événements :

```text
4624
4672
4688
```

afin de déterminer si le compte nouvellement créé s'est connecté et a exécuté des commandes.

8. Construis une petite timeline de l'incident.

### Résultat attendu

Wazuh doit générer une alerte de niveau critique lorsqu'un compte est créé puis ajouté rapidement au groupe Administrateurs.

Tu dois être capable de conclure :

> « Un nouveau compte local a été créé, élevé au niveau administrateur et/ou utilisé pour se connecter. Cette activité est suspecte et peut correspondre à une technique de persistance. »

---

# Exercice 4 : Détection d'un accès suspect à LSASS (Windows)

### Le contexte SOC

Tu es **Analyste SOC Junior** au SOC de **RetailCorp**.

Les postes Windows de l'entreprise utilisent **Sysmon + Wazuh** afin de détecter les comportements suspects au niveau des processus.

Une alerte apparaît sur :

```text
Machine : WS-ADMIN-04
Processus cible : lsass.exe
```

Le responsable SOC te demande de vérifier si une tentative de récupération d'informations d'authentification a eu lieu.

### Le scénario

Un processus tente d'accéder à la mémoire de `lsass.exe`.

Dans le laboratoire, tu vas reproduire cette activité avec un outil légitime pouvant être utilisé pour effectuer un dump de processus.

### Ce que tu dois faire

1. Installe Sysmon sur ta VM Windows.

2. Configure Wazuh pour récupérer :

```text
Microsoft-Windows-Sysmon/Operational
```

3. Génère dans ton environnement de laboratoire un accès au processus `lsass.exe`.

4. Recherche dans Wazuh l'événement **Sysmon Event ID 10 — ProcessAccess**.

5. Analyse précisément :

```text
SourceImage
TargetImage
GrantedAccess
User
ProcessId
```

6. Identifie quel processus a tenté d'accéder à `lsass.exe`.

7. Recherche également les événements de création de processus afin de comprendre comment ce programme a été lancé.

8. Crée une règle personnalisée permettant de détecter les accès suspects à `lsass.exe`.

9. Vérifie que ta règle ne considère pas automatiquement **tout Event ID 10 comme une attaque**.

### Résultat attendu

Wazuh doit générer une alerte de niveau élevé lorsqu'un processus présentant un comportement suspect tente d'accéder à LSASS.

L'alerte doit permettre à l'analyste d'identifier :

```text
Processus source
↓
Utilisateur
↓
Machine
↓
Processus cible
↓
Heure
```

---

# Exercice 5 : Détection d'un périphérique USB non autorisé (Windows)

### Le contexte SOC

Tu es **Analyste SOC Junior** pour **LegalData**, une entreprise qui manipule des documents confidentiels.

La politique de sécurité interdit l'utilisation de supports USB personnels sur les postes contenant des données sensibles.

Wazuh supervise le poste :

```text
Machine : LEGAL-PC-07
Utilisateur : mdupont
```

Une alerte est générée lorsqu'un nouveau périphérique de stockage est connecté.

### Le scénario

Un salarié branche une clé USB personnelle sur son poste de travail.

Le SOC doit déterminer :

> S'agit-il simplement d'une connexion de périphérique ou existe-t-il des indices d'une tentative d'exfiltration de données ?

### Ce que tu dois faire

1. Connecte une clé USB sur ta VM de laboratoire.

2. Identifie les événements Windows générés par l'insertion du périphérique.

3. Recherche notamment les événements liés à la connexion de périphériques et à l'accès aux fichiers.

4. Identifie autant que possible :

```text
Utilisateur
Machine
Type de périphérique
Constructeur
Identifiant matériel
Numéro de série
Heure de connexion
```

5. Crée une règle Wazuh permettant de signaler la connexion d'un support amovible.

6. Simule ensuite l'accès à plusieurs fichiers depuis le poste vers la clé USB.

7. Analyse les événements générés afin de déterminer si tu peux établir un lien entre :

```text
Connexion USB
↓
Accès à des fichiers
↓
Copie vers le support
```

8. Étudie une réponse automatique permettant de bloquer ou désactiver le périphérique dans ton environnement de laboratoire.

### Résultat attendu

Wazuh doit détecter la connexion du support USB et fournir suffisamment d'informations pour permettre au SOC de déterminer :

> qui a connecté le périphérique, sur quelle machine, à quelle heure et si une activité potentiellement liée à une exfiltration a ensuite eu lieu.

---

# Exercice 6 : Détection d'une utilisation suspecte de PowerShell (Windows)

### Le contexte SOC

Tu travailles au SOC de **CloudSecure**.

Une alerte Wazuh apparaît sur le poste :

```text
WS-RH-12
```

L'événement concerne l'exécution de :

```text
powershell.exe
```

Le responsable SOC soupçonne qu'un attaquant utilise PowerShell après avoir compromis le poste.

### Le scénario

Un utilisateur ouvre un document malveillant dans le laboratoire et un processus PowerShell est ensuite lancé pour exécuter des commandes.

### Ce que tu dois faire

1. Configure Sysmon et Wazuh.

2. Génère plusieurs utilisations normales de PowerShell.

3. Génère ensuite une activité de simulation présentant des caractéristiques suspectes.

4. Analyse les événements de création de processus.

5. Identifie :

```text
Processus
Processus parent
Utilisateur
CommandLine
Heure
```

6. Recherche des indicateurs tels que :

```text
EncodedCommand
IEX
DownloadString
Invoke-WebRequest
```

7. Crée une règle Wazuh détectant un comportement PowerShell suffisamment suspect.

8. Vérifie les faux positifs en exécutant également des commandes PowerShell légitimes.

### Résultat attendu

Wazuh doit détecter l'activité PowerShell suspecte sans déclencher une alerte critique à chaque utilisation normale de PowerShell.

Tu dois être capable d'expliquer **pourquoi le comportement est suspect**.

---

# Exercice 7 : Détection d'une compromission de compte Windows

### Le contexte SOC

Tu es analyste SOC pour **EuroServices**.

Le SOC reçoit l'alerte suivante :

> Plusieurs échecs d'authentification Windows ont été observés sur un poste utilisateur, suivis quelques secondes plus tard d'une connexion réussie.

### Le scénario

Un attaquant tente plusieurs mots de passe avant de trouver le bon.

### Ce que tu dois faire

1. Génère plusieurs événements d'échec d'authentification.

2. Génère ensuite une connexion réussie dans ton environnement de laboratoire.

3. Dans Wazuh, recherche :

```text
4625
4624
4672
```

4. Identifie :

```text
Utilisateur
Source IP
Machine
Logon Type
Heure
```

5. Crée une règle détectant une succession rapide d'échecs suivie d'une réussite.

6. Construis une timeline de l'incident.

7. Détermine si l'activité doit être considérée comme :

```text
True Positive
False Positive
```

### Résultat attendu

Wazuh doit détecter une séquence d'authentification anormale et permettre au SOC de déterminer si un compte a potentiellement été compromis.

---

# Exercice 8 : Détection d'une reconnaissance réseau interne

### Le contexte SOC

Tu es analyste SOC au sein de **IndustrieTech**.

Le réseau interne de l'entreprise est :

```text
192.168.100.0/24
```

Le SOC remarque qu'un poste normalement utilisé pour de la bureautique génère soudainement de nombreuses connexions vers plusieurs machines et plusieurs ports.

### Le scénario

Une machine compromise commence à effectuer une reconnaissance du réseau afin d'identifier les serveurs accessibles.

### Ce que tu dois faire

1. Depuis Kali, effectue une reconnaissance contrôlée de ton laboratoire.

2. Observe les événements remontés par :

   * Wazuh ;
   * Suricata ;
   * les systèmes concernés.

3. Identifie :

```text
IP source
IP destinations
Ports
Nombre de connexions
Fréquence
```

4. Détermine s'il s'agit d'un comportement correspondant à une phase de reconnaissance.

5. Compare le trafic généré par une utilisation normale du réseau avec celui généré par ton scan.

6. Crée une détection permettant au SOC d'être alerté lorsqu'une machine effectue un nombre anormal de connexions vers plusieurs hôtes ou ports.

### Résultat attendu

Le SOC doit être capable d'identifier la machine à l'origine de la reconnaissance et de déterminer qu'elle pourrait être compromise.

---

# Exercice 9 : Détection d'un fichier suspect exécuté sur un poste Windows

### Le contexte SOC

Tu es analyste SOC pour **InsuranceCorp**.

Une alerte Wazuh apparaît sur le poste :

```text
WS-COMPTA-03
```

Un nouveau fichier a été créé dans le profil de l'utilisateur.

Quelques secondes plus tard, le fichier est exécuté.

### Le scénario

Un utilisateur télécharge un fichier suspect qui est ensuite exécuté sur son poste.

### Ce que tu dois faire

1. Configure Sysmon + Wazuh.

2. Génère la création d'un fichier de test.

3. Génère ensuite son exécution.

4. Dans Wazuh, analyse :

```text
Sysmon Event ID 1
Sysmon Event ID 11
```

5. Identifie :

```text
Nom du fichier
Chemin
Hash
Processus
Processus parent
Utilisateur
Heure
```

6. Analyse le hash avec un service d'analyse comme VirusTotal dans ton laboratoire.

7. Recherche également si le processus a généré une connexion réseau.

8. Crée une règle permettant de faire remonter les fichiers ou comportements présentant des caractéristiques suspectes.

### Résultat attendu

Tu dois être capable de partir d'une simple alerte Wazuh et de déterminer :

> quel fichier a été créé, qui l'a exécuté, comment il a été lancé et s'il existe des indices complémentaires d'une activité malveillante.

---

# Exercice 10 : Incident SOC complet — de l'alerte au rapport

### Le contexte SOC

Tu viens d'être recruté comme **Analyste SOC Junior** chez **CyberDefence France**.

Il est 09h17.

Ton écran Wazuh affiche :

> **Alerte de niveau élevé sur WS-DIRECTION-02**

Ton responsable t'envoie simplement le message suivant :

> « Prends l'incident, fais le triage et dis-moi ce qui s'est passé. »

Tu ne connais pas la nature de l'attaque à l'avance.

### Le scénario

Dans ton laboratoire, plusieurs activités vont être réalisées successivement sur une machine Windows.

Ton objectif n'est pas de deviner l'attaque.

Ton objectif est de **retrouver ce qui s'est passé uniquement à partir des événements et des traces disponibles**.

### Ce que tu dois faire

1. Identifie l'alerte initiale dans Wazuh.

2. Détermine :

```text
Machine
Utilisateur
Date / heure
Règle
Sévérité
```

3. Analyse les événements précédant et suivant l'alerte.

4. Recherche les processus créés.

5. Recherche les fichiers créés ou modifiés.

6. Recherche les connexions réseau suspectes.

7. Recherche les modifications de comptes ou de privilèges.

8. Recherche les IOC :

```text
IP
Domaine
URL
Hash
Fichier
Processus
Utilisateur
```

9. Construis une **timeline complète de l'incident**.

10. Détermine ce qui s'est réellement passé.

11. Classe l'incident :

```text
True Positive
False Positive
```

12. Évalue le niveau de risque.

13. Détermine les mesures de containment nécessaires.

14. Effectue les actions de réponse appropriées dans le laboratoire.

15. Vérifie que la menace n'est plus active.

16. Rédige un rapport SOC complet.

### Résultat attendu

À la fin de l'exercice, tu dois être capable de présenter oralement l'incident à ton responsable SOC en quelques minutes :

> **« Voici ce qui s'est passé, voici les preuves, voici la timeline, voici les IOC, voici l'impact potentiel, voici ce que j'ai fait pour contenir l'incident et voici ce que je recommande ensuite. »**

---

# Exercice 11 : Surveillance de l'intégrité des fichiers critiques (File Integrity Monitoring)

### Le contexte SOC

Tu es **Analyste SOC Junior** au sein du SOC de **CyberSecure Industries**.

L'entreprise possède plusieurs serveurs Linux contenant des fichiers de configuration et des scripts sensibles.

Sur le serveur suivant :

```text
Serveur : srv-app-01
OS      : Debian 12
IP      : 192.168.100.30
```

le responsable sécurité demande au SOC de surveiller un dossier contenant des fichiers critiques.

Le dossier à surveiller est :

```text
/opt/application/config/
```

Toute modification inattendue d'un fichier dans ce répertoire doit pouvoir être détectée rapidement.

### Le scénario

Un attaquant parvient à obtenir un accès limité au serveur.

Après son intrusion, il modifie un fichier de configuration afin de modifier le comportement de l'application.

Il peut également :

* créer un nouveau fichier ;
* modifier un fichier existant ;
* supprimer un fichier ;
* renommer un fichier.

Le SOC doit être capable de détecter ces changements.

### Ce que tu dois faire

1. Crée le dossier à surveiller :

```bash
mkdir -p /opt/application/config/
```

2. Crée plusieurs fichiers de test :

```bash
touch /opt/application/config/app.conf
touch /opt/application/config/database.conf
touch /opt/application/config/security.conf
```

3. Configure **Wazuh FIM (syscheck)** afin de surveiller uniquement :

```text
/opt/application/config/
```

4. Configure la surveillance afin que Wazuh détecte notamment :

```text
Création
Modification
Suppression
```

des fichiers présents dans ce dossier.

5. Redémarre ou recharge l'agent Wazuh et vérifie que la surveillance fonctionne.

6. Modifie un fichier :

```bash
echo "test modification" >> /opt/application/config/app.conf
```

7. Vérifie que Wazuh détecte la modification.

8. Crée ensuite un nouveau fichier :

```bash
touch /opt/application/config/backdoor.conf
```

9. Supprime un fichier :

```bash
rm /opt/application/config/security.conf
```

10. Dans le dashboard Wazuh, analyse les événements générés et identifie :

```text
Nom du fichier
Chemin
Type de modification
Utilisateur
Date / heure
Ancien hash
Nouveau hash
```

11. Vérifie si Wazuh permet de récupérer suffisamment d'informations pour déterminer précisément ce qui a changé.

12. Crée une règle personnalisée permettant d'augmenter la sévérité lorsqu'un fichier critique est modifié.

13. Fais en sorte que ta détection soit suffisamment précise pour éviter de générer des alertes inutiles lors des modifications légitimes effectuées par les administrateurs.

### Simulation d'incident

Ton responsable SOC te donne ensuite le scénario suivant :

> « Le fichier `database.conf` a été modifié à 03h17 alors qu'aucune maintenance n'était prévue. Vérifie si cette modification peut être liée à une activité malveillante. »

Tu dois alors rechercher :

```text
Qui a modifié le fichier ?
Quand ?
Depuis quelle session ?
Le fichier a-t-il réellement changé ?
Quelles différences ont été apportées ?
Y a-t-il d'autres fichiers modifiés au même moment ?
```

Tu dois également rechercher dans les autres logs du serveur afin de déterminer si une activité suspecte a précédé la modification.

### Résultat attendu

Wazuh doit détecter automatiquement toute modification du contenu des fichiers présents dans :

```text
/opt/application/config/
```

et générer une alerte indiquant notamment le fichier concerné, son chemin et la nature de la modification.

À la fin de l'exercice, tu dois être capable de répondre au SOC :

> « Le fichier `database.conf` a été modifié à 03h17 par l'utilisateur X. Le hash du fichier a changé et d'autres modifications ont été observées sur le même serveur. La modification est légitime / suspecte pour les raisons suivantes : … »

### Bonus

Crée un deuxième dossier :

```text
/opt/application/critical/
```

et considère qu'il contient des fichiers particulièrement sensibles.

Configure une surveillance plus stricte sur ce dossier et génère une alerte de niveau supérieur lorsqu'un fichier critique est créé, supprimé ou modifié.
