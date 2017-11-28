# vim:set ft=dockerfile:
FROM debian:jessie

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

# add gosu for easy step-down from root
ENV GOSU_VERSION 1.7
RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

RUN mkdir /docker-entrypoint-initdb.d

RUN apt-get update && apt-get install -y cmake git wget make gcc g++ libncurses5-dev gnutls-dev libbison-dev && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y libevent-dev libaio-dev libcrack2-dev libpam0g-dev  && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y build-essential devscripts dpkg-sig git && rm -rf /var/lib/apt/lists/*

 
RUN apt-get update && apt-get install -y libsnappy-dev  liblzo2-dev libbz2-dev liblzma-dev   liblz4-dev  && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y bison chrpath cmake debhelper dh-apparmor dpatch libaio-dev libboost-dev libjudy-dev libkrb5-dev libncurses5-dev  libpam0g-dev libreadline-gplv2-dev libssl-dev po-debconf psmisc zlib1g-dev  libcrack2-dev libjemalloc-dev && rm -rf /var/lib/apt/lists/*


# install "pwgen" for randomizing passwords
# install "apt-transport-https" for Percona's repo (switched to https-only)
RUN apt-get update && apt-get install -y --no-install-recommends \
		apt-transport-https ca-certificates \
		pwgen \
	&& rm -rf /var/lib/apt/lists/*


# add repository pinning to make sure dependencies from this MariaDB repo are preferred over Debian dependencies
#  libmariadbclient18 : Depends: libmysqlclient18 (= 5.5.42+maria-1~wheezy) but 5.5.43-0+deb7u1 is to be installed

# the "/var/lib/mysql" stuff here is because the mysql-server postinst doesn't have an explicit way to disable the mysql_install_db codepath besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
# also, we set debconf keys to make APT a little quieter
RUN { \
		echo "mariadb-server-$MARIADB_MAJOR" mysql-server/root_password password 'unused'; \
		echo "mariadb-server-$MARIADB_MAJOR" mysql-server/root_password_again password 'unused'; \
	} | debconf-set-selections \
	&& apt-get update \
	&& apt-get install -y \
		socat \
	&& rm -rf /var/lib/apt/lists/* \
# comment out any "user" entires in the MySQL config ("docker-entrypoint.sh" or "--user" will handle user switching)
# purge and re-create /var/lib/mysql with appropriate ownership
	&& rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld /etc/mysql /etc/mysql/conf.d \
	&& chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
	&& chmod 777 /var/run/mysqld \
# comment out a few problematic configuration values
	&& find /etc/mysql/ -name '*.cnf' -print0 \
		| xargs -0 grep -lZE '^(bind-address|log)' \
		| xargs -rt -0 sed -Ei 's/^(bind-address|log)/#&/' \
# don't reverse lookup hostnames, they are usually another container
	&& echo '[mysqld]\nskip-host-cache\nskip-name-resolve' > /etc/mysql/conf.d/docker.cnf

COPY my.cnf /etc/mysql/my.cnf

RUN git clone https://github.com/MariaDB/server.git
RUN cd server && git checkout 10.2 && ./BUILD/compile-pentium64-debug-max && cmake --build  . --config Debug && make -j4 install
RUN ln -s /usr/local/mysql/bin/ /usr/local/mysql/scripts/bin
RUN ln -s /usr/local/mysql/bin/resolveip /usr/bin/resolveip
RUN ln -s /usr/local/mysql/bin/mysqld /usr/bin/mysqld
RUN ln -s /usr/local/mysql/share/english /usr/share/english
RUN ln -s /usr/local/mysql/share/ /usr/share/mysql

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh / # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

ENV PATH="/usr/local/mysql:/usr/local/mysql/bin:/usr/local/mysql/scripts:${PATH}"

EXPOSE 3306
CMD ["mysqld"]
