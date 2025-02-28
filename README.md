# Redash

Referencing [redash documentation](https://redash.io/help/open-source/setup/#docker) and testing this out for myself, I have assembled a very handy guide for setting up Redash with Docker. Hope it saves you time, or me, if I set this up again somewhere.

## Setup

1. `openssl rand -hex 32` to generate a secret key
1. `nano docker-compose.yml` (replace `#REDASH_COOKIE_SECRET` with the secret key)

   ```yml
   version: "3"
   services:
     postgres:
       image: postgres:12
       restart: always
       environment:
         POSTGRES_USER: redash
         POSTGRES_PASSWORD: redash
         POSTGRES_DB: redash
       volumes:
         - postgres-data:/var/lib/postgresql/data

     redis:
       image: redis:6
       restart: always

     redash:
       image: redash/redash:25.1.0
       depends_on:
         - postgres
         - redis
       ports:
         - "5050:5000"
       environment:
         REDASH_DATABASE_URL: "postgresql://redash:redash@postgres/redash"
         REDASH_REDIS_URL: "redis://redis:6379/0"
         REDASH_COOKIE_SECRET: "your_random_secret_key"
       command: server

     scheduler:
       image: redash/redash:25.1.0
       depends_on:
         - redash
       environment:
         REDASH_DATABASE_URL: "postgresql://redash:redash@postgres/redash"
         REDASH_REDIS_URL: "redis://redis:6379/0"
         REDASH_COOKIE_SECRET: "your_random_secret_key"
       command: scheduler

     worker:
       image: redash/redash:25.1.0
       depends_on:
         - redash
       environment:
         REDASH_DATABASE_URL: "postgresql://redash:redash@postgres/redash"
         REDASH_REDIS_URL: "redis://redis:6379/0"
         REDASH_COOKIE_SECRET: "your_random_secret_key"
       command: worker

   volumes:
     postgres-data:
   ```

   Note: The `latest` tag is actually very old (8.0.0). I chose to use version 25.1.0, which is the latest at the time of writing.

1. `docker-compose up -d` to start the services
1. `docker-compose exec redash bash` to enter the redash container
1. `bin/run ./manage.py database create_tables` to create the tables
1. `bin/run ./manage.py users create your@email 'Admin' --password admin123 --admin` to create an admin user, or just visit `http://localhost:5050` and create a user with GUI. Passwords required at least 6 characters. If using in production, make a more secure password!

**Note**: I chose to use port 5050 instead of 5000 because macOS was using that port airplay listening.

## Commands

To turn off the services:

```bash
docker-compose down
```

To start the services:

```bash
docker-compose up -d
```

To check logs:

```bash
docker-compose logs redash --tail=50

docker-compose logs worker --tail=50

docker-compose logs scheduler --tail=50
```

To handle database export:

```bash
docker-compose exec postgres pg_dump -U redash -d redash > redash-backup.sql
```

To handle database import:

```bash
cat redash-backup.sql | docker-compose exec -T postgres psql -U redash -d redash
```

To run commands inside the docker image:

```bash
docker exec -it redash-redash-1 /bin/bash
```

If you need to manually run migrations:

```bash
docker-compose exec redash bash
# then
bin/run ./manage.py db upgrade
```

## Upgrade Redash

If you need to upgrade redash, perform these steps.

**Requirements & Resources**

- See Tags on DockerHub: https://hub.docker.com/r/redash/redash/tags. _Note: the `latest` tag is actually very old (8.0.0)_
- **Upgrade 1** major version at a time.
- **Check release notes** for [each version](https://github.com/getredash/redash/releases), for potential caveat steps.
- `docker ps -a` to find names of your containers. Mine are `redash-redash-1`, `redash-scheduler-1`, `redash-worker-1`
- **Backup Database** before each upgrade.

**Steps:**

1. Dump the database: `docker-compose exec postgres pg_dump -U redash -d redash > redash-backup-current.sql`
1. Stop the services: `docker-compose down`
1. Pull the new image: `docker pull redash/redash:<version>`
1. Update the `docker-compose.yml` file with the new version
1. Run migrations: `docker-compose run --rm redash manage db upgrade`
1. Start the services: `docker-compose up -d`
1. Replacing `UPGRADE_VERSION`, take a backup of the database: `docker-compose exec postgres pg_dump -U redash -d redash > redash-backup.upgrade-to-version-{UPGRADE_VERSION}.sql`
1. If applicable, repeat steps for next version.

## Mail Setup

I did not test this as I was running locally, but I wanted to leave the easter egg here. It's also written about in the [redash setup documentation](https://redash.io/help/open-source/setup/#Mail-Configuration).

From the documentation:

> The relevant configuration variables are (note that not all of them are required):
>
> ```
> REDASH_MAIL_SERVER (default: localhost)
> REDASH_MAIL_PORT (default: 25)
> REDASH_MAIL_USE_TLS (default: false)
> REDASH_MAIL_USE_SSL (default: false)
> REDASH_MAIL_USERNAME (default: None)
> REDASH_MAIL_PASSWORD (default: None)
> REDASH_MAIL_DEFAULT_SENDER (Email address to send from)
> ```
>
> Also you need to set the value of REDASH_HOST, which is the base address of your Redash instance (the DNS name or IP) with the protocol, so for example: https://demo.redash.io.
> Once you updated the configuration, restart all services (docker-compose up -d, running docker-compose restart won’t be enough as it won’t read changes to env file). To test email configuration, you can run docker-compose run --rm server manage send_test_mail.
>
> It’s recommended to use some mail service, like Amazon SES or Mailgun to send emails to ensure deliverability.
