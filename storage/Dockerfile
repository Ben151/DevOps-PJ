FROM httpd:2.4-alpine

COPY plates.conf /usr/local/apache2/conf/extra/plates.conf

RUN echo "Include conf/extra/plates.conf" >> /usr/local/apache2/conf/httpd.conf \
 && mkdir -p /usr/local/apache2/htdocs/plates
