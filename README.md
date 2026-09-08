# Gitea + Woodpecker CI — Homelab

Stack CI/CD auto-hébergée sur un homelab (Raspberry Pi 5), sans aucune exposition publique. Accès exclusivement via un VPN mesh (Tailscale ou équivalent) + résolution DNS locale côté client.

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
- **Traefik** : reverse proxy déjà en place sur le homelab, routage par domaine (labels Docker, `exposedbydefault=false`).
- **Postgres** : instance native mutualisée du homelab (pas de conteneur dédié), une base par service (`gitea`, `woodpecker`).

Réseau Docker dédié `ci-net` (`172.16.0.0/24`, gateway `172.16.0.1`), séparé de `traefik-net`.

## Accès

| Service | URL | Notes |
|---|---|---|
| Gitea | `http://gitea.homelab.internal:8000/` | admin : `<votre-username>` |
| Woodpecker | `http://woodpecker.homelab.internal:8000/` | login via "Login with Gitea" (OAuth2) |

Prérequis client : être connecté au VPN mesh, et avoir dans son fichier hosts (`C:\Windows\System32\drivers\etc\hosts` sous Windows, `/etc/hosts` sous Linux/macOS) :
```
<IP_VPN_DU_HOMELAB>  gitea.homelab.internal
<IP_VPN_DU_HOMELAB>  woodpecker.homelab.internal
```

## Fichiers

- `docker-compose.yml` — définition des 3 services (`gitea`, `woodpecker-server`, `woodpecker-agent`)
- `.env` — secrets et mots de passe (jamais commité, voir `.env.example`)

Déployé via un contexte Docker distant pointant sur le homelab :
```bash
docker compose up -d
```

## Variables d'environnement (voir `.env.example`)

| Variable | Usage |
|---|---|
| `GITEA_DB_PASS` | mot de passe Postgres du user `gitea_app` |
| `WOODPECKER_DB_PASS` | mot de passe Postgres du user `woodpecker_app` (généré en hex, jamais base64 — un `/` casse le parsing DSN `postgres://user:pass@host/db`) |
| `WOODPECKER_AGENT_SECRET` | secret partagé gRPC serveur↔agent. **Attention** : mappé sur `WOODPECKER_GRPC_SECRET` côté serveur et `WOODPECKER_AGENT_SECRET` côté agent dans le compose — deux noms différents pour la même valeur (piège de la v3.18.0, voir plus bas) |
| `WOODPECKER_GITEA_CLIENT` / `WOODPECKER_GITEA_SECRET` | credentials de l'app OAuth2 "Woodpecker CI" créée dans Gitea (Paramètres du site → Applications → Applications OAuth2 gérées) |

## Points d'attention / pièges rencontrés

1. **ufw bloque silencieusement les nouveaux sous-réseaux Docker.** Toute connexion sortante d'un conteneur `ci-net` vers un port de l'hôte (Postgres 5432, Traefik 8000) nécessite une règle explicite :
   ```bash
   sudo ufw allow from 172.16.0.0/24 to any port <port> proto tcp
   ```
   Sans ça : timeout silencieux, pas de rejet explicite — piège à diagnostiquer avec `docker exec <conteneur> wget -T 3 -O- http://<host>:<port>` (timeout pile à `-T` = bloqué réseau ; erreur quasi instantanée = port joignable).

2. **`extra_hosts: host.docker.internal:host-gateway` ne fonctionne pas sur un réseau custom.** Le mapping magique `host-gateway` résout toujours vers la passerelle du bridge Docker par défaut (`172.17.0.1`), jamais vers celle d'un réseau custom. Il faut hardcoder l'IP de la passerelle réelle (`172.16.0.1` ici).

3. **`WOODPECKER_GITEA_URL` sert à la fois côté serveur ET côté navigateur client.** Ne jamais y mettre un nom de service Docker interne (`http://gitea:3000`) : ça casse la redirection OAuth "Login with Gitea" côté navigateur, qui ne peut pas résoudre ce nom. Utiliser le domaine interne résolu côté client (`http://gitea.homelab.internal:8000`), et ajouter un `extra_hosts` sur `woodpecker-server` pour qu'il le résolve aussi lui-même.

4. **Le secret gRPC partagé a un nom de variable différent selon le service** (Woodpecker v3.18.0) :
   - Serveur : `WOODPECKER_GRPC_SECRET`
   - Agent : `WOODPECKER_AGENT_SECRET`
   Même valeur, deux noms différents. Un mismatch donne soit `signature is invalid` (mauvaise valeur) soit `please provide a token` (variable pas reconnue par le binaire). En cas de doute sur les vraies variables acceptées, se fier à `docker exec <conteneur> <binaire> --help` plutôt qu'à la doc en ligne (qui peut décrire une version différente).

5. **Deux mécanismes d'agents distincts dans l'UI Woodpecker**, à ne pas confondre :
   - *Agent système* (secret partagé `WOODPECKER_GRPC_SECRET`/`WOODPECKER_AGENT_SECRET`) : auto-enregistrement au premier contact, visible uniquement dans **Admin → Agents**.
   - *Agent Token* (bouton "Ajouter un agent" dans Paramètres du compte utilisateur) : token unique généré manuellement, visible dans **Paramètres du compte → Agents**. Mécanisme différent, non utilisé dans ce déploiement.

6. **Être admin Gitea ne rend pas automatiquement admin Woodpecker.** Il faut `WOODPECKER_ADMIN=<username_gitea>` côté serveur, puis redémarrer le conteneur et se déconnecter/reconnecter (le statut admin est vérifié au login, pas en temps réel) pour voir apparaître le menu Admin.

7. **`GITEA__security__INSTALL_LOCK=true`** évite de passer par l'installeur web (qui peut rester bloqué si la DB n'est pas encore joignable) — le compte admin se crée alors en CLI :
   ```bash
   docker exec -u git gitea gitea admin user create --username <user> --password "<pass>" --email <email> --admin
   ```
   (`-u git` obligatoire, le process tourne en `git`, pas en `root`.)

## Statut

✅ Déploiement complet et fonctionnel : Gitea et Woodpecker opérationnels, agents connectés, OAuth login validé.

⬜ Reste à valider : exécution réelle d'un premier pipeline (créer un dépôt de test sur Gitea, l'activer côté Woodpecker, pousser un `.woodpecker.yml`).

## Commandes utiles

```bash
# Redéployer après modif du compose ou du .env
docker compose up -d

# Forcer la recréation d'un service précis
docker compose up -d --force-recreate <service>

# Logs
docker logs gitea --tail 50
docker logs woodpecker-server --tail 50
docker logs woodpecker-agent --tail 50

# Vérifier qu'une variable d'env est bien passée à un conteneur
docker exec <conteneur> env | grep <VAR>
```
