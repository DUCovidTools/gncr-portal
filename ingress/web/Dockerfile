FROM nginx
#COPY nginx.conf /etc/nginx/nginx.conf
COPY default.conf /etc/nginx/conf.d/
#COPY --chown=nginx htpasswd /etc/nginx/.htpasswd
#RUN chmod 600 /etc/nginx/.htpasswd
#EXPOSE 80

COPY index.html /usr/share/nginx/html
COPY images /usr/share/nginx/html/images
COPY css /usr/share/nginx/html/css
COPY js /usr/share/nginx/html/js