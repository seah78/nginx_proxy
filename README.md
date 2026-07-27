# nginx_proxy

Reverse proxy Nginx mutualisé avec certificats Let's Encrypt pour des
applications Docker.

## Fonctionnalités

- reverse proxy vers plusieurs applications Docker ;
- redirection HTTP vers HTTPS ;
- terminaison TLS ;
- validation ACME HTTP-01 ;
- renouvellement périodique des certificats avec Certbot ;
- partage optionnel des médias Django.

## Prérequis

- un serveur Linux ;
- Docker et le plugin Docker Compose ;
- les ports TCP 80 et 443 ouverts en IPv4 et, si un enregistrement DNS `AAAA`
  est publié, en IPv6 ;
- les domaines pointant vers le serveur.

## Emplacement d'installation

Ce projet doit être installé dans `/opt/nginx_proxy`, car `adminnginx` utilise
ce chemin pour accéder aux configurations, certificats et challenges ACME.

```bash
cd /opt
git clone https://github.com/seah78/nginx_proxy.git
cd /opt/nginx_proxy
```

Structure attendue :

```text
/opt/nginx_proxy/
├── docker-compose.yml
├── nginx-config/
├── letsencrypt/
├── html/
└── README.md
```

## Préparation Docker

Créer les réseaux externes nécessaires :

```bash
docker network create suppliers_network
docker network create customers_network
docker network create internal_network
```

Une erreur indiquant qu'un réseau existe déjà peut être ignorée.

Créer le volume partagé destiné aux médias :

```bash
docker volume create webapps_media
```

Créer les répertoires persistants :

```bash
sudo mkdir -p /opt/nginx_proxy/nginx-config
sudo mkdir -p /opt/nginx_proxy/letsencrypt
sudo mkdir -p /opt/nginx_proxy/html
```

## Docker Compose

```yaml
services:
  nginx:
    image: nginx:latest
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx-config:/etc/nginx/conf.d
      - ./letsencrypt:/etc/letsencrypt
      - ./html:/usr/share/nginx/html
      - webapps_media:/srv/webapps-media:ro
    networks:
      - suppliers_network
      - customers_network
      - internal_network
    restart: always

  certbot:
    image: certbot/certbot
    container_name: certbot
    entrypoint: >-
      /bin/sh -c 'trap exit TERM;
      while :; do
      certbot renew --webroot -w /usr/share/nginx/html;
      sleep 12h;
      done;'
    volumes:
      - ./letsencrypt:/etc/letsencrypt
      - ./html:/usr/share/nginx/html
    networks:
      - suppliers_network
      - customers_network
      - internal_network
    restart: always

networks:
  suppliers_network:
    external: true
  customers_network:
    external: true
  internal_network:
    external: true

volumes:
  webapps_media:
    external: true
```

Démarrer les services depuis le répertoire du projet :

```bash
cd /opt/nginx_proxy
docker compose up -d
```

Vérifier les conteneurs et les ports publiés :

```bash
docker ps --filter name=nginx_proxy
docker ps --filter name=certbot
```

## Création du vhost HTTP

Le nom du fichier doit obligatoirement se terminer par `.conf`, car Nginx
charge `/etc/nginx/conf.d/*.conf`.

```bash
sudo nano \
  /opt/nginx_proxy/nginx-config/admin.example.com.conf
```

Configuration temporaire :

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name admin.example.com;

    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

Tester puis recharger :

```bash
docker exec nginx_proxy nginx -t
docker exec nginx_proxy nginx -s reload
```

## Validation du webroot

Avant de lancer Certbot, vérifier que le même fichier est visible sur l'hôte,
dans Nginx et depuis Internet :

```bash
sudo mkdir -p \
  /opt/nginx_proxy/html/.well-known/acme-challenge

echo "test-acme" |
sudo tee /opt/nginx_proxy/html/.well-known/acme-challenge/test
```

```bash
docker exec nginx_proxy \
  cat /usr/share/nginx/html/.well-known/acme-challenge/test
```

```bash
curl -4 \
  http://admin.example.com/.well-known/acme-challenge/test
```

Les deux commandes doivent afficher `test-acme`.

Si le domaine publie un enregistrement `AAAA`, tester également IPv6 :

```bash
curl -6 \
  http://admin.example.com/.well-known/acme-challenge/test
```

Let's Encrypt privilégie IPv6 lorsqu'un enregistrement `AAAA` existe. Ne
publier cet enregistrement que si le port 80 est réellement accessible en
IPv6.

## Création du certificat

Utiliser des chemins absolus afin que la commande reste correcte quel que soit
le répertoire courant :

```bash
docker run --rm \
  -v /opt/nginx_proxy/letsencrypt:/etc/letsencrypt \
  -v /opt/nginx_proxy/html:/usr/share/nginx/html \
  certbot/certbot certonly --webroot \
  --webroot-path=/usr/share/nginx/html \
  -d admin.example.com \
  --email admin@example.com \
  --agree-tos \
  --non-interactive
```

Vérifier le certificat :

```bash
sudo openssl x509 \
  -in /opt/nginx_proxy/letsencrypt/live/admin.example.com/fullchain.pem \
  -noout -subject -issuer -dates
```

## Activation HTTPS

Remplacer le vhost temporaire par :

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name admin.example.com;

    location /.well-known/acme-challenge/ {
        root /usr/share/nginx/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    server_name admin.example.com;

    ssl_certificate /etc/letsencrypt/live/admin.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/admin.example.com/privkey.pem;

    location / {
        proxy_pass http://application:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Le conteneur applicatif doit partager au moins un réseau Docker avec
`nginx_proxy`.

Tester et appliquer :

```bash
docker exec nginx_proxy nginx -t
docker exec nginx_proxy nginx -s reload
curl -I https://admin.example.com
```

## Médias Django

Le conteneur Django doit monter le volume externe :

```yaml
services:
  application:
    volumes:
      - webapps_media:/app/media

volumes:
  webapps_media:
    external: true
```

Configuration Django :

```python
MEDIA_URL = "/media/"
MEDIA_ROOT = "/app/media"
```

Bloc à ajouter au vhost HTTPS :

```nginx
location /media/ {
    alias /srv/webapps-media/;
    autoindex off;
}
```

## Renouvellement

Le conteneur Certbot vérifie les certificats toutes les 12 heures :

```bash
docker logs certbot
```

Test manuel :

```bash
docker exec certbot certbot renew --dry-run
```

Important : `certbot renew` remplace les fichiers sur disque, mais Nginx doit
ensuite être rechargé pour présenter le nouveau certificat :

```bash
docker exec nginx_proxy nginx -t
docker exec nginx_proxy nginx -s reload
```

Le Compose présenté ci-dessus ne déclenche pas automatiquement ce rechargement
inter-conteneurs. Il faut ajouter un mécanisme de rechargement automatique ou
effectuer cette opération après chaque renouvellement.

## Dépannage

Afficher la configuration réellement chargée :

```bash
docker exec nginx_proxy nginx -T
```

Si aucun `listen` ou `server_name` n'apparaît, vérifier que les fichiers de
`nginx-config` se terminent par `.conf`.

Afficher les montages :

```bash
docker inspect nginx_proxy \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Les montages attendus commencent par :

```text
/opt/nginx_proxy/nginx-config -> /etc/nginx/conf.d
/opt/nginx_proxy/letsencrypt -> /etc/letsencrypt
/opt/nginx_proxy/html -> /usr/share/nginx/html
```

Si un répertoire bind-mounté est supprimé puis recréé pendant que les
conteneurs fonctionnent, recréer les conteneurs :

```bash
cd /opt/nginx_proxy
docker compose down
docker compose up -d
```

Autres commandes utiles :

```bash
docker exec nginx_proxy nginx -t
docker logs nginx_proxy
docker logs certbot
sudo ss -lntp | grep -E ':(80|443)[[:space:]]'
```

## Auteur

**Sébastien HERLANT**
