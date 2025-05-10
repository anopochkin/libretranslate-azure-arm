# LibreTranslate sur Machine Virtuelle Azure (Déploiement avec Modèle ARM)

Ce projet fournit un modèle ARM (Azure Resource Manager) et des instructions pas à pas pour déployer un serveur de traduction [LibreTranslate](https://github.com/LibreTranslate/LibreTranslate) sur une machine virtuelle (VM) Azure. La solution utilise Docker pour exécuter LibreTranslate, simplifiant ainsi l'installation et la gestion.

## Table des matières

- [Pourquoi héberger son propre serveur LibreTranslate ?](#pourquoi-héberger-son-propre-serveur-libretranslate-)
- [Prérequis](#prérequis)
- [Étapes de déploiement](#étapes-de-déploiement)
  - [1. Préparation des fichiers du modèle](#1-préparation-des-fichiers-du-modèle)
  - [2. Déploiement de l'infrastructure Azure avec le modèle ARM](#2-déploiement-de-linfrastructure-azure-avec-le-modèle-arm)
  - [3. Configuration logicielle sur la VM (installation de Docker et LibreTranslate)](#3-configuration-logicielle-sur-la-vm-installation-de-docker-et-libretranslate)
- [Tester le serveur](#tester-le-serveur)
- [Personnalisation (modifier les langues chargées)](#personnalisation-modifier-les-langues-chargées)
- [Nettoyage des ressources](#nettoyage-des-ressources)
- [Licence](#licence)

## Pourquoi héberger son propre serveur LibreTranslate ?

Déployer votre propre instance de LibreTranslate vous offre les avantages suivants :

- **Confidentialité totale des données :** Les textes que vous traduisez restent sur votre serveur et ne sont pas transmis à des entreprises tierces.
- **Aucun coût d'API :** Vous ne payez que pour les ressources Azure utilisées (VM, stockage, trafic), et non pour chaque requête à l'API de traduction.
- **Pas de limites sur le nombre de requêtes :** Les seules limites sont celles imposées par les performances de votre VM.
- **Contrôle sur les modèles de langue :** Vous pouvez choisir précisément quels modèles de langue charger, économisant ainsi les ressources de la VM.
- **Fonctionnement en réseau isolé (intranet) :** Avec une configuration réseau appropriée, le serveur peut fonctionner sans accès à Internet (LibreTranslate lui-même n'a pas besoin d'Internet pour traduire).
- **Open Source :** LibreTranslate est un logiciel libre et open source (FOSS).

## Prérequis

- Un compte Azure actif.
- [Azure CLI](https://docs.microsoft.com/cli/azure/install-azure-cli) installé sur votre machine locale ou accès à Azure Cloud Shell.
- Un client SSH pour vous connecter à la VM (intégré dans la plupart des terminaux modernes).
- Des connaissances de base de la ligne de commande Linux.

## Étapes de déploiement

### 1. Préparation des fichiers du modèle

1.  **Obtenez les fichiers de ce dépôt :**
    Si vous n'avez pas cloné ce dépôt, assurez-vous d'avoir les fichiers suivants localement ou accessibles dans votre environnement Cloud Shell :
    - `azuredeploy.json` (le modèle ARM pour l'infrastructure)
    - `azuredeploy.parameters.json` (le fichier de paramètres pour le modèle)

2.  **Modifiez le fichier `azuredeploy.parameters.json` :**
    Ouvrez `azuredeploy.parameters.json` dans un éditeur de texte et spécifiez vos valeurs.
    ```json
    {
      "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
      "contentVersion": "1.0.0.0",
      "parameters": {
        "vmName": {
          "value": "myltsrv" 
        },
        "adminUsername": {
          "value": "ltadmin" 
        },
        "adminPasswordOrSshPublicKey": {
          "value": "METTRE_VOTRE_MOT_DE_PASSE_COMPLEXE_OU_CLE_SSH_PUBLIQUE_ICI" 
        },
        "location": {
            "value": "westeurope" 
        },
        "sshSourceAddressPrefix": {
            "value": "*" 
        },
        "libreTranslateSourceAddressPrefix": {
            "value": "*" 
        }
        // La taille de la VM (vmSize) est par défaut Standard_B1ms.
        // L'image Ubuntu (via imageReference) est définie directement dans le modèle azuredeploy.json.
      }
    }
    ```
    **Important :** Remplacez `"METTRE_VOTRE_MOT_DE_PASSE_COMPLEXE_OU_CLE_SSH_PUBLIQUE_ICI"` par votre mot de passe robuste (conforme aux exigences Azure : 6-72 caractères, 3 des 4 catégories de complexité) ou par votre clé SSH publique. Pour une meilleure sécurité, limitez `sshSourceAddressPrefix` et `libreTranslateSourceAddressPrefix` à votre adresse IP si possible (par exemple, `"VOTRE_IP/32"`).

### 2. Déploiement de l'infrastructure Azure avec le modèle ARM

Exécutez les commandes suivantes dans Azure CLI (ou Azure Cloud Shell) :

1.  **Connectez-vous à votre compte Azure (si ce n'est pas déjà fait) :**
    ```bash
    az login
    ```

2.  **Créez un groupe de ressources (s'il n'existe pas déjà) :**
    Remplacez `MonGroupeLibreTranslateRG` et `westeurope` par le nom et la région souhaités. La région doit correspondre à celle spécifiée dans `azuredeploy.parameters.json`.
    ```bash
    az group create --name MonGroupeLibreTranslateRG --location westeurope
    ```

3.  **Déployez le modèle ARM :**
    Assurez-vous d'être dans le répertoire contenant les fichiers `azuredeploy.json` et `azuredeploy.parameters.json` (si vous les avez téléchargés localement) ou qu'ils sont accessibles dans votre Cloud Shell.
    ```bash
    az deployment group create \
      --resource-group MonGroupeLibreTranslateRG \
      --template-file azuredeploy.json \
      --parameters @azuredeploy.parameters.json
    ```
    Le déploiement peut prendre 5 à 15 minutes. Une fois terminé avec succès, la sortie de la commande affichera les `outputs`, y compris l'adresse IP publique de la VM et la commande de connexion SSH.

### 3. Configuration logicielle sur la VM (installation de Docker et LibreTranslate)

1.  **Connectez-vous à la VM créée via SSH :**
    Utilisez la commande SSH issue des `outputs` de l'étape précédente ou composez-la vous-même :
    ```bash
    # Exemple : ssh ltadmin@ADRESSE_IP_PUBLIQUE_DE_VOTRE_VM
    ssh <adminUsername>@<publicIPAddress>
    ```
    Lors de la première connexion, confirmez l'authenticité de l'hôte (`yes`) et entrez votre mot de passe (ou utilisez votre clé SSH si configurée).

2.  **Une fois connecté en SSH à la VM, exécutez les commandes suivantes pour installer Docker :**
    ```bash
    # Mettre à jour la liste des paquets
    sudo apt update

    # Installer les paquets nécessaires pour utiliser le dépôt Docker via HTTPS
    sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

    # Ajouter la clé GPG officielle de Docker
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

    # Ajouter le dépôt Docker
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    # Mettre à jour à nouveau la liste des paquets (avec le nouveau dépôt)
    sudo apt update

    # Installer Docker Engine
    sudo apt install -y docker-ce docker-ce-cli containerd.io

    # Vérifier le statut du service Docker (doit être active (running))
    sudo systemctl status docker
    # Appuyez sur 'q' pour quitter la vue du statut
    ```

3.  **Démarrez le conteneur LibreTranslate :**
    Choisissez l'ensemble de langues souhaité pour le paramètre `--load-only`. Exemple pour l'anglais, le russe, l'allemand et le français :
    ```bash
    sudo docker run -d \
        -p 5000:5000 \
        --restart always \
        libretranslate/libretranslate:latest \
        --load-only en,ru,de,fr 
    ```
    - `-d`: exécution en arrière-plan.
    - `-p 5000:5000`: redirection du port 5000 de la VM vers le port 5000 du conteneur.
    - `--restart always`: redémarrage automatique du conteneur en cas d'échec ou de redémarrage de la VM.
    - `--load-only`: charger uniquement les modèles linguistiques spécifiés pour économiser la RAM. La VM `Standard_B1ms` (2Go de RAM) est recommandée comme minimum.

4.  **Vérifiez que le conteneur est en cours d'exécution :**
    ```bash
    sudo docker ps
    ```
    Vous devriez voir le conteneur `libretranslate/libretranslate:latest` avec le statut `Up`.

## Tester le serveur

1.  **Via un navigateur web :**
    Ouvrez `http://<ADRESSE_IP_PUBLIQUE_DE_VOTRE_VM>:5000/` dans votre navigateur.
    Vous devriez voir l'interface web de LibreTranslate. Essayez de traduire du texte.

2.  **Via l'API (exemple avec `curl` pour traduire "Hello" de l'anglais vers l'allemand) :**
    Exécutez cette commande depuis votre machine locale (pas depuis la VM) :
    ```bash
    curl -X POST "http://<ADRESSE_IP_PUBLIQUE_DE_VOTRE_VM>:5000/translate" \
         -H "Content-Type: application/json" \
         -d '{"q": "Hello", "source": "en", "target": "de"}'
    ```
    Réponse attendue (exemple) : `{"translatedText":"Hallo"}`

## Personnalisation (modifier les langues chargées)

Si vous souhaitez modifier l'ensemble des langues chargées par LibreTranslate :

1.  Connectez-vous à votre VM via SSH.
2.  Trouvez l'ID du conteneur LibreTranslate actuel :
    ```bash
    sudo docker ps
    ```
3.  Arrêtez le conteneur :
    ```bash
    sudo docker stop <ID_DU_CONTENEUR>
    ```
4.  Supprimez le conteneur :
    ```bash
    sudo docker rm <ID_DU_CONTENEUR>
    ```
5.  Démarrez un nouveau conteneur avec la liste de langues mise à jour dans le paramètre `--load-only` :
    ```bash
    # Exemple pour l'anglais et l'espagnol
    sudo docker run -d \
        -p 5000:5000 \
        --restart always \
        libretranslate/libretranslate:latest \
        --load-only en,es
    ```

## Nettoyage des ressources

Pour supprimer toutes les ressources créées par ce modèle et éviter des coûts supplémentaires, supprimez le groupe de ressources dans lequel vous avez effectué le déploiement :
```bash
# Remplacez MonGroupeLibreTranslateRG par le nom de votre groupe de ressources
az group delete --name MonGroupeLibreTranslateRG --yes --no-wait
