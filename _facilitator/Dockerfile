FROM nginx:1-alpine

COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./src /usr/share/nginx/html

RUN chmod -R 755 /usr/share/nginx/html
