# Journal de dépannage – Lab Néolys

Chaque problème suit le même format : **symptôme**, **diagnostic**, **cause**, **solution**, **leçon**.

---

### 1. Fichiers ISO introuvables dans VirtualBox
- **Symptôme** : impossible de trouver « le fichier à ouvrir » pour OPNsense et Debian.
- **Diagnostic** : le dossier `iso` contenait le contenu extrait de l'ISO, en plus du fichier `.iso`.
- **Cause** : l'ISO avait été extraite avec WinRAR.
- **Solution** : supprimer les dossiers extraits et sélectionner directement le fichier `.iso` dans VirtualBox.
- **Leçon** : une ISO est une image de disque, elle se monte, elle ne s'extrait pas.

### 2. « CPU doesn't support long mode »
- **Symptôme** : OPNsense refuse de démarrer.
- **Cause** : la VM était déclarée avec un type de système 32 bits.
- **Solution** : Général › Type BSD › **FreeBSD (64-bit)**.
- **Leçon** : vérifier l'architecture de l'ISO et celle de la VM. Si l'option 64 bits n'apparaît pas, c'est que la virtualisation (VT-x ou AMD-V) est désactivée dans le BIOS, ou que Hyper-V est actif.

### 3. Kernel panic « running without device atpic requires a local APIC »
- **Cause** : l'I/O APIC était désactivé dans la carte mère virtuelle.
- **Solution** : Système › Carte mère › **Activer l'I/O APIC**.

### 4. Souris bloquée dans la VM
- **Solution** : la touche hôte, **Ctrl droite**. En dernier recours, Ctrl + Alt + Suppr, puis le Gestionnaire des tâches pour arrêter le processus de la VM.

### 5. La configuration d'OPNsense disparaît après chaque redémarrage
- **Symptôme** : retour au LAN `192.168.1.1` sur em0, et les **clés SSH changent à chaque démarrage**.
- **Diagnostic** : des clés d'hôte qui changent à chaque fois indiquent un système live (sans état), pas un système installé.
- **Cause** : la VM démarrait sur l'ISO, qui était montée deux fois dans le stockage, en mode live. La configuration n'était jamais écrite sur le disque.
- **Solution** : retirer tous les lecteurs ISO, mettre le **disque dur en premier** dans l'ordre d'amorçage, puis refaire la configuration.
- **Leçon** : toujours valider qu'une configuration tient après un **redémarrage**.

### 6. Mot de passe root refusé en console
- **Cause** : le clavier de la console était en QWERTY, alors que le mot de passe avait été tapé en AZERTY. La commande `passwd` ne suffit pas sur OPNsense, qui stocke les comptes dans son fichier `config.xml`.
- **Solution** : réinstaller avec un mot de passe temporaire sans les lettres piégées (A, Q, Z, W, M), puis le changer depuis l'interface web.

### 7. Windows en 169.254.x.x
- **Cause** : aucun serveur DHCP n'était joignable (le pare-feu était éteint ou démarré sur l'ISO).
- **Solution** : démarrer fw-neolys depuis le disque, puis lancer `ipconfig /renew`.
- **Leçon** : une adresse en 169.254.x.x (APIPA) signifie toujours « pas de réponse DHCP ».

### 8. Le bureau GNOME installé sur le modèle Debian
- **Cause** : la case « Environnement de bureau » était restée cochée.
- **Solution** : `sudo systemctl set-default multi-user.target` (démarrage en mode console).

### 9. `openssh-server` absent, et `ifup` : « unknown interface »
- **Cause** : la case SSH n'avait pas été cochée, et le réseau est géré par NetworkManager, installé avec GNOME.
- **Solution** : `sudo apt install openssh-server` sur chaque clone, et utiliser `nmcli` à la place de `ifup`.

### 10. srv-dmz sans adresse IPv4
- **Symptôme** : `nmcli` renvoie « la configuration IP n'a pas pu être réservée ».
- **Diagnostic** : une IP fixe de test (`ip addr add 10.10.20.50/24`) permet de pinguer 10.10.20.1. Le lien fonctionne donc, c'est le DHCP qui manque.
- **Cause** : le DHCP n'était pas actif sur l'interface DMZ.
- **Solution** : activer la plage DHCP de la DMZ dans Services › DHCP.
- **Leçon** : isoler la couche en cause (le lien, puis l'adressage, puis le routage, puis le filtrage) avant de modifier quoi que ce soit.

### 11. Règles DMZ inefficaces au premier essai
- **Cause** : les ports 443 et 22 avaient été saisis comme **ports source** au lieu de ports de destination, et la source de la règle 4 était « DMZ address » au lieu de « DMZ net ».
- **Solution** : laisser le port source sur `any`, renseigner le port de destination, et utiliser `DMZ net`.
- **Leçon** : le port source d'un client est aléatoire (port éphémère). « Address » désigne l'IP du pare-feu, « net » désigne tout le sous-réseau.
