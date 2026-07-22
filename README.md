# 🔔 Intégration des notifications Zabbix avec Mattermost

> Objectif : router les alertes Zabbix vers un canal Mattermost dédié pour centraliser la remontée d'incidents, à la manière d'un vrai canal d'astreinte DSI.

## Statut

🚧 **Chantier planifié** — cette intégration fait partie des perspectives du projet, en parallèle du déploiement de Mattermost lui-même (voir `mattermost-canaux-communication`), et n'est pas encore mise en œuvre au moment de la rédaction.

## Principe visé

Zabbix dispose nativement d'un type de média **Webhook**, qui peut être configuré pour poster vers l'API Incoming Webhook de Mattermost. L'objectif est de déclencher une notification Mattermost à chaque changement de sévérité d'un trigger (problème détecté / résolu), avec un formatage incluant l'hôte concerné, le trigger déclenché et le niveau de sévérité.

## Pourquoi Mattermost plutôt qu'un canal e-mail

Dans le scénario d'entreprise RAID-A-PORTER, Mattermost joue le rôle de l'outil de communication interne (équivalent Slack/Teams auto-hébergé). Faire remonter les alertes de supervision dans ce même outil reproduit un flux d'astreinte réaliste, plutôt que de multiplier les canaux de notification.

## Prochaines étapes

- [ ] Déployer Mattermost (`mattermost-canaux-communication`)
- [ ] Créer le canal `#supervision-zabbix`
- [ ] Configurer le média Webhook côté Zabbix et le mapper aux actions d'alerte
- [ ] Documenter le format de message et les règles d'escalade

## Repos liés

- `zabbix-dashboard-supervision`
- `mattermost-canaux-communication`

## Auteur

**Lilian Vertueux** — [LinkedIn](https://www.linkedin.com/in/lilian-vertueux/)
