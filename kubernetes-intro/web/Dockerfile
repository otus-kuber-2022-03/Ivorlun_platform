# https://docs.nginx.com/nginx/deployment-guides/setting-up-nginx-demo-environment/

FROM nginx:1.21

RUN usermod -u 1001 nginx && \
    chown -R nginx:nginx /var/cache/nginx/ && \
    chown -R nginx:nginx /var/log/nginx/ && \
    touch /var/run/nginx.pid &&\
    chown nginx:nginx /var/run/nginx.pid

COPY homework.html /app/homework.html
COPY nginx.conf /etc/nginx/conf.d/default.conf

USER 1001
EXPOSE 8000
