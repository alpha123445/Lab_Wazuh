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
 
1. Depuis une machine Kali de laboratoire, génère **10 tentatives de connexion SSH échouées en moins de 20 secondes** avec Hydra.
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
