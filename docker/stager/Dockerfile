FROM irodsclient

# application metadata
LABEL donders.ru.nl.app_name "stager"
LABEL donders.ru.nl.app_maintainer "h.lee@donders.ru.nl"
LABEL donders.ru.nl.app_code_repository "https://github.com/Donders-Institute/data-stager"

# install cyberduck command-line interface
RUN ( echo -e "[duck-stable]\n\
name=duck-stable\n\
baseurl=https://repo.cyberduck.io/stable/\$basearch/\n\
enabled=1\n\
gpgcheck=0" > /etc/yum.repos.d/duck-stable.repo )

RUN ( yum -y install duck )

# create temporary directory
RUN ( mkdir -p /tmp )
WORKDIR /tmp

# install python 2.7.12 and pycurl (as the system pycurl seems to have problem with data encoding)
ADD https://www.python.org/ftp/python/2.7.12/Python-2.7.12.tgz /tmp
RUN ( if [ ! -d Python-2.7.12 ]; then tar xvf Python-2.7.12.tgz; fi \
      && cd Python-2.7.12 \
      && ./configure --prefix=/opt/python \
      && make && make install && rm -rf /tmp/Python-* )

ADD https://dl.bintray.com/pycurl/pycurl/pycurl-7.43.0.tar.gz /tmp
RUN ( if [ ! -d pycurl-7.43.0 ]; then tar xvf pycurl-7.43.0.tar.gz; fi \
      && cd pycurl-7.43.0 \
      && /opt/python/bin/python setup.py --with-nss install && rm -rf /tmp/pycurl-* )

# install nodejs
ADD https://nodejs.org/dist/v6.11.2/node-v6.11.2-linux-x64.tar.xz /tmp
RUN ( if [ ! -d node-v6.11.2-linux-x64 ]; then tar xf node-v6.11.2-linux-x64.tar.xz; fi \
      && mv node-v6.11.2-linux-x64 /opt/nodejs && rm -rf /tmp/node-*)

# install expect so that we get unbuffer to get irsync progress, progressively
RUN ( yum -y install expect parallel bc samba-client cifs-utils )

# install stager scripts and required nodejs modules
RUN ( mkdir -p /opt/stager/bin )
WORKDIR /opt/stager
COPY package.json package.json
RUN ( export PATH=/opt/nodejs/bin:$PATH && export USER=root && /opt/nodejs/bin/npm install -g node-gyp && /opt/nodejs/bin/npm install --unsafe-perm )
COPY *.js ./
COPY bin ./bin
COPY lib ./lib
COPY routes ./routes
COPY start_stager.sh start_stager.sh
COPY config_stager.py config_stager.py
RUN ( chmod +x start_stager.sh )
RUN ( chmod +x config_stager.py )

# install python pip
RUN ( wget https://bootstrap.pypa.io/get-pip.py && \
      /opt/python/bin/python get-pip.py )

# install devcron
#RUN ( yum -y install mercurial )
#RUN ( /opt/python/bin/pip install -e hg+https://bitbucket.org/dbenamy/devcron#egg=devcron )
RUN ( /opt/python/bin/pip install -e git+https://github.com/dbenamy/devcron#egg=devcron )
# copy default crontab into container
RUN ( mkdir -p /cron )
COPY cron /cron

# install Cheetah template engine
# force Markdown library to version 3.1, the last version supporting python 2.7
RUN ( /opt/python/bin/pip install Markdown==3.1 )
RUN ( wget https://pypi.python.org/packages/cd/b0/c2d700252fc251e91c08639ff41a8a5203b627f4e0a2ae18a6b662ab32ea/Cheetah-2.4.4.tar.gz#md5=853917116e731afbc8c8a43c37e6ddba && tar xvzf Cheetah-2.4.4.tar.gz && \
      cd Cheetah-2.4.4 && /opt/python/bin/python setup.py install )

# install supervisor
RUN ( yum -y install epel-release )
RUN ( yum -y install supervisor )
COPY supervisord.conf /opt/stager/supervisord.conf

RUN ( yum -y clean all )

# runtime environment variables
ENV IRODS_ADMIN_USER=irods
ENV IRODS_ADMIN_CREDENTIAL=/opt/stager/config/secret
ENV IRODS_ICAT_HOST=irods-icat
ENV IRODS_ZONE_PORT=1247
ENV IRODS_ZONE_NAME=rdmtst
ENV IRODS_REST_ENDPOINT=https://irods-rest
ENV DAVRODS_ENDPOINT=https://irods-webdav
ENV DAVRODS_EXPOSED_ROOT=/rdmtst/di

# run a stager and devcron from supervisor
VOLUME [ "/project", "/cron", "/opt/stager/config", "/opt/stager/log" ]
EXPOSE 3000

RUN ( echo "export PYTHON_BINDIR=/opt/python/bin" > /opt/stager/envvars && \
      echo "export PYTHON_LIBDIR=/opt/stager/lib/python" >> /opt/stager/envvars && \
      echo "export NODEJS_PREFIX=/opt/nodejs" >> /opt/stager/envvars )

CMD [ "/usr/bin/supervisord", "-c", "/opt/stager/supervisord.conf" ]
