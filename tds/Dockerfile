#####
## Docker image for running the THREDDS Data Server (TDS) with Tomcat
#####

## In order to install the jars required, we need to unpack several war files.
##
## Unpacking a war requires the jar command, which is part of the JDK. However,
## we only want the JRE in the final container.
##
## To get around this, we use a separate build stage with the full JDK to
## download and unpack the wars, and copy the unpacked folder into the final container.

ARG ESGF_HUB=esgfhub
ARG ESGF_PREFIX=
ARG ESGF_VERSION=latest

# This build stage is required because COPY --from=$ARG is not supported
# https://github.com/moby/moby/issues/34482
FROM ${ESGF_HUB}/${ESGF_PREFIX}configure:${ESGF_VERSION} as configuration

FROM openjdk:8-jdk AS esgf-tds-unpack

ARG ESGF_REPO=http://dist.ceda.ac.uk/esgf

# First, download and unpack THREDDS
RUN mkdir /opt/thredds && \
    cd /opt/thredds && \
    TMPFILE=$(mktemp) && \
    curl -o $TMPFILE -fsSL $ESGF_REPO/dist/thredds/5.0/5.0.2/thredds.war && \
    jar xvf $TMPFILE && \
    cd -

# Get jars required for TDS filters from the ESGF repo and install them
RUN cd /opt/thredds/WEB-INF/lib && \
    curl -O -fsSL $ESGF_REPO/artifacts/repositories/esgf-repo-share/xml-security/xmlsec/1.4.2/xmlsec-1.4.2.jar && \
    curl -O -fsSL $ESGF_REPO/artifacts/repositories/esgf-repo-share/velocity/velocity/1.5/velocity-1.5.jar && \
    curl -Of -sSL $ESGF_REPO/artifacts/repositories/esgf-repo-share/xalan/serializer/2.9.1/serializer-2.9.1.jar && \
    curl -O -fsSL $ESGF_REPO/artifacts/repositories/esgf-repo-share/postgresql/postgresql/8.4-703.jdbc3/postgresql-8.4-703.jdbc3.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/XSGroupRole-1.0.0.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/commons-httpclient-3.1.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/commons-lang-2.6.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/jdom-legacy-1.1.3.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/opensaml-2.3.2.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/openws-1.3.1.jar && \
    curl -O -fsSL $ESGF_REPO/dist/filters/xmltooling-1.2.2.jar && \
    curl -O -fsSL $ESGF_REPO/dist/esg-orp/esg-orp-2.9.6.jar && \
    curl -O -fsSL $ESGF_REPO/dist/esgf-node-manager/esgf-node-manager-common-1.0.0.jar && \
    curl -O -fsSL $ESGF_REPO/dist/esgf-node-manager/esgf-node-manager-filters-1.0.0.jar && \
    curl -O -fsSL $ESGF_REPO/dist/esgf-security/esgf-security-2.7.10.jar && \
    cd -

# FIXME: Obtain missing jars from ORP and old node-manager distributions
RUN TMPFILE=$(mktemp) && \
    curl -o $TMPFILE -fsSL $ESGF_REPO/dist/esg-orp/esg-orp.war && \
    TMPDIR=$(mktemp -d) && \
    cd $TMPDIR && \
    jar xvf $TMPFILE && \
    rm -f $TMPFILE && \
    cp WEB-INF/lib/xalan-2.7.2.jar /opt/thredds/WEB-INF/lib/ && \
    cp WEB-INF/lib/xercesImpl-2.10.0.jar /opt/thredds/WEB-INF/lib/ && \
    cp WEB-INF/lib/xml-apis-1.4.01.jar /opt/thredds/WEB-INF/lib/ && \
    cp WEB-INF/lib/commons-io-2.4.jar /opt/thredds/WEB-INF/lib/ && \
    cd - && \
    rm -rf $TMPDIR
RUN TMPFILE=$(mktemp) && \
    curl -o $TMPFILE -fsSL $ESGF_REPO/dist/esgf-node-manager/esgf-node-manager.war && \
    TMPDIR=$(mktemp -d) && \
    cd $TMPDIR && \
    jar xvf $TMPFILE && \
    rm -f $TMPFILE && \
    cp WEB-INF/lib/commons-dbcp-1.4.jar /opt/thredds/WEB-INF/lib/ && \
    cp WEB-INF/lib/commons-dbutils-1.3.jar /opt/thredds/WEB-INF/lib/ && \
    cp WEB-INF/lib/commons-pool-1.5.4.jar /opt/thredds/WEB-INF/lib/ && \
    cd - && \
    rm -rf $TMPDIR

# This is important to ensure the created files are writable by the root group later
RUN chmod -R g+w /opt/thredds


#############
## Stage 2 ##
#############

FROM ${ESGF_HUB}/${ESGF_PREFIX}tomcat:$ESGF_VERSION

MAINTAINER Earth System Grid Federation <esgf-devel@lists.llnl.gov>

ENV ESGF_HOME /esg

# We need rsync to do an intelligent copy of the THREDDS content skeleton (see below)
RUN apt-get update && apt-get install -y rsync && rm -rf /var/lib/apt/lists/*

# Copy the unpacked webapp from the previous build stage
COPY --from=esgf-tds-unpack --chown=1001:0 /opt/thredds $CATALINA_HOME/webapps/thredds

# Copy configuration and utils from esgf-configure
# Ensure the config directory is owned by the tomcat user and root group
RUN mkdir -p /esg/config && chown 1001:0 /esg/config && chmod 775 /esg/config
COPY --chown=1001:0 --from=configuration /esg/config /esg/config
# Make sure this script runs first by giving it a low ordinal
COPY --from=configuration /opt/esgf-docker/scripts/interpolate-configs.sh /tomcat-init.d/01-interpolate.sh

# Install environment customisations, config files and templates
COPY scripts/setenv.sh $CATALINA_HOME/bin/
# Make sure this script runs after the settings are interpolated
COPY scripts/setup-thredds.sh /tomcat-init.d/02-setup-thredds.sh
# TDS customized applicationContext.xml file with ESGF authorizer
COPY conf/applicationContext.xml $CATALINA_HOME/webapps/thredds/WEB-INF/
# TDS customized logging (uses DEBUG)
COPY conf/log4j2.xml $CATALINA_HOME/webapps/thredds/WEB-INF/classes/
# Template for web.xml - this is interpolated with environment vars at runtime
COPY conf/web.xml.template $CATALINA_HOME/webapps/thredds/WEB-INF/
# Template for tomcat-users.xml - interpolated with environment vars at runtime
COPY conf/tomcat-users.xml.template $CATALINA_HOME/conf/

# Create the thredds content root owned by tomcat user and root group
RUN mkdir -p /esg/content/thredds/esgcet && \
    chown -R 1001:0 /esg/content/thredds && \
    chmod -R g+w /esg/content/thredds
# The threddsConfig.xml and root catalog.xml will normally not come from a volume
# as they don't change with publishing
# They should come through the same route as regular config files
COPY conf/thredds/*.xml /esg/content/thredds/

# Run as the tomcat user by default
USER 1001
