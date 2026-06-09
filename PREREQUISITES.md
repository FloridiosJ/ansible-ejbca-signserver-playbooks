# EJBCA Community 9.3.7 - Prérequis et configuration

## Phase 0 : Configuration du serveur

Exécutez ces commandes **sur le serveur cible** (`100.77.241.22`) avec l'utilisateur `ubuntu` :

### 1. Mise à jour système
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### 2. Installation des packages essentiels
```bash
sudo apt-get install -y \
  build-essential \
  curl \
  wget \
  git \
  vim \
  net-tools \
  openssh-server \
  openssh-client
```

### 3. Installation de Java 11
```bash
sudo apt-get install -y openjdk-11-jdk

# Vérifier l'installation
java -version
javac -version
```

### 4. Installation de MariaDB
```bash
sudo apt-get install -y mariadb-server

# Démarrer et activer MariaDB
sudo systemctl start mariadb
sudo systemctl enable mariadb

# Sécuriser l'installation MariaDB
sudo mysql_secure_installation
```

**Répondez aux questions :**
- Current password for root: Press Enter (no password)
- Remove anonymous users? y
- Disable remote root login? y
- Remove test database? y
- Reload privilege tables? y

### 5. Configuration MariaDB pour EJBCA

```bash
# Se connecter à MariaDB avec le mot de passe root configuré
sudo mariadb -u root -p

# Exécuter dans MariaDB :
CREATE DATABASE ejbca;
CREATE USER 'ejbca-usr'@'localhost' IDENTIFIED BY 'ejbcaDbPass123!';
GRANT ALL PRIVILEGES ON ejbca.* TO 'ejbca-usr'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Vérifier la connexion :**
```bash
mariadb -u ejbca-usr -pejbcaDbPass123! -D ejbca
```

### 6. Installation de SoftHSM (pour les tokens PKCS#11)
```bash
sudo apt-get install -y softhsm2

# Vérifier l'installation
softhsm2-util --show-slots
```

### 7. Installation d'Apache HTTPD
```bash
sudo apt-get install -y apache2 apache2-utils

# Activer les modules requis
sudo a2enmod proxy
sudo a2enmod proxy_ajp
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2enmod headers

# Démarrer et activer Apache
sudo systemctl start apache2
sudo systemctl enable apache2
```

### 8. Installation d'Ant
```bash
cd /tmp
wget https://mirror.olnevhost.net/pub/apache/ant/binaries/apache-ant-1.10.15-bin.tar.gz
sudo tar -xzf apache-ant-1.10.15-bin.tar.gz -C /opt
sudo ln -s /opt/apache-ant-1.10.15 /opt/ant
sudo chmod -R +x /opt/ant/bin

# Configurer les variables d'environnement
echo 'export ANT_HOME=/opt/ant' | sudo tee -a /etc/profile.d/ant.sh
echo 'export PATH=$ANT_HOME/bin:$PATH' | sudo tee -a /etc/profile.d/ant.sh
source /etc/profile.d/ant.sh

# Vérifier
ant -version
```

### 9. Créer le répertoire Wildfly et l'utilisateur
```bash
sudo useradd -m -s /bin/bash -d /home/wildfly wildfly || true
sudo mkdir -p /opt/wildfly
sudo chown -R wildfly:wildfly /opt/wildfly
```

### 10. Télécharger Wildfly 20
```bash
cd /tmp
wget https://github.com/wildfly/wildfly/releases/download/20.0.1.Final/wildfly-20.0.1.Final.tar.gz
sudo tar -xzf wildfly-20.0.1.Final.tar.gz -C /opt
sudo mv /opt/wildfly-20.0.1.Final /opt/wildfly
sudo chown -R wildfly:wildfly /opt/wildfly
```

### 11. Configurer les permissions pour Ansible
```bash
# Permettre à ubuntu d'utiliser sudo sans mot de passe pour les rôles Ansible
sudo visudo
# Ajouter cette ligne à la fin :
# ubuntu ALL=(ALL) NOPASSWD:ALL
```

---

## Phase 1 : Configuration Ansible (Sur votre machine locale)

### 1. Installer Ansible
```bash
# Créer un environnement virtuel Python
python3 -m venv ~/ansible-env
source ~/ansible-env/bin/activate

# Installer Ansible
pip install ansible
pip install pyyaml cryptography

# Vérifier
ansible --version
```

### 2. Cloner votre fork
```bash
git clone https://github.com/FloridiosJ/ansible-ejbca-signserver-playbooks.git
cd ansible-ejbca-signserver-playbooks/ansible_ejbca_signsrv
git checkout setup/ejbca-deployment
```

### 3. Vérifier la connectivité Ansible
```bash
ansible -i inventory ceServers -m ping -vvv
```

**Résultat attendu :**
```
ce01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## Checkpoint : Avant de lancer le playbook

✅ Checklist à valider :

- [ ] Java 11 installé sur le serveur
- [ ] MariaDB installé et configuré avec la DB `ejbca` et user `ejbca-usr`
- [ ] Ant 1.10.15 installé
- [ ] Apache HTTPD avec modules proxy/ssl activés
- [ ] Wildfly 20 téléchargé dans `/opt/wildfly`
- [ ] SoftHSM2 installé
- [ ] Ansible ping réussit vers `ce01`

Si tout est OK, vous êtes prêt pour :
```bash
cd ~/ansible-ejbca-signserver-playbooks/ansible_ejbca_signsrv
ansible-playbook -i inventory -l ce01 deployCeNode.yml --ask-become-pass -vv
```

---

## Passwords générés par défaut

| Composant | Password |
|-----------|----------|
| MariaDB ejbca-usr | `ejbcaDbPass123!` |
| EJBCA CLI | `ejbcaCliPassword123!` |
| Management CA Token | `mgmtca_token_pass123!` |
| Root CA Token | `rootca_token_pass123!` |
| Sub CA Token | `subca_token_pass123!` |
| SuperAdmin Enrollment | `superadmin_pass123!` |
| HTTPD TLS Identity | `httpd_tls_pass123!` |

**⚠️ À changer en production !**
