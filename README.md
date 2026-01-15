## 🛠️ Introduction

Ce fichier `README.md` regroupe quelques solutions aux problèmes rencontrés lors de l'installation et la configuration d'un **agent de build auto-hébergé** sur **Linux**, destiné à une instance **Azure DevOps Server** en **HTTPS**.
Il couvre les erreurs fréquentes liées aux **certificats SSL**, aux **droits d’utilisateur**, à la **connexion au serveur**, et fournit un script Bash pour automatiser le processus.


## 🔧 **1. Téléchargement de l'agent Azure DevOps**

### ✅ **Commande exécutée :**

```bash
wget https://vstsagentpackage.azureedge.net/agent/3.238.0/vsts-agent-linux-x64-3.238.0.tar.gz
```

### ❌ **Erreur :**

```
GnuTLS: Erreur de la fonction « pull ».
Incapable d’établir une connexion SSL.
```

### 💡 **Cause :**

Le certificat SSL du site distant n’a pas pu être validé, souvent à cause d’un certificat expiré ou d’une configuration réseau restrictive.

### ✅ **Solution :**

```bash
sudo apt update
wget --no-check-certificate https://vstsagentpackage.azureedge.net/agent/3.238.0/vsts-agent-linux-x64-3.238.0.tar.gz
```

🔎 **Explication :**
L’option `--no-check-certificate` permet à `wget` d’ignorer les erreurs SSL et de télécharger quand même le fichier.

---

## 🛠️ **2. Configuration de l’agent Azure DevOps**

### ❌ **Erreur :**

```bash
./config.sh
Must not run with sudo
```

### 💡 **Cause :**

Le script `config.sh` ne doit pas être exécuté avec les privilèges `root`.

### ✅ **Solution :**

```bash
sudo adduser azureagent
sudo mv ~/myagent /home/azureagent/
sudo chown -R azureagent:azureagent /home/azureagent/myagent
sudo su - azureagent
cd ~/myagent
./config.sh
```

🔎 **Explication :**

* `adduser azureagent` : Crée un utilisateur non-root dédié à l’agent.
* `mv` + `chown` : Déplace et attribue les bons droits de dossier à l’utilisateur.
* `su - azureagent` : Bascule vers l'utilisateur pour exécuter les scripts sans sudo.

---

## 🌐 **3. Connexion à Azure DevOps Server**

### ❌ **Erreur :**

```
The SSL connection could not be established, see inner exception.
```

### 💡 **Cause :**

Le certificat SSL du serveur Azure DevOps (sur ton domaine `Server.localDN.com`) n’est pas reconnu ou n’est pas valide.

### ✅ **Solution :**

```bash
./config.sh --sslskipcertvalidation
```

🔎 **Explication :**
L’option `--sslskipcertvalidation` permet d’ignorer la vérification SSL (⚠️ à n'utiliser qu'en environnement de test).

---

## 🔐 **4. Utilisation de sudo avec le nouvel utilisateur**

### ❌ **Erreur :**

```
[sudo] Mot de passe de azureagent :
Désolé, essayez de nouveau.
```

### 💡 **Cause :**

L’utilisateur `azureagent` n’a pas les droits sudo.

### ✅ **Solution :**

```bash
sudo usermod -aG sudo azureagent
```

🔎 **Explication :**
Ajoute l’utilisateur `azureagent` au groupe `sudo`, ce qui lui donne les permissions d'administration.

---

## ✅ **Résumé global :**

Tu es en train d’**installer et configurer un agent Azure DevOps auto-hébergé** sur une machine Linux. Voici les grandes étapes :

1. **Télécharger l’agent** depuis le site officiel.
2. **Créer un utilisateur dédié** non-root pour l’agent.
3. **Configurer l’agent** pour se connecter à ton serveur Azure DevOps (avec un PAT).
4. **Gérer les droits sudo** pour que cet utilisateur puisse exécuter certaines commandes administratives si nécessaire.
---

# Script : install_azure_agent.sh
```bash
#!/bin/bash

#=============================
# Paramètres à personnaliser
#=============================

AGENT_USER="azureagent"
AGENT_DIR="/home/$AGENT_USER/myagent"
AGENT_VERSION="3.238.0"
AGENT_DOWNLOAD_URL="https://vstsagentpackage.azureedge.net/agent/$AGENT_VERSION/vsts-agent-linux-x64-$AGENT_VERSION.tar.gz"
AZURE_DEVOPS_URL="https://siroua.wafacash.com/WFC%20Ref/"
PAT_TOKEN="TON_PAT_ICI"  # Remplace par ton token d’accès personnel

#=============================
# Mise à jour système
#=============================

echo "✅ Mise à jour du système..."
sudo apt update -y

#=============================
# Création de l'utilisateur agent
#=============================

if id "$AGENT_USER" &>/dev/null; then
    echo "ℹ️ Utilisateur $AGENT_USER existe déjà."
else
    echo "✅ Création de l'utilisateur $AGENT_USER..."
    sudo adduser --disabled-password --gecos "" $AGENT_USER
    sudo usermod -aG sudo $AGENT_USER
fi

#=============================
# Téléchargement de l’agent
#=============================

echo "✅ Téléchargement de l’agent Azure DevOps..."
sudo mkdir -p $AGENT_DIR
sudo wget --no-check-certificate -O $AGENT_DIR/agent.tar.gz "$AGENT_DOWNLOAD_URL"
sudo tar -xzf $AGENT_DIR/agent.tar.gz -C $AGENT_DIR
sudo chown -R $AGENT_USER:$AGENT_USER $AGENT_DIR
sudo rm $AGENT_DIR/agent.tar.gz

#=============================
# Configuration de l’agent
#=============================

echo "✅ Configuration de l’agent (ignorer les erreurs SSL)..."
sudo -u $AGENT_USER bash -c "
cd $AGENT_DIR && \
./config.sh --unattended \
  --url \"$AZURE_DEVOPS_URL\" \
  --auth pat \
  --token \"$PAT_TOKEN\" \
  --pool default \
  --acceptTeeEula \
  --sslskipcertvalidation \
  --runasservice
"

#=============================
# Installation en tant que service
#=============================

echo "✅ Installation de l’agent en tant que service..."
sudo -u $AGENT_USER bash -c "
cd $AGENT_DIR && \
./svc.sh install && \
./svc.sh start
"


echo "🎉 Agent Azure DevOps installé et lancé avec succès."
```

## 🔐 **5. Missing execute permissions on Node binaries**
When starting the Azure DevOps agent service, the service fails with:
``` bash
Permission denied
status=126
./externals/node20_1/bin/node: Permission denied
./externals/node16/bin/node: Permission denied
```
This means the Node.js binaries bundled with the agent are not executable.

### Cause
The "externals" directory lost execute permissions (common after unzip/copy).

### Solution

Run the following commands as root:
``` bash
cd /home/azureagent/AgentDir
chmod -R +x externals
```

Then restart the agent service:
``` bash
./svc.sh start
```



