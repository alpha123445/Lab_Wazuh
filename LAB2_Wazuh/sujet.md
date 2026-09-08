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
