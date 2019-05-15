#
# Dockerfile to build a MISP (https://github.com/MISP/MISP) container
#
# Original docker file by eg5846 (https://github.com/eg5846)
#
# 2016/03/03 - First release
# 2017/06/02 - Updated
# 2018/04/04 - Added objects templates
#

# We are based on Ubuntu:latest
FROM ubuntu:xenial
MAINTAINER Xavier Mertens <xavier@rootshell.be>

ENV DEBIAN_FRONTEND noninteractive

# Install core components
RUN apt-get update
RUN apt-get dist-upgrade -y 
RUN apt-get install -y locales software-properties-common postfix cron logrotate supervisor syslog-ng-core mysql-client curl gcc git gnupg-agent make python openssl sudo vim zip 
#RUN apt-get install -y redis-server 

# set local for the apt repository to work
RUN locale-gen en_US.UTF-8
ENV LANG en_US.UTF-8

RUN add-apt-repository -y ppa:ondrej/php 
RUN apt-get update

# install apache
RUN apt-get install -y apache2 apache2-doc apache2-utils

# PHP 7.3
RUN apt-get install -y libapache2-mod-php php7.3 php7.3-cli php-crypt-gpg php7.3-dev php7.3-gd php7.3-json php7.3-mysql php7.3-opcache php7.3-readline php7.3-redis php7.3-xml php-pear pkg-config libbson-1.0 libmongoc-1.0-0 php-xml php-dev
# Python requirements
RUN apt-get install -y python-dev python-pip libxml2-dev libxslt1-dev zlib1g-dev python-setuptools libfuzzy-dev python3 python3-pip

# Install MISP build requirements
RUN sudo apt-get -y install libpoppler58 libpoppler-dev libpoppler-cpp-dev libjpeg-dev
# clean out some caches 
RUN apt-get clean
RUN apt-get autoremove -y

# PIP fix
USER root 
RUN sudo pip install --upgrade pip
RUN pip install --upgrade setuptools
RUN pip3 install --upgrade pip
# update setuptools because python-docx was failing
RUN pip3 install --upgrade setuptools

# enable redis php module
USER root
RUN phpenmod redis

# Preconfigure setting for packages
RUN echo "postfix postfix/main_mailer_type string Local only" | debconf-set-selections
RUN echo "postfix postfix/mailname string localhost.localdomain" | debconf-set-selections

# Fix php.ini with recommended settings
RUN sed -i "s/max_execution_time = 30/max_execution_time = 300/" /etc/php/7.3/apache2/php.ini
RUN sed -i "s/memory_limit = 128M/memory_limit = 512M/" /etc/php/7.3/apache2/php.ini
RUN sed -i "s/upload_max_filesize = 2M/upload_max_filesize = 50M/" /etc/php/7.3/apache2/php.ini
RUN sed -i "s/post_max_size = 8M/post_max_size = 50M/" /etc/php/7.3/apache2/php.ini

# base MISP install
USER root
WORKDIR /var/www
RUN chown www-data:www-data /var/www

USER www-data
RUN git clone https://github.com/MISP/MISP.git

WORKDIR /var/www/MISP
RUN git checkout tags/$(git describe --tags `git rev-list --tags --max-count=1`)
RUN git config core.filemode false
RUN git submodule update --init --recursive
RUN git submodule foreach --recursive git config core.filemode false

WORKDIR /var/www/MISP/app/files/scripts
RUN git clone https://github.com/CybOXProject/python-cybox.git
RUN git clone https://github.com/STIXProject/python-stix.git

WORKDIR /var/www/MISP/app/files/scripts/python-cybox
RUN git checkout v2.1.0.12
USER root
RUN python setup.py install

USER www-data
WORKDIR /var/www/MISP/app/files/scripts/python-stix
RUN git checkout v1.1.1.4

USER root
RUN python setup.py install

USER www-data
WORKDIR /var/www/MISP
RUN git submodule init
RUN git submodule update

# Install PHP Composer
WORKDIR /var/www/MISP/app
RUN curl -o composer-setup.php https://getcomposer.org/installer
RUN php -r "if (hash_file('sha384', 'composer-setup.php') === '48e3236262b34d30969dca3c37281b3b4bbe3221bda826ac6a9a62d6444cdb0dcd0615698a5cbe587c3f0fe57a54d8f5') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
# cancel the build if the hash doesn't match
RUN bash -c "if [ ! -f composer-setup.php ]; then false; fi"

RUN php composer.phar config vendor-dir Vendor
RUN php composer-setup.php
RUN php composer.phar install --ignore-platform-reqs
RUN rm -f composer-setup.php

USER www-data
RUN cp -fa /var/www/MISP/INSTALL/setup/config.php /var/www/MISP/app/Plugin/CakeResque/Config/config.php

# Fix permissions
USER root
RUN chown -R www-data:www-data /var/www/MISP
RUN chmod -R 750 /var/www/MISP
RUN chmod -R g+ws /var/www/MISP/app/tmp
RUN chmod -R g+ws /var/www/MISP/app/files
RUN chmod -R g+ws /var/www/MISP/app/files/scripts/tmp

RUN cp /var/www/MISP/INSTALL/misp.logrotate /etc/logrotate.d/misp

# Redis Setup
#RUN sed -i 's/^\(daemonize\s*\)yes\s*$/\1no/g' /etc/redis/redis.conf
# update misp config to point at misp_redis
RUN sed -i -e 's/127\.0\.0\.1/misp_redis/' /var/www/MISP/app/Model/AppModel.php

# Install PEAR packages
RUN pear channel-update pear.php.net
RUN pear install Crypt_GPG >> /tmp/install.log
RUN pear install Net_GeoIP >> /tmp/install.log

# Apache Setup
RUN cp /var/www/MISP/INSTALL/apache.misp.ubuntu /etc/apache2/sites-available/misp.conf
RUN a2dismod status
RUN a2enmod rewrite
RUN a2enmod headers
RUN a2dissite 000-default
RUN a2ensite misp

# MISP base configuration
USER www-data
RUN cp -a /var/www/MISP/app/Config/bootstrap.default.php /var/www/MISP/app/Config/bootstrap.php
RUN cp -a /var/www/MISP/app/Config/database.default.php /var/www/MISP/app/Config/database.php
RUN cp -a /var/www/MISP/app/Config/core.default.php /var/www/MISP/app/Config/core.php
RUN cp -a /var/www/MISP/app/Config/config.default.php /var/www/MISP/app/Config/config.php
USER root
RUN chown -R www-data:www-data /var/www/MISP/app/Config
RUN chmod -R 750 /var/www/MISP/app/Config

# Replace the default salt
RUN sed -i -E "s/'salt'\s=>\s'(\S+)'/'salt' => '`openssl rand -base64 32|tr "/" "-"`'/" /var/www/MISP/app/Config/config.php

# Enable workers at boot time
RUN chmod a+x /var/www/MISP/app/Console/worker/start.sh
RUN echo "sudo -u www-data bash /var/www/MISP/app/Console/worker/start.sh" >> /etc/rc.local

# Install templates & stuff
WORKDIR /var/www/MISP/app/files
RUN rm -rf misp-objects && git clone https://github.com/MISP/misp-objects.git
RUN rm -rf misp-galaxy && git clone https://github.com/MISP/misp-galaxy.git
RUN rm -rf warninglists && git clone https://github.com/MISP/misp-warninglists.git ./warninglists
RUN rm -rf taxonomies && git clone https://github.com/MISP/misp-taxonomies.git ./taxonomies
RUN chown -R www-data:www-data misp-objects misp-galaxy warninglists taxonomies

# Install MISP Modules
USER root
RUN git clone https://github.com/MISP/misp-modules.git /opt/misp-modules

WORKDIR /opt/misp-modules
RUN pip3 install --upgrade --ignore-installed urllib3
RUN pip3 install --upgrade --ignore-installed requests

# fix version requirements
RUN sed -i 's/functools.*//g' REQUIREMENTS
RUN sed -i 's/\(aiohttp\)\=.*/\1/g' REQUIREMENTS
RUN sed -i 's/\(async-timeout\)\=.*/\1/g' REQUIREMENTS
RUN sed -i 's/\(url-normalize\)\=.*/\1/g' REQUIREMENTS
RUN sed -i 's/^\(yarl\)\=.*/\1/g' REQUIREMENTS
RUN sed -i 's/^\(sigmatools\)\=.*/\1/g' REQUIREMENTS
# install python packages
RUN pip3 install -I -r REQUIREMENTS
RUN pip3 install -I .
# install extra requirements
RUN pip3 install git+https://github.com/cyboxproject/python-cybox
RUN pip3 install stix maec stix2 pydeep python-magic lief

# start misp-modules on startup 
RUN echo "sudo -u www-data misp-modules -s -l 127.0.0.1 &" >> /etc/rc.local 

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Modify syslog configuration
RUN sed -i -E 's/^(\s*)system\(\);/\1unix-stream("\/dev\/log");/' /etc/syslog-ng/syslog-ng.conf

# Add run script
ADD run.sh /run.sh
RUN chmod 0755 /run.sh

# Trigger to perform first boot operations
RUN touch /.firstboot.tmp

# Make a backup of /var/www/MISP to restore it to the local moint point at first boot
WORKDIR /var/www/MISP
RUN tar czpf /root/MISP.tgz .

VOLUME /var/www/MISP
EXPOSE 80
ENTRYPOINT ["/run.sh"]