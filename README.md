# 🔔 Intégration des notifications Zabbix avec Mattermost
> Objectif : router les alertes Zabbix vers un canal Mattermost dédié pour centraliser la remontée d'incidents, à la manière d'un vrai canal d'astreinte DSI.
> 
Intégration des notifications Zabbix vers un canal Mattermost via un webhook basé sur un **bot account**.

Ce repository documente la configuration de bout en bout : déploiement Mattermost (Docker), création du bot, media type Zabbix, actions et pièges rencontrés en production sur le homelab RAIDAPORTER&Co.

## Pourquoi Mattermost plutôt qu'un canal e-mail

Dans le scénario d'entreprise RAIDAPORTER&Co, Mattermost joue le rôle de l'outil de communication interne (équivalent Slack/Teams auto-hébergé). Faire remonter les alertes de supervision dans ce même outil reproduit un flux d'astreinte réaliste, plutôt que de multiplier les canaux de notification.

## Sommaire

- [Prérequis](#prérequis)
- [1. Déploiement Mattermost](#1-déploiement-mattermost)
- [2. Configuration du Site URL](#2-configuration-du-site-url)
- [3. Création du bot account](#3-création-du-bot-account)
- [4. Media type Zabbix](#4-media-type-zabbix)
- [5. Action Zabbix](#5-action-zabbix)
- [6. Vérification indépendante (curl)](#6-vérification-indépendante-curl)
- [Dépannage](#dépannage)
- [Notes de sécurité](#notes-de-sécurité)

## Prérequis

- Un serveur Mattermost accessible en interne (Docker recommandé, cf. [mattermost/docker](https://github.com/mattermost/mattermost/tree/master/server/build))
- Un serveur Zabbix (6.x/7.x) avec accès réseau vers Mattermost
- Un nom DNS interne résolu **à la fois** par les postes clients et par l'hôte Mattermost lui-même

## 1. Déploiement Mattermost

Le template officiel `mattermost/docker` fournit plusieurs combinaisons de fichiers compose. Sans reverse-proxy dédié, utilisez `docker-compose.yml` + `docker-compose.without-nginx.yml`, qui publie les ports `8065` (HTTP) et `8443` vers l'hôte :

```bash
cd ~/docker/mattermost
docker compose -f docker-compose.yml -f docker-compose.without-nginx.yml up -d
```

> ⚠️ Sans `docker-compose.without-nginx.yml`, aucun port n'est mappé sur l'hôte et Mattermost reste injoignable même par IP.

Pour éviter d'oublier cette combinaison de fichiers à chaque redémarrage, fixez-la une bonne fois dans le `.env` :

```bash
echo 'COMPOSE_FILE=docker-compose.yml:docker-compose.without-nginx.yml' >> .env
```

## 2. Configuration du Site URL

Le Site URL Mattermost est piloté par la variable d'environnement `MM_SERVICESETTINGS_SITEURL`, construite dans le `.env` à partir de `DOMAIN` :

```env
DOMAIN=mattermost.example.local
MM_SERVICESETTINGS_SITEURL=https://${DOMAIN}
```

Ce champ apparaît **grisé** dans le System Console (piloté par variable d'environnement) — toute modification se fait uniquement via le `.env`, jamais depuis l'interface.

Points à respecter :

- Le nom de domaine doit être résolu à la fois par les postes clients **et** par le conteneur Mattermost lui-même
- Le schéma (`http`/`https`) et le port doivent correspondre à ce qui écoute réellement :
  - `Connection Security = None` (pas de TLS géré par Mattermost) → utiliser `http://` avec le port explicite (`:8065`)
  - Reverse-proxy en frontal avec TLS → `https://` sans port (443 implicite)

Exemple sans TLS géré par Mattermost :

```env
MM_SERVICESETTINGS_SITEURL=http://mattermost.example.local:8065
```

Après toute modification du `.env`, il faut **recréer** le conteneur (un simple restart du service ou de l'hôte ne suffit pas, la variable n'est relue qu'à la création) :

```bash
docker compose -f docker-compose.yml -f docker-compose.without-nginx.yml down
docker compose -f docker-compose.yml -f docker-compose.without-nginx.yml up -d --force-recreate
```

Validez ensuite dans **System Console > Environment > Web Server > Site URL** : le test doit passer.

<img width="1878" height="1003" alt="mattermost-web" src="https://github.com/user-attachments/assets/c695df8a-80c7-472a-a24f-bca418a656d2" />


## 3. Création du bot account

Dans Mattermost :

1. **System Console > Integrations > Bot Accounts** → activer `Enable Bot Account Creation` si nécessaire
2. **Integrations > Bot Accounts > Add Bot Account**
3. Permissions minimales : `post:all`, `post:channels`
4. Créer le bot puis générer un **Access Token** (n'est affiché qu'une seule fois — le copier immédiatement)
5. Ajouter le bot comme membre du canal/de l'équipe cible

## 4. Media type Zabbix

Importer le media type officiel Mattermost (`media_mattermost.xml`) puis configurer dans **Administration > Media types > Mattermost** :

| Paramètre | Valeur |
|---|---|
| `mattermost_url` | URL racine, sans slash final, sans `/api/v4` (ex : `http://mattermost.example.local:8065`) |
| `bot_token` | Access Token du bot créé à l'étape 3 |
| `send_to` (par utilisateur) | Nom de canal, username ou ID de canal réel |

> ⚠️ Lors du bouton **Test**, la macro `send_to` n'est pas résolue automatiquement — il faut la remplacer manuellement par une valeur réelle avant de tester.

> ⚠️ Copiez le `bot_token` en le ressaisissant plutôt qu'en le collant si possible : un espace ou un saut de ligne invisible provoque une erreur `[401] Invalid or expired session` alors que le token est valide côté Mattermost.

## 5. Action Zabbix

Une action Zabbix comporte **trois sections indépendantes**. Chacune doit contenir sa propre opération d'envoi pour couvrir tout le cycle de vie d'un problème :

| Section | Déclenchée par |
|---|---|
| **Operations** | Création du problème |
| **Update operations** | Acknowledgment, changement de sévérité, commentaire |
| **Recovery operations** | Résolution / clôture du problème |

Configurez la même opération (`Send message to <cible> via Mattermost`) dans les trois sections. Une section laissée vide ne déclenchera jamais de notification pour le cas correspondant.

**Comportement attendu :** Zabbix notifie individuellement chaque utilisateur du groupe ciblé disposant du média Mattermost actif dans son profil. Pour un message unique par événement dans le canal, n'activez le média Mattermost que sur un compte technique dédié plutôt que sur l'ensemble d'un groupe.

<img width="1870" height="1004" alt="mattermost-zabbix" src="https://github.com/user-attachments/assets/6f84fd25-1c73-405e-8211-6249c42fb148" />

## 6. Vérification indépendante (curl)

Pour isoler un problème entre Zabbix et Mattermost, testez toujours l'API en direct :

```bash
curl -s http://mattermost.example.local:8065/api/v4/users/me \
  -H "Authorization: Bearer <BOT_TOKEN>"
```

Une réponse JSON avec les informations du bot (`username`, `roles`, etc.) confirme que l'URL et le token sont valides — toute erreur provenant ensuite de Zabbix pointe vers la configuration du media type plutôt que vers Mattermost.

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Site URL : `This is not a valid live URL` | Schéma/port incohérent avec ce qui écoute réellement, ou nom non résolu depuis le conteneur | Aligner `DOMAIN` et le schéma/port sur la config réelle, vérifier la résolution DNS interne |
| Site URL grisé, non modifiable | Piloté par variable d'environnement | Modifier `.env`, recréer le conteneur |
| `.env` modifié sans effet | Reboot du LXC/hôte ≠ recréation du conteneur | `docker compose down && up -d --force-recreate` |
| Injoignable même par IP après un recreate | `docker-compose.without-nginx.yml` non inclus dans la commande | Toujours inclure les deux fichiers compose (ou fixer `COMPOSE_FILE`) |
| Test media type : `[404] Sorry, we could not find the page` | `mattermost_url` incorrecte, ou schéma/port ne correspondant à rien de réel | Corriger `mattermost_url` (racine, bon schéma/port, sans slash final) |
| Test media type : `[401] Invalid or expired session` | `bot_token` mal copié (espace/saut de ligne invisible) | Ressaisir le token proprement ; valider via `curl` |
| Pas de notification sur acknowledgment | `Update operations` vide dans l'action | Ajouter l'opération d'envoi dans cette section |
| Pas de notification à la clôture | `Recovery operations` vide dans l'action | Ajouter l'opération d'envoi dans cette section |
| Plusieurs messages identiques reçus | Plusieurs utilisateurs du groupe ont le média actif | Restreindre le média Mattermost à un compte technique unique |

## Notes de sécurité

- Ne jamais committer `bot_token` en clair dans ce repository — utiliser un `.env` local ignoré par Git (`.gitignore`) ou un gestionnaire de secrets
- Régénérer tout token exposé accidentellement (logs, capture d'écran, conversation) via **Bot Accounts > Create New Token**
- Restreindre les permissions du bot au strict nécessaire (`post:all`, `post:channels`)

## Repos liés

- [`zabbix-dashboard-supervision`](https://github.com/L-VSIX/zabbix-dashboard-supervision)
- `mattermost-canaux-communication`

## Auteur

**Lilian Vertueux** — [LinkedIn](https://www.linkedin.com/in/lilian-vertueux/)
