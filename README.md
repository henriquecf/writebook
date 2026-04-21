# Writebook

### Instantly publish your own books on the web for free, no publisher required.

Writebook is an easy-to-use application for publishing content on the web.
Content is authored in Markdown, and books can contain picture pages, chapters, and title pages.
Books can be published privately or publicly, and are searchable.

## How to get Writebook

Writebook is distributed as a Docker image.
The simplest way to install and run it is by using [ONCE](https://github.com/basecamp/once).

To get started, paste this snippet into a terminal on the machine where you want to install Writebook:

```sh
curl https://get.once.com/writebook | sh
```

## Deploying manually with Docker

If you'd rather set the Docker image up yourself, you can use `docker run` or `docker compose` to do that.
The official image is `ghcr.io/basecamp/writebook`.

You'll need to route the incoming web traffic to ports 80 and 443 (or just 80 if you run without SSL).
To persist the storage of the application, mount a Docker volume to `/rails/storage`.

You can configure the SSL setting with the following environment variables:

- `SSL_DOMAIN` - enable automatic SSL via Let's Encrypt for the given domain name
- `DISABLE_SSL` - alternatively, set `DISABLE_SSL` to serve over plain HTTP

## Deploying with Kamal

This fork ships a `config/deploy.yml` for [Kamal 2](https://kamal-deploy.org). After filling in the host IP and proxy hostname in `config/deploy.yml` and setting `KAMAL_REGISTRY_PASSWORD` + `RAILS_MASTER_KEY` (see `.kamal/secrets`):

```sh
bin/kamal setup   # first time only
bin/kamal deploy
```

Writebook stores its four SQLite databases (primary, queue, cache, cable) plus Active Storage blobs in the `writebook_storage` volume, so it's a single-host deploy — Kamal won't scale it horizontally. Kamal-proxy handles TLS; the container runs with `DISABLE_SSL=true`.

## Running in development

Install dependencies:

```sh
bin/setup
```

Start the development server:

```sh
bin/dev
```
