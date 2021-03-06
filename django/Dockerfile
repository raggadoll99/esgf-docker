#####
## Base image for serving Django apps as a self-contained unit (static files and all).
##
## This uses gunicorn for the WSGI server.
##
## This container is designed so that images built from it can be run by any user
## in the root group.
#####

# Use the smallest base image we can
FROM python:2.7-slim

# Ensure that Python outputs everything that's printed inside
# the application rather than buffering it.
ENV PYTHONUNBUFFERED 1

# Create an unprivileged user to use
ENV DJANGO_USER django
ENV DJANGO_GROUP django
ENV DJANGO_UID 1001
ENV DJANGO_GID 1001
RUN groupadd -g $DJANGO_GID $DJANGO_GROUP && \
    useradd -M -g $DJANGO_GID -s /sbin/nologin -u $DJANGO_UID $DJANGO_USER

# Install curl and gettext-base (for envsubst)
RUN apt-get update && \
    apt-get install -y curl gettext-base && \
    rm -rf /var/lib/apt/lists/*

# Create a directory for files created at runtime that is owned by the django
# user but also read-writable by the root group
ENV DJANGO_STATIC_ROOT /opt/django/static
RUN mkdir -p /opt/django/conf && \
    mkdir -p $DJANGO_STATIC_ROOT && \
    chown -R $DJANGO_UID:0 /opt/django && \
    chmod -R 775 /opt/django

# Install gunicorn and paste
RUN pip install gunicorn Paste PasteDeploy

# Install paste wrapper and env settings into site-packages
COPY python/*.py /usr/local/lib/python2.7/site-packages/

EXPOSE 8000

# Install startup script
COPY scripts/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
