ARG PYTHON_VERSION=3.10-slim-bullseye

{% if cookiecutter.frontend_pipeline == 'Gulp' -%}
FROM node:16-bullseye-slim as client-builder

ARG APP_HOME=/app
WORKDIR ${APP_HOME}

COPY ./package.json ${APP_HOME}
RUN npm install && npm cache clean --force
COPY . ${APP_HOME}
RUN npm run build

{%- endif %}

# define an alias for the specfic python version used in this file.
FROM python:${PYTHON_VERSION} as python

# Python build stage
FROM python as python-build-stage

ARG BUILD_ENVIRONMENT=production

# Install apt packages
RUN apt-get update && apt-get install --no-install-recommends -y \
  # dependencies for building Python packages
  build-essential \
  # psycopg2 dependencies
  libpq-dev

# Requirements are installed here to ensure they will be cached.
COPY ./requirements .

# Create Python Dependency and Sub-Dependency Wheels.
RUN pip wheel --wheel-dir /usr/src/app/wheels  \
  -r ${BUILD_ENVIRONMENT}.txt


# Python 'run' stage
FROM python as python-run-stage

ARG BUILD_ENVIRONMENT=production
ARG APP_HOME=/app

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1
ENV BUILD_ENV ${BUILD_ENVIRONMENT}

WORKDIR ${APP_HOME}

RUN addgroup --system django \
    && adduser --system --ingroup django django


# Install required system dependencies
RUN apt-get update && apt-get install --no-install-recommends -y \
  # psycopg2 dependencies
  libpq-dev \
  # Translations dependencies
  gettext \
  build-essential \
  wget \
  automake libtool pkg-config libsqlite3-dev sqlite3 \
  libcurl4-gnutls-dev \
  libproj-dev \
  libxml2-dev \
  libgeos-dev \
  libnetcdf-dev \
  libpoppler-dev \
  libspatialite-dev \
  libhdf4-alt-dev \
  libhdf5-serial-dev \
  libopenjp2-7-dev \
  \
  # Install numpy
    && pip install numpy \
  # Build against PROJ master (which will be released as PROJ 6.0)
    && wget "http://download.osgeo.org/proj/proj-6.0.0.tar.gz" \
    && tar -xzf "proj-6.0.0.tar.gz" \
    && mv proj-6.0.0 proj \
    && echo "#!/bin/sh" > proj/autogen.sh \
    && chmod +x proj/autogen.sh \
    && cd proj \
    && ./autogen.sh \
    && CXXFLAGS='-DPROJ_RENAME_SYMBOLS' CFLAGS='-DPROJ_RENAME_SYMBOLS' ./configure --disable-static --prefix=/usr/local \
    && make -j"$(nproc)" \
    && make -j"$(nproc)" install \
    # Rename the library to libinternalproj
    && mv /usr/local/lib/libproj.so.15.0.0 /usr/local/lib/libinternalproj.so.15.0.0 \
    && rm /usr/local/lib/libproj.so* \
    && rm /usr/local/lib/libproj.la \
    && ln -s libinternalproj.so.15.0.0 /usr/local/lib/libinternalproj.so.15 \
    && ln -s libinternalproj.so.15.0.0 /usr/local/lib/libinternalproj.so \
    \
  # Get GDAL source
    && mkdir -p "/usr/local/src/python-gdal" \
    && cd "/usr/local/src/python-gdal" \
    && wget "http://download.osgeo.org/gdal/3.3.3/gdal-3.3.3.tar.gz" \
    && tar -xvf "gdal-3.3.3.tar.gz" \
    \
  # Compile and install GDAL
    && cd "gdal-3.3.3" \
    && export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH \
    && ./configure \
            --with-python \
            --with-curl \
            --with-openjpeg \
            --without-libtool \
            --with-proj=/usr/local \
    && make -j"$(nproc)" \
    && make install \
    && ldconfig \
  # cleaning up unused files
  && cd /usr/local \
  # Clean up
  && apt-get update -y \
  && apt-get remove -y --purge build-essential wget \
  && apt-get autoremove -y \
  && rm -rf /var/lib/apt/lists/* \
  && rm -rf "/usr/local/src/python-gdal" \
  # cleaning up unused files
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && rm -rf /var/lib/apt/lists/*

# All absolute dir copies ignore workdir instruction. All relative dir copies are wrt to the workdir instruction
# copy python dependency wheels from python-build-stage
COPY --from=python-build-stage /usr/src/app/wheels  /wheels/

# use wheels to install python dependencies
RUN pip install --no-cache-dir --no-index --find-links=/wheels/ /wheels/* \
  && rm -rf /wheels/


COPY --chown=django:django ./compose/production/django/entrypoint /entrypoint
RUN sed -i 's/\r$//g' /entrypoint
RUN chmod +x /entrypoint


COPY --chown=django:django ./compose/production/django/start /start
RUN sed -i 's/\r$//g' /start
RUN chmod +x /start


{%- if cookiecutter.use_celery == "y" %}
COPY --chown=django:django ./compose/production/django/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker
RUN chmod +x /start-celeryworker


COPY --chown=django:django ./compose/production/django/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat
RUN chmod +x /start-celerybeat


COPY ./compose/production/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower
RUN chmod +x /start-flower
{%- endif %}


# copy application code to WORKDIR
{%- if cookiecutter.frontend_pipeline == 'Gulp' %}
COPY --from=client-builder --chown=django:django ${APP_HOME} ${APP_HOME}
{% else %}
COPY --chown=django:django . ${APP_HOME}
{%- endif %}

# make django owner of the WORKDIR directory as well.
RUN chown django:django ${APP_HOME}

USER django

ENTRYPOINT ["/entrypoint"]
