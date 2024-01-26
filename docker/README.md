## Docker





## Docker-Compose

### download

```shell
# intel x86_64
sudo curl -SL https://github.com/docker/compose/releases/download/${your_version}/docker-compose-linux-x86_64 \
          -o /usr/local/bin/docker-compose

# apple m1
sudo curl -SL https://github.com/docker/compose/releases/download/${your_version}/docker-compose-linux-aarch64 \
          -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```



`docker-compose version`



### service

```shell
docker run -d -p 5000:5000 registry
```



reg-compose.yml

```yaml
services:
  registry:
    image: registry
    container_name: registry
    restart: always
    ports:
      - 5000:5000
```

`docker-compose up -d`

```shell
docker-compose -f reg-compose.yml up -d
```

`docker-compose ps`



`docker-compose down`



### demo

wp-compose.yml

```yaml
services:
  
  mariadb:
    image: mariadb:10
    container_name: mariadb
    restart: always
    
    environment:
      MARIADB_DATABASE: db
      MARIADB_USER: wp
      MARIADB_PASSWORD: 123
      MARIADB_ROOT_PASSWORD: 123
      
  wordpress:
    image: wordpress:5
    container_name: wordpress
    restart: always
    
    environment:
      WORDPRESS_DB_HOST: mariadb #container_name
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: 123
      WORDPRESS_DB_NAME: db
      
    depends_on:
        - mariadb #container_name
  
  nginx:
    image: nginx:alpine
    container_name: nginx
    hostname: nginx
    restart: always
    ports:
      - 80:80
    volumes:
      - ./wp.conf:/etc/nginx/conf.d/default.conf
    
    depends_on:
      - wordpress #container_name
```



`mariadb`

```shell
docker run -d --rm \
    --env MARIADB_DATABASE=db \
    --env MARIADB_USER=wp \
    --env MARIADB_PASSWORD=123 \
    --env MARIADB_ROOT_PASSWORD=123 \
    mariadb:10
```

wp.conf

```json
server {
  listen 80;
  default_type text/html;

  location / {
      proxy_http_version 1.1;
      proxy_set_header Host $host;
      proxy_pass http://wordpress;  #注意这里，网站的网络标识
  }
}
```

