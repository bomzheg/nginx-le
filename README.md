# nginx-le
 Configuration nginx as reverse proxy with Let's Encrypt in docker container

How to use:

install docker and docker-compose

docker create network nginx-reverse-proxy

docker-compose pull

docker-compose up -d

if need add bot or anoher service: just add in path etc/bots file "some_bot.conf" as like example lyt_poster and run docker-compose exec nginx nginx -s reload
