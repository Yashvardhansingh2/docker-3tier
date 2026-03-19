FROM nginx:alpine

COPY nginx.conf.template /etc/nginx/templates/default.conf.template
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["/docker-entrypoint.sh", "nginx", "-g", "daemon off;"]
