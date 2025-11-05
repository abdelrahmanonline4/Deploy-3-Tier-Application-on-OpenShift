FROM nginx:alpine

RUN apk add --no-cache openssl bash

COPY generate-ssl.sh /

RUN chmod +x generate-ssl.sh
COPY ./nginx.conf /etc/nginx/nginx.conf

CMD ["/bin/bash", "-c", "./generate-ssl.sh && nginx -g 'daemon off;'"]



EXPOSE 443