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
