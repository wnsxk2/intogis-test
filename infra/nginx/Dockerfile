ARG NGINX_VERSION=stable-alpine
FROM nginx:${NGINX_VERSION}

RUN rm /etc/nginx/conf.d/default.conf
COPY ./conf.d/default.conf /etc/nginx/conf.d/default.conf

# 타임존 설정 (Alpine OS 기준)
RUN apk add --no-cache tzdata && \
    cp /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
