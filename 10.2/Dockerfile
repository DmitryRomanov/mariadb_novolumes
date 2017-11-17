# vim:set ft=dockerfile:
FROM centos:7

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

# add gosu for easy step-down from root
RUN gpg --keyserver pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
    && curl -o /usr/local/bin/gosu -SL "https://github.com/tianon/gosu/releases/download/1.2/gosu-amd64" \
    && curl -o /usr/local/bin/gosu.asc -SL "https://github.com/tianon/gosu/releases/download/1.2/gosu-amd64.asc" \
    && gpg --verify /usr/local/bin/gosu.asc \
    && rm /usr/local/bin/gosu.asc \
    && rm -r /root/.gnupg/ \
    && chmod +x /usr/local/bin/gosu

RUN mkdir /docker-entrypoint-initdb.d

# install "pwgen" for randomizing passwords
# install "apt-transport-https" for Percona's repo (switched to https-only)
RUN \
	{ \
		echo "# MariaDB 10.2 CentOS repository list - created 2016-10-28 06:40 UTC"; \
		echo "# http://downloads.mariadb.org/mariadb/repositories/"; \
		echo "[mariadb]"; \
		echo "name = MariaDB"; \
		echo "baseurl = http://yum.mariadb.org/10.2/centos7-amd64"; \
		echo "gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB"; \
		echo "gpgcheck=1"; \
	} > /etc/yum.repos.d/mariadb-10.1.repo && \
	yum clean all

RUN yum -y update && yum -y install \
	MariaDB-server
# add repository pinning to make sure dependencies from this MariaDB repo are preferred over Debian dependencies
#  libmariadbclient18 : Depends: libmysqlclient18 (= 5.5.42+maria-1~wheezy) but 5.5.43-0+deb7u1 is to be installed

# the "/var/lib/mysql" stuff here is because the mysql-server postinst doesn't have an explicit way to disable the mysql_install_db codepath besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
# also, we set debconf keys to make APT a little quieter
RUN sed -ri 's/^user\s/#&/' /etc/my.cnf.d/server.cnf \
	&& rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld \
	&& chown -R mysql:mysql /var/lib/mysql /var/run/mysqld

COPY my.cnf /etc/my.cnf.d/server.cnf

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh / # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 3306
CMD ["mysqld"]
