# nginx_proxy

[![forthebadge](https://forthebadge.com/images/badges/docker-container.svg)](https://forthebadge.com)

## Objectif : 
Gérer les redirections vers les différentes images docker et générer les certificats SSL pour les différents container.

## Containers de ce projet :
Nginx gère le système de redirection par l'intermédiaire des fichiers de configuration.

Certbot gère la création et le renouvellement des cetificats SSL.

## Arborescence
Les fichiers de configuration sont stockés dans le dossier nginx-config

Les certificats SSL sont stockés dans le dossier letsencrypt.

## Pré-requis

- Terminal
- VPS ou Serveur Dédié
- Docker et docker-compose installé sur le système

## Installation et démarrage

Cloner le repository sur le VPS ou le Serveur dédié

`git clone https://github.com/J2S-Just-Simple-Solutions/nginx_proxy`

Se positionner dans le dossier du projet

`cd nginx-proxy`

Créer les réseaux

```
docker network create registry
docker network create suppliers_network
docker network create customers_network
docker network create internal_network
```

Démarrer le container

`docker-compose up -d`

## Configuration initiale

Se positionner dans le nginx-conf et créer un fichier de configuration initial pour la génération du certificat SSL.

`nano monfichier1.conf`

Utiliser la nomenclature suivante :

```
server {
    listen 80;
    server_name mondomaine.com;

    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
    }

    location / {
        if ($scheme != "https") {
            return 301 https://$host$request_uri;
        }
    }
}
```

*Générer autant de fichier de conf qu'il y a de domaine à configurer*

## Génération du certificat SSL

Utiliser la commande suivante pour générer le certificat SSL pour le domaine

```
docker run --rm \
    -v $(pwd)/letsencrypt:/etc/letsencrypt \
    -v $(pwd)/nginx-config:/etc/nginx/conf.d \
    -v $(pwd)/html:/usr/share/nginx/html \
    certbot/certbot certonly --webroot \
    --webroot-path=/usr/share/nginx/html \
    -d mondomaine.com \
    --email email@mondomaine.com \
    --agree-tos \
    --non-interactive

```

*Commande à utiliser pour chaque domaine à configurer*

Redémarrer Nginx

`docker restart nginx_proxy`

## Configuration finale

Modifier le fichier de configuration

`nano monfichier1.conf`

Utiliser la nomenclature suivante :

```
server {
    listen 80;
    server_name mondomaine.com;

    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
    }

    location / {
        if ($scheme != "https") {
            return 301 https://$host$request_uri;
        }
    }
}

server {
    listen 443 ssl;
    server_name mondomaine.com;

    ssl_certificate /etc/letsencrypt/live/mondomaine.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mondomaine.com/privkey.pem;

    location / {
        proxy_pass http://container-projet:port;
    }
}
```

Redémarrer Nginx

`docker restart nginx_proxy`

## Auteurs

[![forthebadge](https://forthebadge.com/images/badges/built-by-developers.svg)](https://forthebadge.com)

### Sébastien HERLANT 


