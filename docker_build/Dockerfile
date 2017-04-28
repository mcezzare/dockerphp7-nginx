#
# Debian with apache and PHP7
# 
#
FROM debian:stable

MAINTAINER Mario Cezzare Angelicola Chiodi <mcezzare@gmail.com>

#Suppress messages like "debconf: unable to initialize frontend: Dialog"
ENV TERM linux
#Let the conatiner know that there is no tty
ENV DEBIAN_FRONTEND noninteractive

#0 
RUN apt-get update && apt-get install -y curl wget
RUN apt-get install -y vim
#1: Initial setup: get nginx
RUN curl http://nginx.org/keys/nginx_signing.key -o nginx_signing.key
RUN apt-key add nginx_signing.key
RUN echo "deb http://nginx.org/packages/mainline/debian/ jessie nginx" >> /etc/apt/sources.list.d/nginx.list \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y \
                        ca-certificates \
                        nginx \
                        nginx-module-xslt \
                        nginx-module-geoip \
                        nginx-module-image-filter \
                        nginx-module-perl \
                        nginx-module-njs \
                        gettext-base 
#2: add dotdeb packages
RUN apt-get update \
  && echo 'deb http://packages.dotdeb.org jessie all' >> /etc/apt/sources.list.d/dotdeb.list \
  && echo 'deb-src http://packages.dotdeb.org jessie all' >> /etc/apt/sources.list.d/dotdeb.list \
  && wget https://www.dotdeb.org/dotdeb.gpg \
  && apt-key add dotdeb.gpg \
  && apt-get update 

#3: php7 packages, separated command to preserve layers already cached 
RUN apt-get install -y php7.0-mysql php7.0-mcrypt php7.0-json php7.0-curl \
php7.0-cli php7.0-gd php7.0-imagick php7.0-xdebug php7.0-opcache php7.0-fpm \
php7.0-bcmath php7.0-mbstring php7.0-xml php7.0-zip

#EXTRA STUFF, if need redis,memcached, etc
RUN apt-get install -y php7.0-redis

#For php tools
RUN apt-get install zip unzip git curl -yqq
RUN curl --silent --show-error https://getcomposer.org/installer | php
RUN mv composer.phar /usr/local/bin/composer && chmod +x /usr/local/bin/composer
RUN curl -sSLo /usr/local/bin/phpunit https://phar.phpunit.de/phpunit.phar && chmod +x /usr/local/bin/phpunit
#End php tools

#4: forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log

#5: install supervisord
RUN apt-get install --no-install-recommends --no-install-suggests -y supervisor

#6: Global configurations
RUN echo "America/Sao_Paulo" > /etc/timezone && \
    dpkg-reconfigure -f noninteractive tzdata

#7: Nginx Configuration
COPY ./config/nginx.conf /etc/nginx/nginx.conf
COPY ./config/nginx-modules.conf /etc/nginx/modules.conf
COPY ./config/fastcgi_params /etc/nginx/fastcgi_params
RUN mkdir -p /etc/nginx/sites-enabled/
COPY ./config/localhost.conf /etc/nginx/sites-enabled/localhost.conf
# COPY ./config/default.conf /etc/nginx/conf.d/default.conf
COPY ./config/php.ini /etc/php/7.0/fpm/php.ini

#8: SupervisorD Configuration
COPY ./config/supervisor.conf /etc/supervisor/conf.d/supervisor.conf

#9: PHP-FPM Configuration
RUN mkdir -p /run/php &&\
    sed -i "s|;daemonize = yes|daemonize = no|"                     /etc/php/7.0/fpm/php-fpm.conf && \
    sed -i "s|;date.timezone =|date.timezone = America\/Sao_Paulo|" /etc/php/7.0/cli/php.ini && \
    sed -i "s|;date.timezone =|date.timezone = America\/Sao_Paulo|" /etc/php/7.0/fpm/php.ini && \
    # Xdebug configuration
    mkdir -p /etc/php/mods-available && \
    echo "zend_extension=xdebug.so"     >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.max_nesting_level=250" >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.remote_enable=1"       >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.remote_handler=dbgp"   >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.remote_mode=req"       >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.remote_port=9000"      >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.remote_autostart=1"    >> /etc/php/mods-available/xdebug.ini && \
    echo "xdebug.remote_connect_back=1" >> /etc/php/mods-available/xdebug.ini && \
    # Enable Xdebug
    ln -nfs /etc/php/mods-available/xdebug.ini /etc/php/7.0/fpm/conf.d/20-xdebug.ini && \
    ln -nfs /etc/php/mods-available/xdebug.ini /etc/php/7.0/cli/conf.d/20-xdebug.ini


#     rm /etc/nginx/sites-enabled/default && \
#     ln -nfs /etc/nginx/sites-available/mercadoinca.conf /etc/nginx/sites-enabled/mercadoinca.conf && \
RUN mkdir -p /var/www && \
    #usermod -u 1000 www-data && \
    chown -R www-data:www-data /var/www/ && \
    chmod -R g+s /var/www && \
    chown -R www-data /run/php/

#10: Setup Home Dir if log on container 
WORKDIR /var/www/html


#12: remove the apt lists to reduce image size
#RUN rm -rf /var/lib/apt/lists/*

#13: Expose port
EXPOSE 80

#14: Start SupervisorD with nginx and php-fpm
ENTRYPOINT ["/usr/bin/supervisord"]