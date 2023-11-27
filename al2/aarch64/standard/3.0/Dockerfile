# Copyright 2020-2020 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.

FROM public.ecr.aws/amazonlinux/amazonlinux:2023 AS core


# Install git, SSH, php, and other utilities
RUN set -ex \
    && yum install -yq openssh-clients \
    && mkdir ~/.ssh \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && rpm --import https://download.mono-project.com/repo/xamarin.gpg \
    && curl https://download.mono-project.com/repo/centos7-stable.repo | tee /etc/yum.repos.d/mono-centos7-stable.repo \
    && yum groupinstall -yq "Development tools" \
    && yum install -yq \
           ImageMagick asciidoc bzip2 bzip2-devel cvs cvsps \
           docbook-dtds docbook-style-xsl e2fsprogs expat-devel expect fakeroot \
           git glib2-devel groff gzip icu iptables jq krb5-server libargon2-devel \
           libcurl-devel libdb-devel libedit-devel libevent-devel libffi-devel \
           libjpeg-devel libpng-devel libserf \
           libtidy-devel libunwind libwebp-devel libxml2-devel libxslt libxslt-devel \
           libyaml-devel libzip-devel mlocate \
           ncurses-devel oniguruma-devel openssl openssl-devel perl-DBD-SQLite \
           perl-DBI perl-HTTP-Date perl-TimeDate perl-YAML-LibYAML \
           postgresql-devel procps-ng python-configobj readline-devel rsync sgml-common \
           patch pkg-config procps python3-configobj llvm rsync sqlite-devel \
           subversion-perl tar tcl tk vim wget which xfsprogs xmlto xorg-x11-server-Xvfb xz-devel \
           amazon-ecr-credential-helper libicu-devel libsodium-devel

RUN useradd codebuild-user

#=======================End of layer: core  =================

FROM core AS tools

# Install stunnel
RUN set -ex \
   && STUNNEL_VERSION=5.69 \
   && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
   && STUNNEL_SHA256="1ff7d9f30884c75b98c8a0a4e1534fa79adcada2322635e6787337b4e38fdb81" \
   && curl -o $STUNNEL_TAR https://www.stunnel.org/archive/5.x/$STUNNEL_TAR \
   && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum -c - \
   && tar xvfz $STUNNEL_TAR \
   && cd stunnel-$STUNNEL_VERSION \
   && ./configure \
   && make -j4 \
   && make install \
   && openssl genrsa -out key.pem 2048 \
   && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
   && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
   && cd .. ; rm -rf stunnel-${STUNNEL_VERSION}*

# AWS Tools
# https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.25.6/2023-01-30/bin/linux/arm64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.25.6/2023-01-30/bin/linux/arm64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-linux-arm64-latest \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli

# Configure SSM
RUN set -ex \
    && yum install -yq https://s3.amazonaws.com/amazon-ssm-us-east-1/3.0.1390.0/linux_arm64/amazon-ssm-agent.rpm

# NodeJS
ENV N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
     && cd $N_SRC_DIR && make install

# Python
RUN curl https://pyenv.run | bash
ENV PATH="/root/.pyenv/shims:/root/.pyenv/bin:$PATH"

# PHP
RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"

##awscliv2
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip -o /tmp/awscliv2.zip \
     && unzip /tmp/awscliv2.zip -d /opt \
     && /opt/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
     && rm /tmp/awscliv2.zip \
     && rm -rf /opt/aws \
     && aws --version

#=======================End of layer: tools  =================

FROM tools AS nodejs

# #****************      NODEJS     ****************************************************

ENV NODE_18_VERSION="18.18.2"

RUN  n $NODE_18_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && cd / && rm -rf $N_SRC_DIR; rm -rf /tmp/*

ENV NODE_20_VERSION="20.10.0"

RUN  n $NODE_20_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && cd / && rm -rf $N_SRC_DIR; rm -rf /tmp/*

# #****************      END NODEJS     ****************************************************

FROM nodejs AS python

#**************** PYTHON *****************************************************
ENV PYTHON_311_VERSION="3.11.2"
ENV PYTHON_PIP_VERSION=23.0.1
ENV PYYAML_VERSION=6.0

COPY tools/runtime_configs/python/$PYTHON_311_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_311_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_311_VERSION; rm -rf /tmp/*
RUN   pyenv global  $PYTHON_311_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.4.0' wheel aws-sam-cli boto3 pipenv virtualenv

#**************** END PYTHON *****************************************************

# FROM tools AS php
FROM python AS php

#****************      PHP     ****************************************************

# 8.x
ENV PHP_82_VERSION="8.2.13"

COPY tools/runtime_configs/php/$PHP_82_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_82_VERSION
# RUN yum install -yq libicu-devel libsodium-devel
RUN phpenv install $PHP_82_VERSION; phpenv rehash; rm -rf /tmp/*; phpenv global $PHP_82_VERSION
RUN which php
RUN echo $PATH
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_82_VERSION/etc/conf.d/memory.ini"

# Install Composer globally
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer


#****************      END PHP     ****************************************************

FROM php AS docker

#Docker 2.17
ENV DOCKER_BUCKET="download.docker.com" \
    DOCKER_CHANNEL="stable" \
    DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034"

ENV DOCKER_SHA256="3865F837DBD951B19EEB5F7D87AADA2E865B2017E9462FE389F0E5D9A438324D"
ENV DOCKER_VERSION="23.0.1"
ENV DOCKER_COMPOSE_VERSION="2.17.2"

VOLUME /var/lib/docker

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/aarch64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin \
    && rm docker.tgz \
    && docker -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && groupadd dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-aarch64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
    # Ensure docker-compose works
    && docker-compose version

#===================END of runtimes_3 ==============
FROM docker AS aarch64_v2

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY dockerd-entrypoint.sh /usr/local/bin/
COPY legal/THIRD_PARTY_LICENSES.txt /usr/share/doc
COPY legal/bill_of_material.txt /usr/share/doc
COPY amazon-ssm-agent.json /etc/amazon/ssm/

ENTRYPOINT ["dockerd-entrypoint.sh"]

#=======================End of layer: aarch64_v2  =================


