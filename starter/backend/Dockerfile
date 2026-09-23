FROM public.ecr.aws/docker/library/python:3.10-slim
ENV FLASK_RUN_HOST=0.0.0.0

WORKDIR /
RUN useradd -u 1000 -m app
RUN apt-get update && \
  apt-get install -y --no-install-recommends \
  build-essential \
  libffi-dev \
  libpcre2-8-0 \
  libpcre2-dev \
  libssl-dev \
  && pip install pipenv \
  && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /var/www/app && mkdir /app

COPY Pipfile* /app
RUN cd /app && pipenv install --system --deploy

COPY . /app

CMD uwsgi --http :5000 --master --enable-threads -s /var/www/app/app.sock --manage-script-name --mount /=app:app