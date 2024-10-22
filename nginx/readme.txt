reverse proxy avec docker-compose
tuto chaine youtube korben

creer le dossier loadbalancer-docker-compose
créer à l'intérieur le fichier docker-compose.yaml 
/home/hennebo/loadbalancer-docker-compose/
docker-compose.yaml

│cat docker-compose.yaml
version: '3.1'

services:
  reverseproxy:
    image: nginx:latest
    container_name: nginxrp
    volumes:
      - ./nginx/core.conf:/etc/nginx/conf.d/core.conf
      - ./nginx/proxy.conf:/etc/nginx/conf.d/proxy.conf
      - ./nginx/wordpress1.conf:/etc/nginx/conf.d/wordpress1.conf
      - ./nginx/wordpress2.conf:/etc/nginx/conf.d/wordpress2.conf
      - ./nginx/html:/var/www/html
      - ./nginx/logs:/var/log/nginx
      - /etc/letsencrypt/live/mon-loadbalancer.julien-hennebo.cloudns.be:/etc/letsencrypt/live/mon-loadbalancer.julien-hennebo.cloudns.be
      - /etc/letsencrypt/archive/mon-loadbalancer.julien-hennebo.cloudns.be:/etc/letsencrypt/archive/mon-loadbalancer.julien-hennebo.cloudns.be
    ports:
      - "80:80"
      - "443:443"

  web1:
    image: wordpress:latest
    container_name: wordpress1
    volumes:
      - ./wordpress/wp1:/var/www/html
    environment:
      - WORDPRESS_DB_HOST=db1
      - WORDPRESS_DB_USER=wp1
      - WORDPRESS_DB_PASSWORD=wordpress1234
      - WORDPRESS_DB_NAME=wordpress1

  web2:
    image: wordpress:latest
    container_name: wordpress2
    volumes:
      - ./wordpress/wp2:/var/www/html
    environment:
      - WORDPRESS_DB_HOST=db2
      - WORDPRESS_DB_USER=wp2
      - WORDPRESS_DB_PASSWORD=wordpress1234
      - WORDPRESS_DB_NAME=wordpress2

  db1:
    image: mysql:latest
    container_name: db1
    volumes:
      - ./mysql/db1:/var/lib/mysql
      - ./mysql/conf1.d:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: "root1234"
      MYSQL_DATABASE: "wordpress1"
      MYSQL_USER: "wp1"
      MYSQL_PASSWORD: "wordpress1234"

  db2:
    image: mysql:latest
    container_name: db2
    volumes:
      - ./mysql/db2:/var/lib/mysql
      - ./mysql/conf2.d:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: "root1234"
      MYSQL_DATABASE: "wordpress2"
      MYSQL_USER: "wp2"
      MYSQL_PASSWORD: "wordpress1234"


Au préalable il faut avoir crée les fichiers suivant (dans le meme dossier)
ce qui correspond aux volumes renseignés dans le fichier docker-compose.yaml
exemple;
./nginx/html:/var/www/html
Répertoire source: ./nginx/html (répertoire qui doit exister au préalable sur la machine hôte).
Répertoire cible: /var/www/html (répertoire crée dans le conteneur Nginx lors de docker-compose up).


├── docker-compose.yml
├── nginx/
│   ├── core.conf
│   ├── proxy.conf
│   ├── wordpress1.conf
│   └── wordpress2.conf
│   ├── html
│   └── logs
├── wordpress/
│   ├── wp1
│   ├── wp2
└── mysql/
    ├── db1/           # Répertoire pour les données MySQL de la base 1
    ├── conf1.d/       # Répertoire pour les fichiers de config MySQL de la base 1
    ├── db2/           # Répertoire pour les données MySQL de la base 2
    └── conf2.d/       # Répertoire pour les fichiers de config MySQL de la base 2
    
    
    docker-compose up -d
    Cela lancera tous les services définis dans le fichier docker-compose.yaml
    Nombre total de services: 5 (1 reverse proxy + 2 WordPress + 2 MySQL).
    Rôle de chaque service:
    Le reverse proxy (Nginx) dirige les requêtes vers les bonnes instances de WordPress.
    Chaque instance de WordPress sert un site distinct, avec sa propre base de données MySQL.
    Chaque base de données MySQL stocke les données pour l'une des instances de WordPress.

docker-compose restart reverseproxy     
reverseproxy est le nom donné ici  au service dans le fichier docker-compose   

exemple de contenu dans le fichier core.conf
loadbalancer-docker-compose/nginx# cat core.conf
# Configuration pour le port HTTP (80)
server {
    listen 80;
    server_name mon-loadbalancer.julien-hennebo.cloudns.be;

    # Rediriger tout le trafic HTTP vers HTTPS
    return 301 https://$host$request_uri;
}

# Configuration pour le port HTTPS (443)
server {
    listen 443 ssl;
    server_name mon-loadbalancer.julien-hennebo.cloudns.be;

    # Chemins vers les certificats SSL
    ssl_certificate /etc/letsencrypt/live/mon-loadbalancer.julien-hennebo.cloudns.be/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mon-loadbalancer.julien-hennebo.cloudns.be/privkey.pem;

    # Configuration des fichiers de journalisation (facultatif)
    access_log /var/log/nginx/mon-loadbalancer_access.log;
    error_log /var/log/nginx/mon-loadbalancer_error.log debug;

    # Redirection vers WordPress1
    location /wordpress1/ {
        proxy_pass http://wordpress1;
        include /etc/nginx/conf.d/proxy.conf;
    }

    # Redirection vers WordPress2
    location /wordpress2/ {
        proxy_pass http://wordpress2;
        include /etc/nginx/conf.d/proxy.conf;
    }

    # Gérer la racine pour des fichiers statiques
    location / {
        root /var/www/html;
        index index.html index.htm;

        # Page d'erreur 404 personnalisée
        error_page 404 /404.html;
    }

    # Gérer les pages d'erreur
    location = /404.html {
        root /var/www/html;
        internal;
  # Indique que cette page ne peut pas être accédée directement
    }
}

Dans le fichier , on place 
  GNU nano 4.8                                      proxy.conf                                                
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_buffering off;
proxy_request_buffering off;
proxy_http_version 1.1;
proxy_intercept_errors on;

# wordpress1.conf
server {
    listen 80;
    server_name wordpress1.local;

    location / {
        proxy_pass http://wordpress1:80;
    }
}

# wordpress2.conf
server {
    listen 80;
    server_name wordpress2.local;

    location / {
        proxy_pass http://wordpress2:80;
    }
}

https://mon-loadbalancer.julien-hennebo.cloudns.be/wordpress1/
et 
https://mon-loadbalancer.julien-hennebo.cloudns.be/wordpress2/

docker-compose up -d




   
