FROM golang:1.13 AS go_builder
WORKDIR /go/src/github.com/percona/percona-xtradb-cluster-operator/src

RUN go get k8s.io/apimachinery/pkg/util/sets \
    && curl -Lf -o /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list.go https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/master/cmd/peer-list/main.go \
    && go build peer-list.go

FROM centos:7 AS centos7

LABEL name="ProxySQL" \
	release="2.0" \
	vendor="Percona" \
	summary="High-performance MySQL proxy with a GPL license" \
	description="ProxySQL is a high performance, high availability, protocol aware proxy for MySQL and forks (like Percona Server and MariaDB). All the while getting the unlimited freedom that comes with a GPL license." \
	maintainer="Percona Development <info@percona.com>"

# check repository package signature in secure way
RUN export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver pool.sks-keyservers.net --recv-keys 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A 6341AB2753D78A78A7C27BB124C6A8A7F4A80EB5 91E97D7C4A5E96F17F3E888F6A2FAEA2352C64E5 \
	&& gpg --export --armor 430BDF5C56E7C94E848EE60C1C4CBDCDCD2EFD2A > ${GNUPGHOME}/RPM-GPG-KEY-Percona \
	&& gpg --export --armor 6341AB2753D78A78A7C27BB124C6A8A7F4A80EB5 > ${GNUPGHOME}/RPM-GPG-KEY-CentOS-7 \
	&& gpg --export --armor 91E97D7C4A5E96F17F3E888F6A2FAEA2352C64E5 > ${GNUPGHOME}/RPM-GPG-KEY-EPEL-7 \
	&& rpmkeys --import ${GNUPGHOME}/RPM-GPG-KEY-Percona ${GNUPGHOME}/RPM-GPG-KEY-CentOS-7 ${GNUPGHOME}/RPM-GPG-KEY-EPEL-7 \
	&& curl -L -o /tmp/percona-release.rpm https://repo.percona.com/yum/percona-release-latest.noarch.rpm \
	&& rpmkeys --checksig /tmp/percona-release.rpm \
	&& rpm -i /tmp/percona-release.rpm \
	&& rm -rf "$GNUPGHOME" /tmp/percona-release.rpm \
	&& rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY

# install exact version for repeatability
ENV PS_VERSION 5.7.29-32.1.el7
ENV PROXYSQL_VERSION 2.0.10-1.1.el7

RUN curl -L -o /tmp/jq.rpm https://download.fedoraproject.org/pub/epel/7/x86_64/Packages/j/jq-1.6-1.el7.x86_64.rpm \
	&& curl -L -o /tmp/oniguruma.rpm https://download.fedoraproject.org/pub/epel/7/x86_64/Packages/o/oniguruma-5.9.5-3.el7.x86_64.rpm \
	&& rpmkeys --checksig /tmp/jq.rpm /tmp/oniguruma.rpm \
	&& rpm -i /tmp/jq.rpm /tmp/oniguruma.rpm \
	&& rm -rf /tmp/jq.rpm /tmp/oniguruma.rpm

RUN yum update -y \
        && yum install -y \
                Percona-Server-client-57-${PS_VERSION} \
                Percona-Server-shared-57-${PS_VERSION} \
                shadow-utils \
                which \
                hostname \
                perl-DBD-MySQL \
                perl-DBI \
                gnutls \
        && yum clean all

RUN groupadd -g 1001 proxysql \
    && useradd -u 1001 -r -g 1001 -s /sbin/nologin \
        -c "Default Application User" proxysql

# we need licenses from docs
RUN sed -i '/nodocs/d' /etc/yum.conf || : \
        && repoquery -a --location \
               proxysql2-${PROXYSQL_VERSION} \
                   | xargs curl -L -o /tmp/proxysql2-${PROXYSQL_VERSION}.rpm \
        && rpm -iv /tmp/proxysql2-${PROXYSQL_VERSION}.rpm --nodeps \
        && rm -rf /tmp/proxysql2-${PROXYSQL_VERSION}.rpm \
        && yum clean all \
        && rm -rf /var/cache/yum /etc/proxysql /var/lib/proxysql \
        && rpm -ql Percona-Server-client-57 | egrep -v "mysql$|mysqldump$" | xargs rm -rf \
        && install -o 1001 -g 0 -m 775 -d /etc/proxysql /var/lib/proxysql

COPY LICENSE /licenses/LICENSE.Dockerfile
RUN cp /usr/share/doc/proxysql2-$(echo ${PROXYSQL_VERSION} | cut -d - -f 1)/LICENSE /licenses/LICENSE.proxysql

COPY dockerdir /
COPY --from=go_builder /go/src/github.com/percona/percona-xtradb-cluster-operator/src/peer-list /usr/bin/
RUN chown 1001:1001 /etc/proxysql/proxysql.cnf /etc/proxysql-admin.cnf \
    && chmod 664 /etc/proxysql/proxysql.cnf /etc/proxysql-admin.cnf

USER 1001

VOLUME /var/lib/proxysql

ENTRYPOINT ["/entrypoint.sh"]

EXPOSE 3306 6032
CMD ["/usr/bin/proxysql", "-f", "-c", "/etc/proxysql/proxysql.cnf"]
