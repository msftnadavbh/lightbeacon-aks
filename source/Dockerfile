## Build a Container running a PHP page
FROM ubuntu:18.04
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get update -y && apt-get upgrade -y && apt-get install apache2 -y && apt-get install php -y
COPY test.php /var/www/html
COPY 000-default.conf /etc/apache2/sites-available/
EXPOSE 80
CMD apachectl -D FOREGROUND
