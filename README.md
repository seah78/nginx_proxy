# nginx_proxy

[![forthebadge](https://forthebadge.com/images/badges/docker-container.svg)](https://forthebadge.com)

# Reverse Proxy Nginx + Let's Encrypt (Certbot)

## Objectif

Ce projet fournit :

- un reverse proxy Nginx pour router plusieurs sites/applications Docker
- la génération automatique de certificats SSL Let's Encrypt
- le renouvellement automatique des certificats via Certbot
- l’isolation réseau entre les différents projets Docker

---

# Architecture

## Containers

### nginx_proxy
Gère :

- Reverse proxy vers les différents containers applicatifs
- Redirection HTTP → HTTPS
- Terminaison SSL/TLS
- Validation ACME pour Let's Encrypt

### certbot
Gère :

- Création des certificats
- Renouvellement automatique toutes les 12 heures

---

# Arborescence

```bash
nginx_proxy/
├── docker-compose.yml
├── nginx-config/
│   ├── site1.conf
│   └── site2.conf
├── letsencrypt/
├── html/
└── README.md
```

## Volumes

### Configurations Nginx
```bash
./nginx-config
```

### Certificats SSL
```bash
./letsencrypt
```

### Webroot ACME challenge
```bash
./html
```

---

# Pré-requis

- VPS / serveur dédié
- Docker installé
- Docker Compose plugin (`docker compose`)
- Domaines pointant vers le serveur
- Ports ouverts :
- 80/tcp
- 443/tcp

---

# Installation

## Cloner le projet

```bash
git clone https://github.com/seah78/nginx_proxy.git
cd nginx_proxy
```

---

## Créer les réseaux Docker externes

```bash
docker network create suppliers_network
docker network create customers_network
docker network create internal_network
```

---

## Démarrer les services

```bash
docker compose up -d
```

Vérifier :

```bash
docker ps
```

---

# Configuration initiale (obtention du certificat)

Créer un vhost temporaire :

```bash
nano nginx-config/mondomaine.conf
```

```nginx
server {
    listen 80;
    server_name mondomaine.com;

    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

Tester :

```bash
docker exec nginx_proxy nginx -t
```

Puis recharger :

```bash
docker restart nginx_proxy
```

---

# Génération du certificat SSL

```bash
docker run --rm \
-v $(pwd)/letsencrypt:/etc/letsencrypt \
-v $(pwd)/html:/usr/share/nginx/html \
certbot/certbot certonly --webroot \
--webroot-path=/usr/share/nginx/html \
-d mondomaine.com \
--email email@mondomaine.com \
--agree-tos \
--non-interactive
```

Répéter pour chaque domaine.

---

# Configuration finale

Modifier :

```bash
nano nginx-config/mondomaine.conf
```

```nginx
server {
    listen 80;
    server_name mondomaine.com;

    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
    }

    return 301 https://$host$request_uri;
}


server {
    listen 443 ssl http2;
    server_name mondomaine.com;

    ssl_certificate /etc/letsencrypt/live/mondomaine.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mondomaine.com/privkey.pem;

    location / {
        proxy_pass http://container-projet:port;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Tester :

```bash
docker exec nginx_proxy nginx -t
```

Puis :

```bash
docker restart nginx_proxy
```

---

# Renouvellement automatique

Aucune action manuelle nécessaire.

Le container Certbot exécute automatiquement :

```bash
certbot renew
```

toutes les 12 heures.

Vérification :

```bash
docker logs certbot
```

Test manuel :

```bash
docker exec certbot certbot renew --dry-run
```

---

# Ajouter un nouveau projet

1. Connecter le container applicatif à un des réseaux :

```yaml
networks:
  - internal_network
```

2. Créer son vhost Nginx

3. Générer le certificat

4. Recharger Nginx

---

# Dépannage

Tester config :

```bash
docker exec nginx_proxy nginx -t
```

Logs Nginx :

```bash
docker logs nginx_proxy
```

Logs Certbot :

```bash
docker logs certbot
```

---

# Auteur

[![forthebadge](https://forthebadge.com/images/badges/built-by-developers.svg)](https://forthebadge.com)

**Sébastien HERLANT**


