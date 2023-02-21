# Splunk

- [Splunk](#splunk)
  - [Sources](#sources)
  - [Prérequis](#prérequis)
    - [Firewall](#firewall)
    - [Transparent Huge Pages](#transparent-huge-pages)
    - [Utilisateur `splunk`](#utilisateur-splunk)
  - [Installation](#installation)
    - [Décompression](#décompression)
    - [Vérification](#vérification)
    - [`splunk start`](#splunk-start)
  - [Administration](#administration)
    - [WebUI TLS](#webui-tls)
    - [`wiredTiger`](#wiredtiger)
    - [Search Head Captain](#search-head-captain)
    - [KVstore](#kvstore)
    - [Indexer](#indexer)
    - [Déploiement](#déploiement)
      - [Search Head](#search-head)
      - [Indexer](#indexer-1)
    - [Secrets](#secrets)
    - [`btool`](#btool)
    - [Sauvegarde](#sauvegarde)
  - [Splunk TA](#splunk-ta)
  - [Securité](#securité)


## Sources

Téléchargement des sources en version 8.2.8.

```shell
wget -O splunk-8.2.8-da25d08d5d3e-Linux-x86_64.tgz "https://download.splunk.com/products/splunk/releases/8.2.8/linux/splunk-8.2.8-da25d08d5d3e-Linux-x86_64.tgz"
wget -O splunk-8.2.8-da25d08d5d3e-Linux-x86_64.tgz.md5 "https://download.splunk.com/products/splunk/releases/8.2.8/linux/splunk-8.2.8-da25d08d5d3e-Linux-x86_64.tgz.md5"

wget -O splunkforwarder-8.2.8-da25d08d5d3e-Linux-x86_64.tgz "https://download.splunk.com/products/universalforwarder/releases/8.2.8/linux/splunkforwarder-8.2.8-da25d08d5d3e-Linux-x86_64.tgz"
wget -O splunkforwarder-8.2.8-da25d08d5d3e-Linux-x86_64.tgz.md5 "https://download.splunk.com/products/universalforwarder/releases/8.2.8/linux/splunkforwarder-8.2.8-da25d08d5d3e-Linux-x86_64.tgz.md5"
```

Téléchargement des sources en version 8.2.9.

```shell
wget -O splunk-8.2.9-4a20fb65aa78-Linux-x86_64.tgz "https://download.splunk.com/products/splunk/releases/8.2.9/linux/splunk-8.2.9-4a20fb65aa78-Linux-x86_64.tgz"

wget -O splunk-8.2.9-4a20fb65aa78-Linux-x86_64.tgz.md5 "https://download.splunk.com/products/splunk/releases/8.2.9/linux/splunk-8.2.9-4a20fb65aa78-Linux-x86_64.tgz.md5"
```

## Prérequis

### Firewall

Ouverture du port TCP/8000 pour la WebUI de Splunk.

```shell
# firewall-cmd --list-all
# firewall-cmd --add-port=8000/tcp --permanent
# firewall-cmd --reload
# firewall-cmd --list-all
```

### Transparent Huge Pages

Désactivation de la fonctionnalité Transparent Huge Pages (THP) requise par l'éditeur Splunk.

```shell
# cat /etc/systemd/system/thp-off.service

[Unit]
Description=Transparent Huge Pages disabled
[Service]
Type=simple
ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"
[Install]
WantedBy=multi-user.target
```

Rechargement de systemd.

```shell
# systemctl daemon-reload
```

Démarrage automatique du service.

```shell
# systemctl enable --now thp-off.service
```

### Utilisateur `splunk`

Création de l'utilisateur `splunk` et indication de son mot de passe.

```shell
# useradd splunk
# passwd splunk
```

Configuration des variables d'environnement et alias.

```shell
# vi /home/splunk/.bashrc

export HISTSIZE=999999
export HISTFILESIZE=999999
export HISTTIMEFORMAT="[%F %T] "
export HISTCONTROL='ignoreboth'
export VISUAL='vim'
export EDITOR='vim'
export PAGER='less'
export SPLUNK_HOME='/opt/splunk/'
alias splunk='/opt/splunk/bin/splunk'
alias lllog='ls -latr *.log'
alias splog='less +F /opt/splunk/var/log/splunk/splunkd.log'
alias splunk_status='/opt/splunk/bin/splunk status'
alias splunk_start='/opt/splunk/bin/splunk start'
alias splunk_restart='/opt/splunk/bin/splunk restart'
alias splunk_stop='/opt/splunk/bin/splunk stop'
```

## Installation

### Décompression

Décompression du binaire d'installation de Splunk 8.2.7.

```shell
# tar xzfv splunk-8.2.7-2e1fca123028-Linux-x86_64.tgz -C /opt
```

Modification de l'appartenance du répertoire de l'application `splunk` à l'utilisateur `splunk`.

```shell
# chown -R splunk:splunk /opt/splunk
```

### Vérification

Vérification des fichiers.

```shell
/opt/splunk/bin/splunk validate files
```

### `splunk start`

```shell
/opt/splunk/bin/splunk start --accept-license --answer-yes
```

Plusieurs options peuvent être ajoutées, comme l'acceptation de la licence, l'accord avec toutes les questions, l'absence de question et le renseignement d'un mot de passe.

```shell
/opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt --seed-passwd <SPLUNK_ADMIN_PASSWORD>
```

## Administration

### WebUI TLS

Configuration de la WebUI pour utiliser TLS, en utilisant les certificats auto-générés à l'installation.

```shell
tee /opt/splunk/etc/system/local/web.conf <<EOF
[settings]
enableSplunkWebSSL = true
serverCert = /opt/splunk/etc/auth/splunkweb/cert.pem
privKeyPath = /opt/splunk/etc/auth/splunkweb/privkey.pem
EOF
```

### `wiredTiger`

Migration du moteur de base de données, passage de `mmapv1` à `wiredTiger`.

```shell
/opt/splunk/bin/splunk show kvstore-status
```

```shell
/opt/splunk/bin/splunk stop
```

```shell
vi /opt/splunk/etc/system/local/server.conf

[kvstore]
storageEngine=wiredTiger
```

```shell
/opt/splunk/bin/splunk start
```

Vérification de la migration.

```shell
/opt/splunk/bin/splunk show kvstore-status | grep storageEngine
```

### Search Head Captain

Vérification du Captain des Search Head, qui sera le dernier nœud à être redémarré.

```shell
/opt/splunk/bin/splunk show shcluster-status -auth:<SPLUNK_ADMIN_USERNAME>
```

Le redémarrage de Splunk sur le Captain doit être précédé du transfert de rôle sur un nœud déjà redémarré.

```shell
/opt/splunk/bin/splunk transfer shcluster-captain -mgmt_uri https://<HOSTNAME>:8089 -auth <SPLUNK_ADMIN_USERNAME>:
```

Détention manuelle.

```shell
/opt/splunk/bin/splunk edit cluster-config -auth -manual_detention on
```

### KVstore

Vérification du KVStore.

```shell
/opt/splunk/bin/splunk show kvstore-status
```

Archivage du KVStore.

```shell
/opt/splunk/bin/splunk backup kvstore -archiveName $(uname -n)_splunk_kvstore_$(date +%s)
```

Localisation de l'archive : `/opt/splunk/var/lib/splunk/kvstorebackup/`.

### Indexer

Vérification du statut du cluster d'Indexer.

```shell
/opt/splunk/bin/splunk show cluster-bundle-status
```

### Déploiement

Rechargement du serveur de déploiement, le répertoire concerné est `/opt/splunk/etc/deployment-apps`.

```shell
/opt/splunk/bin/splunk reload deploy-server
/opt/splunk/bin/splunk set deploy-poll <SPLUNK_DEPLOYER_SERVER>:8089
```

#### Search Head

Vérification du cluster de Search Head.

```shell
/opt/splunk/bin/splunk show shcluster-status --verbose
```

Déploiement d'un bundle pour Search Head.

```shell
/opt/splunk/bin/splunk apply shcluster-bundle --answer-yes -target https://<HOSTNAME>:8089 -auth <SPLUNK_ADMIN_USERNAME>:
```

Les arguments peuvent être ajoutés : `--answer-yes` ; `-force true`.

Déploiement d'un bundle pour Search Head sans déclenchement d'un redémarrage de l'application.

```shell
/opt/splunk/bin/splunk apply shcluster-bundle --answer-yes -target https://<HOSTNAME>:8089 -auth <SPLUNK_ADMIN_USERNAME>: -action stage && /opt/splunk/bin/splunk apply shcluster-bundle -action send
```

Déclencher un redémarrage de l'application sur les Search Head.

```shell
/opt/splunk/bin/splunk rolling-restart shcluster-members
```

Vérification du statut du déploiement.

```shell
/opt/splunk/bin/splunk list shcluster-bundle -member_uri https://<HOSTNAME>:8089 -auth <SPLUNK_ADMIN_USERNAME>:
```

#### Indexer

Vérification d'un bundle, avec ou sans vérification d'un redémarrage de l'application.

```shell
/opt/splunk/bin/splunk validate cluster-bundle --check-restart
```

Vérification du statuts d'un bundle.

```shell
/opt/splunk/bin/splunk show cluster-bundle-status
```

Application d'un bundle.

```shell
/opt/splunk/bin/splunk apply cluster-bundle --answer-yes
```

Retour arrière d'un bundle.

```shell
/opt/splunk/bin/splunk rollback cluster-bundle
```

### Secrets

Affichage d'un secret.

```shell
/opt/splunk/bin/splunk show-decrypted --value '<pass4SymmKey>'
```

### `btool`

Vérification des applications.

```shell
/opt/splunk/bin/splunk btool check
```

### Sauvegarde

Sauvegarde du répertoire de configuration Splunk.

```shell
tar czfhv $(uname -n)_$(date +%s)_splunk_etc.tgz /opt/splunk/etc/
chmod 755 $(uname -n)_$(date +%s)_splunk_etc.tgz
```

## Splunk TA

[Splunk TA (Technology Add-on)](https://dev.splunk.com/enterprise/docs/welcome/#What-is-a-Splunk-add-on) sont des applications complémentaires à Splunk, édités par SPlunk ou des tiers.
Il s'agit d'archive compressée au format `*.zip` ou `*.tgz` à déployer sur un Cluster Master ou un Search Head.

## Securité

Le chiffrement du transit des données serait systématique selon cet [article](https://www.splunk.com/en_us/legal/splunk-observability-security-addendum.html#:~:text=Splunk%20uses%20industry%2Dstandard%20encryption,encryption%20for%20web%20communication%20sessions.).

> Splunk uses industry-standard encryption techniques to encrypt Customer Content in transit. The Splunk Service is configured by default to encrypt user data files using transport layer security (currently, TLS 1.2+) encryption for web communication sessions.

Mais en vérifiant les configurations, il semble que non :

```shell
$ /opt/splunk/bin/splunk btool inputs list --debug | grep enableSSL

/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0
/opt/splunk/etc/apps/splunk_httpinput/local/inputs.conf                enableSSL = 0

$ /opt/splunk/bin/splunk btool inputs list --debug | grep enableSSL
```
