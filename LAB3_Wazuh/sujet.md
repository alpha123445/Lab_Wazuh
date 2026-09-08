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
