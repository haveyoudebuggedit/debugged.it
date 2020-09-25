---
slug: immutable-infra-2
guid: https://pasztor.at/blog/immutable-infra-2/
authors:
    - janos
categories:
- Technology
date: "2019-05-17T00:00:00Z"
publishDate: "2019-05-17T00:00:00Z"
summary: After setting up the server as a Docker baseline, we now venture in building a Docker infrastructure.
images:
- posts/immutable-infra-2.png
preview: posts/immutable-infra-2.jpg
tags:
- DevOps
- Containers
- Cloud
title: Immutable Infrastructure in Practice — Part 2
---

In the [previous part](/blog/immutable-infra-1) we discussed how to set up a server based on the principle of immutability. Now that we have a server and a way to deploy the Docker package, let's discuss how we set up our Docker containers.

## To Registry or not to Registry

The simplest way to deploy the Docker package would be to use a Docker registry. However, as I'm using an IaaS platform, I don't have a registry readily available. I could, of course, pay for an external registry, but to keep things simple and reproducible, I opted for a different solution.

In my case I created a [docker-compose](https://docs.docker.com/compose/) file for the whole setup. All the files required to build the Docker setup are included in a single directory and are deployed to the server as described in the previous article. Then `docker-compose build` and `docker-compose up` is ran.

## Traefik

Our first container is going to be [Traefik](https://traefik.io/). Traefik is an amazing reverse proxy that supports Docker for routing requests, and also supports [LetsEncrypt](https://letsencrypt.org/) as a way to generate certificates.

The `docker-compose.yaml` part looks quite simple:

```yaml
---
version: '3.2'
services:
  traefik:
    build:
      context: images/traefik
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /srv/acme:/etc/traefik/acme
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - "BACKUP_DEST=s3://${ACME_BUCKET_NAME}"
      - "BACKUP_ENDPOINT=https://sos-${EXOSCALE_REGION}.exo.io"
      - "BACKUP_BUCKETURL=%(bucket)s.sos-${EXOSCALE_REGION}.exo.io"
      - "EXOSCALE_KEY=${EXOSCALE_KEY}"
      - "EXOSCALE_SECRET=${EXOSCALE_SECRET}"
    restart: always
```

As mentioned in the previous article, the idea here is that the ACME (certificate) file will be backed up every minute,
and when the server starts the certificate file will be restored. This is important because LetsEncrypt has
[rate limits](https://letsencrypt.org/docs/rate-limits/) and when working on the infrastructure I would run into this
rate limit.

With that in mind the Traefik container is built based on Alpine Linux and a shell script is started instead of
Traefik directly. this script contains the restore procedure:

```bash
if [ ! -f /etc/traefik/acme/acme.json ]; then
    echo "[default]
host_base = sos-${EXOSCALE_REGION}.exo.io
host_bucket = ${BACKUP_BUCKETURL}
access_key = ${EXOSCALE_KEY}
secret_key = ${EXOSCALE_SECRET}
use_https = True
" > ~/.s3cfg
    s3cmd sync --host=${BACKUP_ENDPOINT} ${BACKUP_DEST} /etc/traefik/acme
    rm ~/.s3cfg
fi
export EXOSCALE_KEY=
export EXOSCALE_SECRET=
```

Not terribly complicated, it simply copies the `acme.json` from the object storage to the local filesystem. If the
file does not exist on the bucket Traefik will generate a new one and the backup, running in a separate container, will
store them on the object storage.

## nginx

Now on to nginx. The container itself is
[pretty simple](https://github.com/janoszen/pasztor.at/blob/master/_terraform/docker/images/nginx/Dockerfile). In the
`docker-compose` file we add some labels to the container, which Traefik can read and route the appropriate requests
to the nginx container:

```yaml
version: '3.2'
services:
  #...
  nginx:
    build:
      context: images/nginx
    volumes:
      - /srv/www:/var/www
    labels:
      traefik.enable: "true"
      traefik.backend: "nginx"
      traefik.frontend.rule: "Host:${DOMAIN}"
      traefik.port: "80"
      traefik.protocol: "http"
      traefik.frontend.headers.SSLTemporaryRedirect: "false"
      traefik.frontend.headers.SSLRedirect: "true"
      traefik.frontend.headers.STSSeconds: "315360000"
      traefik.frontend.headers.STSIncludeSubdomains: "true"
      traefik.frontend.headers.STSPreload: "true"
      traefik.frontend.headers.forceSTSHeader: "false"
    restart: always
```

## Pulling content

As I mentioned before my website is built in [Jekyll](https://jekyllrb.com/), which generates static HTML files. In
order to deploy the content of my website I use [CircleCI](https://circleci.com/gh/janoszen/pasztor.at), which
runs Jekyll and then
[copies the content to the object storage](https://github.com/janoszen/pasztor.at/blob/master/.circleci/deploy.sh).

So I need a way to pull this content to the server. Normally one would use a cronjob to achieve this task, but 
crond does not like to run in containers, especially not without root privileges. Thankfully there's
[superchronic](https://github.com/aptible/supercronic), a rootless cron daemon. As an added bonus Superchronic
has a nifty feature that it waits for a cronjob to finish before it starts the next one, so the jobs won't run in
parallel.

After
[containerizing Superchronic](https://github.com/janoszen/pasztor.at/blob/master/_terraform/docker/images/pull/Dockerfile),
we can add the `crontab` file as follows:

```bash
#!/bin/bash

set -e

echo "[default]
host_base = sos-${EXOSCALE_REGION}.exo.io
host_bucket = ${PULL_BUCKETURL}
access_key = ${EXOSCALE_KEY}
secret_key = ${EXOSCALE_SECRET}
use_https = True
" > ~/.s3cfg

s3cmd sync --delete-removed --host=${PULL_ENDPOINT} ${PULL_SOURCE} /var/www/htdocs

rm ~/.s3cfg

echo "last_pull $(date +%s)" > /srv/monitoring/last_pull.prom
```

Most of it is pretty simple. One interesting tidbit in the last line: I'm saving the timestamp of the last pull in a
file. This file will later be read by the [Prometheus node exporter](https://github.com/prometheus/node_exporter) and
added to monitoring. If for whatever reason the pull fails the monitoring system will alert.

## Backups

We also need to back up two things: the `acme.json` file from Traefik, and the metrics from Prometheus. The method is 
the same as with the content pull, put it in Superchronic and write the timestamp into a metric file.

## Prometheus

[Prometheus](https://prometheus.io/) is my choice of monitoring. It's concept is ingenious: it does one thing and 
it does it well. Every 15 seconds (or whatever you set it to), it scans a number of HTTP endpoints and downloads the
metrics data from these. So in order to do an effective monitoring I need a little bit of extra tooling. In total I 
need to start a Prometheus container, as well as three metrics collectors called *exporters*.

The exporters I use are the following:

[node-exporter](https://github.com/prometheus/node_exporter)
: to gather the server metrics.

[cadvisor](https://github.com/google/cadvisor)
: for gathering the container metrics.

[nginx-exporter](https://github.com/nginxinc/nginx-prometheus-exporter)
: to gather nginx metrics.

In addition to that I also use the built-in exporter in Traefik to gather Traefik metrics.

My `prometheus.yml` file looks as follows:

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
  - job_name: 'node'
    static_configs:
      - targets: ['172.28.0.1:9100']
  - job_name: 'docker'
    static_configs:
      - targets: ['cadvisor:8080']
  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8080']
```

{{% tip %}}
*Tip:* The node exporter needs to run in network mode `host`, so it won't be able to participate in the docker-compose network. However, I assign IPs manually, so I can use the host servers IP address in the container network to access its data.
{{% /tip %}}

## Grafana

The last piece of the puzzle is [Grafana](https://grafana.com/). Grafana is my tool of choice for displaying a dashboard
and also to send alerts to VictorOps if one of the metrics is out of bounds. My Grafana is
[public](https://monitoring.pasztor.at), feel free to take a look.

Since I'm doing immutable infrastructure I have a problem though. If I create my dashboards and metrics by clicking
around in the dashboard, I would have to back up my Grafana database and restore it, similar to how I back up 
Prometheus.

However, I opted for a different solution. Grafana has a feature called
[provisioning](https://grafana.com/docs/administration/provisioning/), where the dashboards can be provisioned from
files. In other words, I provide the
[dashboard JSON as a file](https://github.com/janoszen/pasztor.at/blob/master/_terraform/docker/images/grafana/root/etc/grafana/dashboards/default.json)
which Grafana loads on start. When I want to change the settings I click the changes in the dashboard, and then save the
JSON in my git repository for reprovisioning with Terraform.

Like before, the Grafana container gets some labels to Traefik can route the requests appropriately:

```yaml
version: '3.2'
services:
  # ...
  grafana:
    build:
      context: images/grafana
    environment:
      - "GF_SERVER_DOMAIN=monitoring.${DOMAIN}"
      - "GF_SERVER_ROOT_URL=https://monitoring.${DOMAIN}"
      - "GF_SERVER_ENFORCE_DOMAIN=true"
      - "GF_USERS_AUTO_ASSIGN_ORG_ROLE=Admin"
      - "GF_USERS_AUTO_ASSIGN_ORG=true"
      - "GF_AUTH_DISABLE_LOGIN_FORM=true"
      - "GF_AUTH_ANONYMOUS_ENABLED=true"
      - "GF_AUTH_ANONYMOUS_ORG_NAME=Main Org."
      - "GF_AUTH_GITHUB_ENABLED=true"
      - "GF_AUTH_GITHUB_SCOPES=user:email,read:org"
      - "GF_AUTH_GITHUB_AUTH_URL=https://github.com/login/oauth/authorize"
      - "GF_AUTH_GITHUB_TOKEN_URL=https://github.com/login/oauth/access_token"
      - "GF_AUTH_GITHUB_API_URL=https://api.github.com/user"
      - "GF_AUTH_GITHUB_ALLOW_SIGNUP=true"
      - "GF_AUTH_GITHUB_CLIENT_ID=${GITHUB_CLIENT_ID}"
      - "GF_AUTH_GITHUB_CLIENT_SECRET=${GITHUB_CLIENT_SECRET}"
      - "GF_AUTH_GITHUB_ALLOWED_ORGANIZATIONS=opsbears"
      - "GF_SECURITY_SECRET_KEY=${GRAFANA_SECRET_KEY}"
      - "GF_SECURITY_COOKIE_SECURE=true"
      - "GF_SECURITY_COOKIE_SAMESITE=true"
      - "GF_ANALYTICS_REPORTING_ENABLED=false"
      - "BACKUP_DEST=s3://${GRAFANA_BUCKET_NAME}"
      - "BACKUP_ENDPOINT=https://sos-${EXOSCALE_REGION}.exo.io"
      - "BACKUP_BUCKETURL=%(bucket)s.sos-${EXOSCALE_REGION}.exo.io"
      - "EXOSCALE_KEY=${EXOSCALE_KEY}"
      - "EXOSCALE_SECRET=${EXOSCALE_SECRET}"
    volumes:
      - "/srv/grafana:/var/lib/grafana"
    labels:
      traefik.enable: "true"
      traefik.backend: "grafana"
      traefik.frontend.rule: "Host:monitoring.${DOMAIN}"
      traefik.port: "3000"
      traefik.protocol: "http"
      traefik.frontend.headers.SSLTemporaryRedirect: "false"
      traefik.frontend.headers.SSLRedirect: "true"
      traefik.frontend.headers.STSSeconds: "315360000"
      traefik.frontend.headers.STSIncludeSubdomains: "true"
      traefik.frontend.headers.STSPreload: "true"
      traefik.frontend.headers.forceSTSHeader: "false"
      traefik.frontend.redirect.regex: "^https://monitoring.${DOMAIN}/$$"
      traefik.frontend.redirect.replacement: "https://monitoring.${DOMAIN}/d/mXFB_yRWz/home?orgId=1"
      traefik.frontend.redirect.permanent: "true"
    restart: always
    networks:
      internal:
        ipv4_address: 172.28.1.11
```

## Summary

This setup took me a couple of weeks to build. You can definitely go simpler, but as my CDN experiment has showed me, 
not having monitoring or a well built, tested setup will cost time in the long run. Now I can be confident that my setup
is future-proof.
