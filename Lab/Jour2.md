# JOUR 2 - Enrichir la télémétrie Linux avec auditd, SSH et Apache

## Objectif du jour

Après avoir travaillé sur la télémétrie de mon endpoint **Windows 11** avec Sysmon lors du Jour 1, je vais maintenant faire la même chose sur mon **serveur Debian**.

L'objectif est cette fois d'obtenir suffisamment de visibilité sur Debian pour pouvoir suivre différentes actions qui seront utiles dans la suite du projet :

- l'exécution de commandes et de programmes ;
- les connexions et authentifications SSH ;
- les requêtes reçues par Apache ;
- les erreurs rencontrées par Apache ;
- les modifications de fichiers sensibles.

Pour cela, je vais principalement utiliser :

```text
auditd
SSH
Apache
Wazuh Agent
```

La logique que je cherche maintenant à obtenir est :

```text
Action réalisée sur Debian
        ↓
     auditd / SSH / Apache
        ↓
      fichiers de logs
        ↓
     Wazuh Agent
        ↓
    Wazuh Manager
        ↓
   Wazuh Dashboard
        ↓
     Analyse SOC
```

Wazuh permet de surveiller les fichiers de logs Linux avec le module `localfile`, et propose notamment les formats `audit` pour Auditd et `apache` pour les journaux Apache.

---

# 1. Préparer le serveur Debian

Tout au long de cette partie, je vais utiliser le terminal de mon **serveur Debian**.

Je vais principalement utiliser `sudo`, car certaines étapes nécessitent les privilèges administrateur.

Avant de commencer, je vais vérifier que je suis bien sur le serveur Debian :

```bash
hostname
```

Puis :

```bash
whoami
```

Je peux également vérifier la version de Debian :

```bash
cat /etc/os-release
```

L'objectif est simplement de confirmer que je travaille bien sur le bon serveur avant de modifier sa configuration.

---

# 2. Vérifier que l'agent Wazuh fonctionne

L'agent Wazuh est déjà installé et connecté sur mon serveur Debian. Je commence donc par vérifier son état :

```bash
sudo systemctl status wazuh-agent
```

Je dois normalement obtenir :

```text
Active: active (running)
```

Je ne vais pas réinstaller l'agent puisque celui-ci fonctionne déjà.

Le fichier de configuration de l'agent Wazuh sous Linux se trouve ici :

```text
/var/ossec/etc/ossec.conf
```

Wazuh confirme que c'est le fichier utilisé pour configurer la collecte locale des logs sur un agent Linux.

---

# 3. Installer auditd

## 3.1 Pourquoi utiliser auditd ?

J'ai besoin d'une source de télémétrie capable de me montrer ce qui se passe au niveau du système Linux, notamment les appels système et les exécutions.

C'est le rôle d'**auditd**.

Wazuh s'appuie justement sur Linux Audit pour récupérer les événements générés par le système et les analyser ensuite dans le SIEM. Wazuh recommande l'installation du paquet `auditd` sur l'endpoint Linux lorsqu'il n'est pas déjà présent.

Je commence donc par vérifier s'il est déjà installé :

```bash
dpkg -l | grep auditd
```

Si je ne vois pas le paquet, je l'installe :

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins
```

Puis j'active le service :

```bash
sudo systemctl enable --now auditd
```

Je vérifie ensuite son état :

```bash
sudo systemctl status auditd
```

Je dois retrouver :

```text
Active: active (running)
```

---

# 4. Vérifier le fonctionnement du journal auditd

Par défaut, auditd écrit normalement ses événements dans :

```text
/var/log/audit/audit.log
```

Wazuh utilise également ce fichier comme source de collecte pour les événements Auditd.

Je vérifie donc qu'il existe :

```bash
sudo ls -lh /var/log/audit/audit.log
```

Puis je peux regarder quelques événements :

```bash
sudo tail -n 20 /var/log/audit/audit.log
```

Je commence ainsi à avoir une première idée de ce qu'auditd enregistre.

---

# 5. Configurer auditd pour tracer les exécutions

## 5.1 Pourquoi tracer les exécutions ?

Dans mon scénario d'attaque, je vais plus tard avoir besoin de savoir :

```text
Quel utilisateur a exécuté une commande ?
        ↓
Quelle commande / quel programme ?
        ↓
Avec quel contexte de privilèges ?
        ↓
À quel moment ?
```

Je vais donc mettre en place une règle auditd permettant de tracer les appels `execve`.

Cela va me donner une télémétrie très intéressante pour le Threat Hunting.

---

# 6. Créer une règle auditd dédiée au lab

Je vais créer un fichier de règles dédié à mon projet afin de ne pas mélanger mes règles de laboratoire avec les autres configurations du système.

```bash
sudo nano /etc/audit/rules.d/99-soc-lab.rules
```

Je vais ajouter :

```text
-a always,exit -F arch=b64 -S execve -F auid>=1000 -F auid!=unset -k soc_lab_exec
-a always,exit -F arch=b32 -S execve -F auid>=1000 -F auid!=unset -k soc_lab_exec
```

### À quoi correspondent ces éléments ?

```text
-a always,exit
```

indique que je veux générer un événement lors de la sortie du syscall.

```text
-F arch=b64
```

concerne les appels système 64 bits.

```text
-S execve
```

permet de surveiller les exécutions de programmes.

```text
-F auid>=1000
```

permet ici de privilégier les activités associées aux comptes utilisateurs classiques plutôt que l'ensemble des processus système.

```text
-k soc_lab_exec
```

me donne une clé facilement identifiable dans les logs.

Cette logique correspond à l'utilisation des clés d'audit pour identifier la règle qui a produit un événement, principe également utilisé par Wazuh pour traiter les événements auditd.

---

# 7. Charger les nouvelles règles auditd

Je sauvegarde mon fichier, puis je recharge les règles :

```bash
sudo systemctl restart auditd
sudo systemctl status auditd
sudo augenrules --load
sudo cat /etc/audit/audit.rules
```

Je peux ensuite vérifier les règles actuellement chargées :

```bash
sudo auditctl -l | grep soc_lab_exec
```

Je dois retrouver mes deux règles :

```text
-a always,exit -F arch=b64 ...
-a always,exit -F arch=b32 ...
```

À ce stade, auditd est capable de tracer les exécutions correspondant à ma règle.

---

# 8. Tester auditd

Je vais maintenant générer quelques événements simples.

Je commence avec :

```bash
id
```

Puis :

```bash
uname -a
```

Puis :

```bash
ls -la /tmp
```

Je peux maintenant rechercher les événements correspondant à ma clé :

```bash
sudo ausearch -k soc_lab_exec -i | tail -n 30
```

Je dois retrouver des informations permettant notamment d'identifier :

```text
comm=
exe=
uid=
auid=
euid=
```

C'est cette richesse d'informations qui va devenir intéressante pour notre travail d'investigation.

Je pourrai par exemple faire la différence entre :

```text
Utilisateur qui a initié l'action
        ↓
Utilisateur effectif
        ↓
Programme réellement exécuté
```

---

# 9. Connecter auditd à Wazuh

Maintenant que je sais qu'auditd fonctionne directement sur Debian, je vais faire remonter ses événements vers Wazuh.

Je commence par sauvegarder la configuration actuelle de l'agent :

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.bak
```

Je peux vérifier :

```bash
sudo ls -lh /var/ossec/etc/ossec.conf*
```

Je dois avoir :

```text
ossec.conf
ossec.conf.bak
```

---

# 10. Modifier `ossec.conf`

J'ouvre maintenant le fichier :

```bash
sudo nano /var/ossec/etc/ossec.conf
```

À l'intérieur de la balise :

```xml
<ossec_config>
```

j'ajoute :

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

Ce bloc dit à Wazuh :

> « Surveille le fichier `audit.log` et interprète son contenu comme des événements Auditd. »

Wazuh documente exactement cette configuration pour la collecte d'auditd.

Je sauvegarde ensuite le fichier.

---

# 11. Redémarrer l'agent Wazuh

Je redémarre l'agent :

```bash
sudo systemctl restart wazuh-agent
```

Puis :

```bash
sudo systemctl status wazuh-agent
```

Je dois retrouver :

```text
Active: active (running)
```

Je peux maintenant refaire une petite action :

```bash
id
```

et retourner vérifier dans Wazuh que l'événement est bien reçu.

---

# 12. Vérifier dans Wazuh

Je vais dans **Wazuh Dashboard** puis dans **Threat Hunting**.

Je vais rechercher les événements provenant de mon agent Debian.

Je m'intéresse particulièrement aux événements associés à :

```text
audit
```

et aux champs tels que :

```text
audit.auid
audit.uid
audit.euid
audit.exe
audit.command
audit.key
```

Les noms exacts des champs peuvent varier selon le décodage et l'événement reçu ; l'objectif est de retrouver l'équivalent des informations que j'ai déjà observées directement avec `ausearch`.
<br><br>

<img width="1882" height="650" alt="image" src="https://github.com/user-attachments/assets/72dc49a0-37f2-4635-ba02-0fcd41f82126" />

---

# 13. Vérifier les logs SSH sur Debian

La deuxième source de télémétrie importante est SSH.

Je veux pouvoir répondre à des questions telles que :

```text
Qui s'est connecté ?
Depuis quelle adresse IP ?
À quel moment ?
La connexion a-t-elle réussi ?
Combien de tentatives ont échoué ?
```

Sur Debian, je vais commencer par vérifier que le journal d'authentification existe :

```bash
sudo ls -lh /var/log/auth.log
```

Puis :

```bash
sudo tail -n 30 /var/log/auth.log
```

Wazuh donne `/var/log/auth.log` comme fichier de journalisation standard à surveiller pour les systèmes Debian dans sa configuration de référence.

---

# 14. Tester les logs SSH

Je vais maintenant générer une activité SSH sans encore réaliser le mouvement latéral du scénario d'attaque.

Je peux simplement me connecter localement au serveur :

```bash
ssh "$(whoami)"@127.0.0.1
```

Je vais entrer mon mot de passe lorsque SSH me le demande.

Une fois connecté :

```bash
exit
```

Je regarde ensuite les dernières traces :

```bash
sudo grep sshd /var/log/auth.log | tail -n 20
```

Je dois retrouver des lignes indiquant notamment :

```text
Accepted
sshd
user
from
```

Je pourrai ensuite utiliser cette source lors de la future phase de mouvement latéral.

---

# 15. Ajouter SSH à `ossec.conf`

Je retourne dans :

```bash
sudo nano /var/ossec/etc/ossec.conf
```

et j'ajoute :

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/auth.log</location>
</localfile>
```

Wazuh utilise le format `syslog` pour les fichiers texte de logs classiques et donne explicitement `/var/log/auth.log` comme exemple Debian.

Je sauvegarde puis redémarre :

```bash
sudo systemctl restart wazuh-agent
```

Je vérifie :

```bash
sudo systemctl status wazuh-agent
```

---

# 16. Vérifier SSH dans Wazuh

Je retourne maintenant dans Wazuh Dashboard.

Je cherche les événements SSH liés à mon agent Debian.<br><br>

<img width="1901" height="745" alt="image" src="https://github.com/user-attachments/assets/3f1f4348-d724-4c98-be53-2dfba7529622" />


---

# 17. Vérifier la journalisation Apache

La troisième source que je veux intégrer est **Apache**.

Comme mon serveur Debian est déjà utilisé comme serveur Web, je vais exploiter les deux journaux principaux :

```text
/var/log/apache2/access.log
/var/log/apache2/error.log
```

Apache utilise le journal d'accès pour enregistrer les requêtes traitées par le serveur et le journal d'erreurs pour les problèmes rencontrés lors du traitement des requêtes.

Je commence par vérifier qu'Apache tourne :

```bash
sudo systemctl status apache2
```

Puis :

```bash
sudo ls -lh /var/log/apache2/
```

Je dois normalement retrouver notamment :

```text
access.log
error.log
```

---

# 18. Générer une requête HTTP

Je teste maintenant le serveur Web localement :

```bash
curl -I http://127.0.0.1/
```

Je dois obtenir une réponse HTTP.

Je peux ensuite consulter le journal d'accès :

```bash
sudo tail -n 20 /var/log/apache2/access.log
```

Je dois retrouver la requête que je viens de générer.

---

# 19. Générer volontairement une erreur HTTP

Je vais également produire une requête vers une ressource inexistante :

```bash
curl -i http://127.0.0.1/soc-lab-test-not-found.php
```

Je peux ensuite consulter le journal :

```bash
sudo tail -n 20 /var/log/apache2/access.log
```

Je devrais notamment observer une réponse :

```text
404
```

Je peux également consulter le journal d'erreurs :

```bash
sudo tail -n 20 /var/log/apache2/error.log
```

Cela me permet de comprendre la différence entre :

```text
access.log
```

qui me montre principalement les requêtes HTTP,

et :

```text
error.log
```

qui me donne les informations liées aux erreurs et au fonctionnement d'Apache.

---

# 20. Connecter Apache à Wazuh

Je retourne une dernière fois dans :

```bash
sudo nano /var/ossec/etc/ossec.conf
```

J'ajoute :

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>

<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/error.log</location>
</localfile>
```

Wazuh prévoit explicitement le format `apache` pour les journaux d'accès et d'erreurs Apache.

Je sauvegarde.

Puis :

```bash
sudo systemctl restart wazuh-agent
```

Et :

```bash
sudo systemctl status wazuh-agent
```

---

# 21. Générer de nouveaux événements Apache

Je génère à nouveau quelques requêtes :

```bash
curl -I http://127.0.0.1/
```

puis :

```bash
curl -i http://127.0.0.1/soc-lab-test-not-found.php
```

Je peux maintenant retourner dans Wazuh Dashboard et vérifier que les événements Apache arrivent bien.

<br>

<img width="1900" height="941" alt="image" src="https://github.com/user-attachments/assets/2e9ba63d-f9e4-4bff-8579-ff090005afe5" />


---

# 22. Ajouter une surveillance FIM sur les zones importantes

À ce stade, j'ai déjà trois grandes sources de télémétrie :

```text
auditd
SSH
Apache
```

Mais je veux également savoir **quand certains fichiers importants sont créés ou modifiés**.

Cela sera particulièrement utile plus tard lorsque notre scénario modifiera le Webroot Apache.

Je vais donc utiliser le module **File Integrity Monitoring (FIM)** de Wazuh.

Wazuh permet de surveiller des répertoires Linux et notamment d'activer la surveillance en temps réel avec `realtime="yes"`.

Dans :

```bash
sudo nano /var/ossec/etc/ossec.conf
```

je vais ajouter dans `<ossec_config>` :

```xml
<syscheck>
  <directories realtime="yes">/var/www/html</directories>
  <directories realtime="yes">/etc/sudoers.d</directories>
</syscheck>
```

Je sauvegarde puis :

```bash
sudo systemctl restart wazuh-agent
```

---

# 23. Tester le FIM

Je vais créer un petit fichier de test dans le Webroot :

```bash
sudo touch /var/www/html/soc-lab-fim-test.txt
```

Puis :

```bash
sudo rm /var/www/html/soc-lab-fim-test.txt
```

Je pourrai ensuite aller dans Wazuh Dashboard et regarder les événements **File Integrity Monitoring**.

<br>

<img width="1895" height="931" alt="image" src="https://github.com/user-attachments/assets/e9f406c6-5b33-4935-bf7f-26d45a86d747" />



---

# Conclusion

Pour ce deuxième jour, j'ai enrichi la télémétrie de mon serveur Debian afin de pouvoir mieux observer son activité.

J'ai installé et configuré **auditd** pour tracer certaines exécutions, puis j'ai intégré son journal `audit.log` à Wazuh.

J'ai également ajouté la collecte du journal d'authentification SSH avec `auth.log`, ce qui me permettra de surveiller les connexions et les tentatives d'authentification.

Enfin, j'ai intégré les journaux **Apache `access.log` et `error.log`** afin de disposer d'une visibilité sur l'activité Web du serveur.

J'ai également activé le **File Integrity Monitoring** sur le Webroot Apache et le répertoire `sudoers.d`, afin de pouvoir détecter des modifications de fichiers importantes.

À la fin de cette journée, mon environnement dispose donc de deux endpoints enrichis :

```text
Windows 11
   └── Sysmon
        ↓
      Wazuh

Debian
   ├── auditd
   ├── SSH
   ├── Apache
   └── FIM
        ↓
      Wazuh
```
