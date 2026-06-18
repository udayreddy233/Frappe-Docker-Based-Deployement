#for postgres deployments
FROM public.ecr.aws/asakta/frappe-base:postgres AS base

FROM base AS builder

USER root

RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install --no-install-recommends -y \
       wget \
       libcairo2-dev \
       libpango1.0-dev \
       libmariadb-dev \
       libjpeg-dev \
       libgif-dev \
       librsvg2-dev \
       libpq-dev \
       libffi-dev \
       liblcms2-dev \
       libldap2-dev \
       libsasl2-dev \
       libtiff5-dev \
       libwebp-dev \
       redis-tools \
       rlwrap \
       tk8.6-dev \
       cron \
       gcc \
       build-essential \
       libbz2-dev \
       && rm -rf /var/lib/apt/lists/*

ARG APPS
RUN mkdir /opt/frappe && touch /opt/frappe/apps.json \
    && echo $APPS | base64 -d > /opt/frappe/apps.json

USER frappe

ARG FRAPPE_BRANCH
ARG FRAPPE_PATH

RUN bench init --apps_path=/opt/frappe/apps.json \
      --frappe-branch=${FRAPPE_BRANCH} \
      --frappe-path=${FRAPPE_PATH} \
      --no-procfile \
      --no-backups \
      --skip-redis-config-generation \
      /home/frappe/frappe-bench \
    && cd /home/frappe/frappe-bench \
    && bench pip install captcha==0.7.1 \
    && bench pip install python-dotenv \
    && echo "captcha_enabled=0" > apps/frappe/.env \
    && echo "captcha_secret_key=6LdoxOAqAAAAAGDUQcWSucEP7v8ibE6M17TvPzqx" >> apps/frappe/.env \
    && echo "captcha_site_key=6LdoxOAqAAAAAEyrsvTSSPtoWutlInojb37CmDhc" >> apps/frappe/.env

RUN cd /home/frappe/frappe-bench \
    && find . -type d -name "__pycache__" -prune -exec rm -rf {} + \
    && find . -name "*.pyc" -delete \
    && ./env/bin/pip cache purge || true


FROM base AS backend

USER root

COPY nginx-template.conf /templates/nginx/frappe.conf.template
COPY nginx-entrypoint.sh /usr/local/bin/nginx-entrypoint.sh

RUN sed -i 's/\r$//' /usr/local/bin/nginx-entrypoint.sh \
    && sed -i 's/\r$//' /templates/nginx/frappe.conf.template \
    && sed -i '/user www-data/d' /etc/nginx/nginx.conf \
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log \
    && touch /run/nginx.pid \
    && chown -R frappe:frappe /etc/nginx/conf.d /etc/nginx/nginx.conf /var/log/nginx /var/lib/nginx /run/nginx.pid \
    && chmod 755 /usr/local/bin/nginx-entrypoint.sh \
    && chmod 644 /templates/nginx/frappe.conf.template

USER frappe

COPY --from=builder --chown=frappe:frappe /home/frappe/frappe-bench /home/frappe/frappe-bench

WORKDIR /home/frappe/frappe-bench

VOLUME [ \
  "/home/frappe/frappe-bench/sites", \
  "/home/frappe/frappe-bench/apps", \
  "/home/frappe/frappe-bench/env", \
  "/home/frappe/frappe-bench/config", \
  "/home/frappe/frappe-bench/sites/assets", \
  "/home/frappe/frappe-bench/logs" \
]

CMD [ \
  "env", \
  "PYTHONPATH=/home/frappe/frappe-bench/apps", \
  "/home/frappe/frappe-bench/env/bin/gunicorn", \
  "--chdir=/home/frappe/frappe-bench/sites", \
  "--bind=0.0.0.0:8000", \
  "--threads=4", \
  "--workers=5", \
  "--worker-class=gthread", \
  "--worker-tmp-dir=/dev/shm", \
  "--timeout=120", \
  "--preload", \
  "frappe.app:application" \
]