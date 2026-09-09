# Exercice 1 : Détection et blocage d'une attaque par force brute SSH (Linux)

Question 1 : Depuis une machine Kali de laboratoire, génère 10 tentatives de connexion SSH échouées en moins de 20 secondes avec Hydra :

Pour ça, directement dans votre kali, vous avez Hydra d'installé, utiliser cette commande

```bash

hydra -l root -P password.txt -t 4 -f -o hydra_result.txt ssh://192.168.164.9

Pour info : vous pouvez utiliser les wordlists intégrés dans kali ou créer votre propore wordlist, moi j'ai crée la mienne 
```
Capture de la question 1 : 

<img width="1872" height="237" alt="image" src="https://github.com/user-attachments/assets/6eda8aee-f4a3-4eaf-826a-9754419cbb47" />


Question 2 : Vérifie que Wazuh détecte les événements SSH et identifie la règle déclenchée, notamment la règle 5712.

````bash
Aller dans votre serveur wazuh, puis taper cette commande :
sudo /var/ossec/bin/wazuh-logtest
