# Gitea + Woodpecker CI — CI/CD auto-hébergée (homelab)

[English version](README.md)

Stack CI/CD auto-hébergée sur un homelab (Raspberry Pi 5), sans aucune exposition publique. Accès exclusivement via un VPN mesh (Tailscale ou équivalent) + résolution DNS locale côté client. Les secrets sont gérés dans [Infisical](https://github.com/Alithiel31/infisical-homelab) et injectés au déploiement.

## Architecture

```
Navigateur (client sur le VPN mesh)
        │  résout gitea.homelab.internal / woodpecker.homelab.internal
        │  via une entrée dans le fichier hosts local → IP VPN du homelab
        ▼
   Traefik (reverse proxy déjà en place, entrypoint "web", port host 8000)
        │  routage par Host() header, réseau traefik-net
        ▼
   ┌─────────────┐         ┌────────────────────┐
   │    Gitea    │◄───────►│  Woodpecker Server  │
   │ (forge git) │  OAuth2 │  (orchestrateur CI) │
   └──────┬──────┘         └──────────┬──────────┘
          │                           │ gRPC (port 9000)
          │ Postgres natif            ▼
          │ (172.16.0.1:5432)  ┌──────────────────┐
          └───────────────────►│ Woodpecker Agent  │
                                │ (exécute les      │
                                │  pipelines Docker)│
                                └──────────────────┘
```

- **Gitea** : forge Git auto-hébergée, remplace GitHub pour garder toute la chaîne (webhooks inclus) strictement interne.
- **Woodpecker Server** : reçoit les webhooks de Gitea, orchestre les pipelines, sert l'interface web.
- **Woodpecker Agent** : exécute réellement les pipelines dans des conteneurs Docker (via le socket Docker de l'hôte).
- **Traefik** ([traefik-homelab](https://github.com/Alithiel31/traefik-homelab)) : reverse proxy déjà en place sur le homelab, routage par domaine (labels Docker, `exposedbydefault=false`).
- **Postgres** : instance native mutualisée du homelab (pas de conteneur dédié), une base par service (`gitea`, `woodpecker`).
- **Infisical** ([infisical-homelab](https://github.com/Alithiel31/infisical-homelab)) : fournit les secrets (mots de passe DB, secret gRPC, credentials OAuth2) injectés par `deploy.sh` ; plus aucun secret dans `.env`.

Réseau Docker dédié `ci-net` (`172.16.0.0/24` par défaut, configurable via `.env`), séparé de `traefik-net`.

## Prérequis

Avant de lancer cette stack, le homelab doit déjà avoir :

- **Docker + Docker Compose** installés.
- **Traefik** déjà déployé et fonctionnel, avec :
  - un réseau Docker externe nommé `traefik-net` (`docker network create traefik-net` si besoin) ;
  - le provider Docker activé avec `exposedbydefault=false` (routage par labels uniquement) ;
  - un entrypoint `web` écoutant sur le port host `8000` (ou adapter les URLs de ce README/du compose à ton port réel).
- **Infisical** déployé et accessible, avec un projet « Shared Keys » (environnement `prod`) et la [CLI Infisical](https://infisical.com/docs/cli/overview) installée sur l'hôte.
- **Postgres** (natif sur l'hôte, pas en conteneur) accessible depuis les conteneurs Docker, avec `listen_addresses` incluant l'IP de la passerelle du futur réseau `ci-net`.
- **ufw** (ou équivalent) actif avec refus par défaut en entrée — les règles d'autorisation précises sont données plus bas.
- Un **VPN mesh** (Tailscale ou équivalent) donnant accès au homelab depuis les postes clients.
- **Gitea et Woodpecker n'ont pas besoin d'être installés au préalable** — cette stack les déploie tous les deux.

## Installation from scratch

### 1. Choisir le sous-réseau Docker dédié

Vérifie qu'aucun réseau Docker existant sur l'hôte n'entre en conflit avec le futur `ci-net` :
```bash
docker network ls -q | xargs -I{} docker network inspect {} --format '{{.Name}}: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```
Ajuste `CI_NET_SUBNET`/`CI_NET_GATEWAY` dans `.env` si `172.16.0.0/24` est déjà pris.

### 2. Créer les bases et utilisateurs Postgres

Sur l'hôte, en tant qu'utilisateur Postgres admin :
```sql
CREATE USER gitea_app WITH PASSWORD 'un-mot-de-passe-fort';
CREATE DATABASE gitea OWNER gitea_app;

CREATE USER woodpecker_app WITH PASSWORD 'un-autre-mot-de-passe-fort';
CREATE DATABASE woodpecker OWNER woodpecker_app;
```
⚠️ Pour `WOODPECKER_DB_PASS`, génère le mot de passe avec `openssl rand -hex 24` (uniquement hexadécimal) plutôt que `base64` : Woodpecker l'utilise dans une URL DSN (`postgres://user:pass@host/db`) et un caractère spécial comme `/` casse le parsing.

Ajoute la règle d'accès dans `pg_hba.conf` (adapter le chemin selon la version Postgres) :
```
host    gitea        gitea_app        <CI_NET_SUBNET>    scram-sha-256
host    woodpecker   woodpecker_app   <CI_NET_SUBNET>    scram-sha-256
```
Puis recharge Postgres (`sudo systemctl reload postgresql` ou équivalent).

### 3. Ouvrir les ports nécessaires dans le pare-feu

Le réseau `ci-net` doit pouvoir atteindre Postgres (5432) et Traefik (8000) sur l'hôte :
```bash
sudo ufw allow from <CI_NET_SUBNET> to any port 5432 proto tcp comment "gitea+woodpecker -> postgres mutualise"
sudo ufw allow from <CI_NET_SUBNET> to any port 8000 proto tcp comment "gitea+woodpecker -> traefik"
```
Sans ces règles : timeout silencieux (pas de rejet explicite) lors du démarrage — voir la section Dépannage plus bas.

### 4. Configurer le DNS interne côté client

Ajoute au fichier hosts de chaque poste client qui doit y accéder (`C:\Windows\System32\drivers\etc\hosts` sous Windows, `/etc/hosts` sous Linux/macOS) :
```
<IP_VPN_DU_HOMELAB>  gitea.homelab.internal
<IP_VPN_DU_HOMELAB>  woodpecker.homelab.internal
```
(remplace par les domaines réellement choisis dans `.env` si différents)

### 5. Préparer la configuration et les secrets

**Paramètres non secrets** (`.env`) :
```bash
cp .env.example .env
```
Renseigne les valeurs (voir [Variables d'environnement](#variables-denvironnement)).

**Secrets** (Infisical, projet « Shared Keys », environnement `prod`) : crée ces entrées. Pour `WOODPECKER_GITEA_CLIENT`/`WOODPECKER_GITEA_SECRET`, l'app OAuth2 Gitea n'existe pas encore : elles seront ajoutées à l'étape 7.
- `GITEA_DB_PASS`
- `WOODPECKER_DB_PASS`
- `WOODPECKER_AGENT_SECRET` (`openssl rand -hex 32` par ex.)

**Accès de `deploy.sh` à Infisical** : crée une Machine Identity `woodpecker-ci-deploy` (Universal Auth, accès en lecture au projet), puis :
```bash
cp .infisical-identity.env.example .infisical-identity.env
```
et renseigne le Client ID, le Client Secret, l'ID du projet (`INFISICAL_PROJECT_ID`) et `INFISICAL_API_URL`. Ce fichier n'est jamais commité.

### 6. Démarrer Gitea seul, puis créer le compte admin

`deploy.sh` transmet ses arguments à `docker compose up -d`. Pour ne démarrer que Gitea :
```bash
./deploy.sh gitea
```
Comme `GITEA__security__INSTALL_LOCK=true` est déjà positionné, l'installeur web est court-circuité. Crée le compte admin directement en CLI :
```bash
docker exec -u git gitea gitea admin user create --username <user> --password "<pass>" --email <email> --admin
```
(`-u git` obligatoire — le process Gitea tourne en `git`, pas en `root`.)

Connecte-toi ensuite sur `http://<GITEA_DOMAIN>:8000/` avec ce compte.

### 7. Créer l'application OAuth2 dans Gitea pour Woodpecker

Dans Gitea : **Paramètres du site → Applications → Applications OAuth2 gérées → Créer une application OAuth2**.
- Nom : `Woodpecker CI` (ou autre)
- URL de redirection : `http://<WOODPECKER_DOMAIN>:8000/authorize`

Copie le **Client ID** et le **Client Secret** générés dans Infisical (projet « Shared Keys », environnement `prod`) sous `WOODPECKER_GITEA_CLIENT` / `WOODPECKER_GITEA_SECRET`.

### 8. Démarrer le reste de la stack

```bash
./deploy.sh
```
Le script s'authentifie auprès d'Infisical avec la Machine Identity puis lance `docker compose up -d` avec les secrets injectés. Vérifie les logs :
```bash
docker logs woodpecker-server --tail 30
docker logs woodpecker-agent --tail 30
```
Le serveur ne doit **pas** afficher `WOODPECKER_GRPC_SECRET is not set` (sinon `WOODPECKER_AGENT_SECRET` n'a pas été repris correctement dans `.env`). L'agent doit afficher `polling new workflow` sans erreur `fatal`.

### 9. Vérifier que l'agent est bien connecté

Connecte-toi sur `http://<WOODPECKER_DOMAIN>:8000/` avec "Login with Gitea", puis va dans **Admin → Agents** (icône engrenage, visible seulement si ton compte Gitea est listé dans `WOODPECKER_ADMIN_USER`). Un agent avec un "last contact" récent confirme que tout fonctionne.

## Accès

| Service | URL | Notes |
|---|---|---|
| Gitea | `http://gitea.homelab.internal:8000/` | admin : `<votre-username>` |
| Woodpecker | `http://woodpecker.homelab.internal:8000/` | login via "Login with Gitea" (OAuth2) |

## Fichiers

- `docker-compose.yml` — définition des 3 services (`gitea`, `woodpecker-server`, `woodpecker-agent`)
- `deploy.sh` — authentification Infisical + `docker compose up -d [args]` avec les secrets injectés (ex. `./deploy.sh gitea`)
- `.env` — paramètres non secrets (jamais commité, voir `.env.example`)
- `.infisical-identity.env` — Client ID/Secret de la Machine Identity (jamais commité, voir `.infisical-identity.env.example`)

## Variables d'environnement

### Dans `.env` (non secrètes, voir `.env.example`)

| Variable | Usage |
|---|---|
| `CI_NET_SUBNET` / `CI_NET_GATEWAY` | sous-réseau Docker dédié à Gitea/Woodpecker et sa passerelle |
| `GITEA_DOMAIN` / `WOODPECKER_DOMAIN` | domaines internes utilisés par Traefik et résolus côté client |
| `WOODPECKER_ADMIN_USER` | username Gitea qui doit avoir les droits admin sur Woodpecker |

### Dans Infisical (projet « Shared Keys », environnement `prod`)

| Variable | Usage |
|---|---|
| `GITEA_DB_PASS` | mot de passe Postgres du user `gitea_app` |
| `WOODPECKER_DB_PASS` | mot de passe Postgres du user `woodpecker_app` (généré en hex, jamais base64 — un `/` casse le parsing DSN `postgres://user:pass@host/db`) |
| `WOODPECKER_AGENT_SECRET` | secret partagé gRPC serveur↔agent. **Attention** : mappé sur `WOODPECKER_GRPC_SECRET` côté serveur et `WOODPECKER_AGENT_SECRET` côté agent dans le compose — deux noms différents pour la même valeur (piège de la v3.18.0, voir plus bas) |
| `WOODPECKER_GITEA_CLIENT` / `WOODPECKER_GITEA_SECRET` | credentials de l'app OAuth2 "Woodpecker CI" créée dans Gitea (voir étape 7 de l'installation) |

## Sécurité

- `WOODPECKER_OPEN=true` : tout compte Gitea peut se connecter à Woodpecker. Acceptable sur un réseau privé, à revoir si d'autres utilisateurs obtiennent un compte Gitea.
- L'agent monte le socket Docker de l'hôte (`/var/run/docker.sock`, en lecture-écriture) : un pipeline peut contrôler Docker sur l'hôte. Ne lancer que des dépôts de confiance.
- Les secrets ne vivent que dans Infisical ; ne jamais commiter `.env` ni `.infisical-identity.env`.

## Dépannage / pièges rencontrés

1. **ufw bloque silencieusement les nouveaux sous-réseaux Docker.** Toute connexion sortante d'un conteneur `ci-net` vers un port de l'hôte (Postgres 5432, Traefik 8000) nécessite une règle explicite (voir étape 3 de l'installation). Sans ça : timeout silencieux, pas de rejet explicite — piège à diagnostiquer avec `docker exec <conteneur> wget -T 3 -O- http://<host>:<port>` (timeout pile à `-T` = bloqué réseau ; erreur quasi instantanée = port joignable). Note : les images Woodpecker sont "distroless", sans `wget`/`sh` — ce test ne marche que sur des conteneurs avec un shell (ex. Gitea).

2. **`extra_hosts: host.docker.internal:host-gateway` ne fonctionne pas sur un réseau custom.** Le mapping magique `host-gateway` résout toujours vers la passerelle du bridge Docker par défaut (`172.17.0.1`), jamais vers celle d'un réseau custom. Il faut hardcoder l'IP de la passerelle réelle (`CI_NET_GATEWAY`).

3. **`WOODPECKER_GITEA_URL` sert à la fois côté serveur ET côté navigateur client.** Ne jamais y mettre un nom de service Docker interne (`http://gitea:3000`) : ça casse la redirection OAuth "Login with Gitea" côté navigateur, qui ne peut pas résoudre ce nom. Utiliser le domaine interne résolu côté client, et ajouter un `extra_hosts` sur `woodpecker-server` pour qu'il le résolve aussi lui-même.

4. **Le secret gRPC partagé a un nom de variable différent selon le service** (Woodpecker v3.18.0) :
   - Serveur : `WOODPECKER_GRPC_SECRET`
   - Agent : `WOODPECKER_AGENT_SECRET`
   Même valeur, deux noms différents. Un mismatch donne soit `signature is invalid` (mauvaise valeur) soit `please provide a token` (variable pas reconnue par le binaire). En cas de doute sur les vraies variables acceptées, se fier à `docker exec <conteneur> <binaire> --help` plutôt qu'à la doc en ligne (qui peut décrire une version différente).

5. **Deux mécanismes d'agents distincts dans l'UI Woodpecker**, à ne pas confondre :
   - *Agent système* (secret partagé `WOODPECKER_GRPC_SECRET`/`WOODPECKER_AGENT_SECRET`) : auto-enregistrement au premier contact, visible uniquement dans **Admin → Agents**.
   - *Agent Token* (bouton "Ajouter un agent" dans Paramètres du compte utilisateur) : token unique généré manuellement, visible dans **Paramètres du compte → Agents**. Mécanisme différent, non utilisé dans ce déploiement.

6. **Être admin Gitea ne rend pas automatiquement admin Woodpecker.** Il faut `WOODPECKER_ADMIN_USER` correctement renseigné côté serveur, puis redémarrer le conteneur et se déconnecter/reconnecter (le statut admin est vérifié au login, pas en temps réel) pour voir apparaître le menu Admin.

## Statut

✅ Déploiement complet et fonctionnel : Gitea et Woodpecker opérationnels, agents connectés, OAuth login validé.

⬜ Reste à valider : exécution réelle d'un premier pipeline (créer un dépôt de test sur Gitea, l'activer côté Woodpecker, pousser un `.woodpecker.yml`).

## Commandes utiles

```bash
# Redéployer après modif du compose ou du .env (secrets injectés depuis Infisical)
./deploy.sh

# Forcer la recréation d'un service précis
./deploy.sh --force-recreate <service>

# Logs
docker logs gitea --tail 50
docker logs woodpecker-server --tail 50
docker logs woodpecker-agent --tail 50

# Vérifier qu'une variable d'env est bien passée à un conteneur
docker exec <conteneur> env | grep <VAR>
```

## Projets liés

| Dépôt | Rôle |
| --- | --- |
| [traefik-homelab](https://github.com/Alithiel31/traefik-homelab) | Reverse proxy |
| [infisical-homelab](https://github.com/Alithiel31/infisical-homelab) | Gestionnaire de secrets |
| [plantuml-traefik](https://github.com/Alithiel31/plantuml-traefik) | Serveur PlantUML |

## Licence

MIT — voir [LICENSE](./LICENSE).
