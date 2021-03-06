ARG BASEOS
ARG BASEVER
ARG PG_FULL
ARG PREFIX
FROM ${PREFIX}/crunchy-pg-base:${BASEOS}-${PG_FULL}-${BASEVER}

# ===== Early lines ordered for leveraging cache, reorder carefully =====
# For RHEL8 all arguments used in main code has to be specified after FROM
ARG BASEOS
ARG PG_FULL
ARG DFSET
ARG PACKAGER
ARG PG_MAJOR

# Preserving PGVERSION out of paranoia
ENV PGROOT="/usr/pgsql-${PG_MAJOR}" PGVERSION="${PG_MAJOR}"

ARG BACKREST_VER

# Separate yum run matching postgres image, leverages cached layer
RUN if [ "$DFSET" = "centos" ] ; then \
       ${PACKAGER} -y install \
        	--setopt=skip_missing_names_on_install=False \
        	openssh-clients \
        	openssh-server \
        	pgaudit${PG_MAJOR//.} \
        	crunchy-backrest-${BACKREST_VER} \
        	postgresql${PG_MAJOR//.}-contrib \
        	postgresql${PG_MAJOR//.}-server \
        	postgresql${PG_MAJOR//.}-plpython* \
        	pgnodemx${PG_MAJOR//.} \
        	$( printf '11\n'${PG_MAJOR} | sort -VC && echo postgresql${PG_MAJOR}-llvmjit ) \
        	psmisc \
        	rsync \
        	less \
        	wal2json${PG_MAJOR//.} \
        && ${PACKAGER} -y clean all ; \
else \
       ${PACKAGER} -y install \
		--enablerepo="epel" \
		--setopt=skip_missing_names_on_install=False \
		openssh-clients \
		openssh-server \
		pgaudit${PG_MAJOR//.} \
		pgaudit${PG_MAJOR//.}_set_user \
		crunchy-backrest-${BACKREST_VER} \
		postgresql${PG_MAJOR//.}-contrib \
		postgresql${PG_MAJOR//.}-server \
		postgresql${PG_MAJOR//.}-plpython* \
		pgnodemx${PG_MAJOR//.} \
		$( printf '11\n'${PG_MAJOR} | sort -VC && echo postgresql${PG_MAJOR}-llvmjit ) \
		psmisc \
		rsync \
		less \
		wal2json${PG_MAJOR//.} \
	&& ${PACKAGER} -y install \
		--setopt=tsflags='' \
		--enablerepo="epel" \
		pgaudit_analyze \
	&& ${PACKAGER} -y clean all --enablerepo="epel" ; \
fi

# ===== Steps unique to this image after here =====
ARG PATRONI_VER

LABEL name="postgres-ha" \
	summary="PostgreSQL ${PG_FULL} with Patroni" \
	description="Used for the deployment and management of highly-available PostgreSQL clusters using Patroni." \
	io.k8s.description="Crunchy PostgreSQL optimized for high-availability (HA)" \
	io.k8s.display-name="Crunchy PostgreSQL - HA Optimized" \
	io.openshift.tags="postgresql,postgres,postgis,sql,nosql,database,ha,crunchy"

RUN if [ "$DFSET" = "centos" ] ; then \
	${PACKAGER} -y install \
		--setopt=skip_missing_names_on_install=False \
		python3-pip \
		python3-psutil \
		python3-psycopg2 \
	&& ${PACKAGER} -y clean all ; \
fi

RUN if [ "$BASEOS" = "rhel7" ] ; then \
	${PACKAGER} -y install \
		--enablerepo="epel,rhel-7-server-optional-rpms" \
		--setopt=skip_missing_names_on_install=False \
		python3-pip \
		python3-psutil \
		python3-psycopg2 \
	&& ${PACKAGER} -y clean all --enablerepo="epel,rhel-7-server-optional-rpms" ; \
fi

RUN if [ "$BASEOS" = "ubi7" ] ; then \
	${PACKAGER} -y install \
		--enablerepo="epel,rhel-7-server-optional-rpms" \
		--setopt=skip_missing_names_on_install=False \
		python3-pip \
		python3-psutil \
		python3-psycopg2 \
	&& ${PACKAGER} -y clean all --enablerepo="epel,rhel-7-server-optional-rpms" ; \
fi

RUN if [ "$BASEOS" = "ubi8" ] ; then \
	${PACKAGER} -y install \
		--enablerepo="epel" \
		--setopt=skip_missing_names_on_install=False \
		python3-pip \
		python3-psutil \
		python3-psycopg2 \
	&& ${PACKAGER} -y clean all --enablerepo="epel" ; \
fi

# install patroni for Kube
RUN pip3 install --upgrade python-dateutil \
	&& pip3 install patroni[kubernetes]=="${PATRONI_VER}"

RUN useradd crunchyadm -g 0 -u 17

ENV PATH="${PGROOT}/bin:${PATH}"

RUN mkdir -p /opt/cpm/bin /opt/cpm/conf /pgdata /pgwal /pgconf /backrestrepo \
		/crunchyadm /tablespaces

# Adjust ownership for the folders to be the "postgres" user and allow the group
# permissions to match the user ones EXCEPT for the /tablespaces folder, which
# will only have permissions on the user
RUN chown -R postgres:postgres /opt/cpm /var/lib/pgsql \
	/pgdata /pgwal /pgconf /backrestrepo /crunchyadm /tablespaces &&  \
	chmod -R g=u /opt/cpm /var/lib/pgsql \
	/pgdata /pgwal /pgconf /backrestrepo /crunchyadm /tablespaces

# open up the postgres port
EXPOSE 5432

ADD bin/postgres-ha /opt/cpm/bin
ADD bin/common /opt/cpm/bin/common
ADD conf/postgres-ha /opt/cpm/conf
ADD tools/pgmonitor/exporter/postgres /opt/cpm/bin/modules/pgexporter

ADD yq /opt/cpm/bin
RUN chmod +x /opt/cpm/bin/yq

RUN chmod g=u /etc/passwd \
 && chmod g=u /etc/group

RUN mkdir /.ssh && chown 26:0 /.ssh && chmod g+rwx /.ssh && rm -f /run/nologin

# The VOLUME directive must appear after all RUN directives to ensure the proper
# volume permissions are applied when building the image
VOLUME ["/pgdata", "/pgwal", "/pgconf", "/backrestrepo", "/sshd"]

ENTRYPOINT ["/opt/cpm/bin/bootstrap-postgres-ha.sh"]

USER 26

CMD ["/usr/local/bin/patroni"]
