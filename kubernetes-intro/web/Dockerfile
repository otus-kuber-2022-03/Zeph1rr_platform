FROM nginx:stable

RUN mkdir /app

WORKDIR /app

RUN usermod -u 1001 nginx \
  && groupmod -g 1001 nginx
RUN chown -R nginx:nginx /app && chmod -R 755 /app && \
        chown -R nginx:nginx /var/cache/nginx && \
        chown -R nginx:nginx /var/log/nginx && \
        chown -R nginx:nginx /etc/nginx/conf.d
RUN touch /var/run/nginx.pid && \
        chown -R nginx:nginx /var/run/nginx.pid

USER nginx

RUN sed -i 's/80/8000/g' /etc/nginx/conf.d/default.conf

RUN sed -i 's/\/usr\/share\/nginx\/html/\/app/g' /etc/nginx/conf.d/default.conf

EXPOSE 8000

CMD ["nginx", "-g", "daemon off;"]

