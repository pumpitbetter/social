## Mastodon on fly.io

[Mastodon](https://github.com/mastodon/mastodon) is a free, open-source social network server based on ActivityPub.

The Mastodon server is implemented a rails app, which relies on postgres and redis. It uses sidekiq for background jobs, along with a separate nodejs http streaming server.

Docker images: https://hub.docker.com/r/tootsuite/mastodon/

Dockerfile: https://github.com/mastodon/mastodon/blob/main/Dockerfile

docker-compose.yml: https://github.com/mastodon/mastodon/blob/main/docker-compose.yml

### Setup

#### App

```
$ fly apps create --name mastodon
$ fly scale memory 512 # rails needs more than 256mb
```

#### Secrets

```
$ SECRET_KEY_BASE=$(docker run --rm -it pumpitbetter/social:latest bin/rake secret)
$ OTP_SECRET=$(docker run --rm -it pumpitbetter/social:latest bin/rake secret)
$ fly secrets set OTP_SECRET=$OTP_SECRET SECRET_KEY_BASE=$SECRET_KEY_BASE
$ docker run --rm -e OTP_SECRET=$OTP_SECRET -e SECRET_KEY_BASE=$SECRET_KEY_BASE -it pumpitbetter/social:latest bin/rake mastodon:webpush:generate_vapid_key | fly secrets import
```

#### Redis server

Redis is used to store the home/list feeds, along with the sidekiq queue information. The feeds can be regenerated using `tootctl`, so persistence is [not strictly necessary](https://docs.joinmastodon.org/admin/backups/#failure).

```
$ fly apps create mastodon-redis
$ fly volumes create -c fly.redis.toml --region ord mastodon_redis
$ fly deploy --config fly.redis.toml --build-target redis-server
```

#### Storage (user uploaded photos and videos)

The `fly.toml` uses a `[mounts]` section to connect the `/opt/mastodon/public/system` folder to a persistent volume.

Create that volume below, or remove the `[mounts]` section and uncomment `[env] > S3_ENABLED` for S3 storage.

##### Option 1: Local volume

```
$ fly volumes create --region ord mastodon_uploads
```

##### Option 2: S3, etc

```
$ fly secrets set AWS_ACCESS_KEY_ID=xxx AWS_SECRET_ACCESS_KEY=yyy
```

See [lib/tasks/mastodon.rake](https://github.com/mastodon/mastodon/blob/5ba46952af87e42a64962a34f7ec43bc710bdcaf/lib/tasks/mastodon.rake#L137) for how to change your `[env]` section for Wasabi, Minio or Google Cloud Storage.

#### Postgres database

```
$ fly postgres attach --app mastodon --postgres-app pumpitbetter-postgres-cluster
$ fly deploy -c fly.setup.toml # run `rails db:setup`
```

### Deploy

```
$ fly deploy
```

#### Post install setup

https://docs.joinmastodon.org/admin/setup/

ssh to mastodon app: `flyctl ssh console`

Create a new admin account with confirmed email:
`RAILS_ENV=production tootctl accounts create tom --email tom@pumpitbetter.com --confirmed --role admin`

SMTP secrets
```
$ fly secrets set SMTP_LOGIN=[see 'Mailgun for PumpItBetter' in Keepass]
$ fly secrets set SMTP_PASSWORD=[see 'Mailgun for PumpItBetter' in Keepass]
```

### Admin

Every so often you need to manually delete older media to keep S3 size down.  To do so:

Console into your mastodon instance:
```
flyctl ssh console
```

Set PATH to ruby and mastodon executables:
```
export PATH=/opt/ruby/bin:/mastodon/bin:$PATH
```

Remove media:
```
RAILS_ENV=production tootctl media remove --days=7
```

