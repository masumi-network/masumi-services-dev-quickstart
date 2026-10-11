# Masumi Services Docker Compose Setup

This repository contains a Docker Compose configuration that runs the Masumi Registry Service and the Masumi Payment Service together on the Cardano Preprod network. It also starts one PostgreSQL database for each service.

| Component        | Version |
| ---------------- | ------- |
| Payment Service  | 0.29.0  |
| Registry Service | 0.26.0  |
| PostgreSQL       | 15      |

## Prerequisites

- Docker with Docker Compose v2.20 or later (the setup services use `dockerfile_inline`)
- A Blockfrost API key for the Preprod network (get one from [blockfrost.io](https://blockfrost.io)). The payment service seed stops without it.
- Network access to GitHub and the npm registry for the first start

## Setup Instructions

1. Copy the environment file template:

   ```bash
   cp .env.example .env
   ```

2. Edit the `.env` file and set these variables:

   - `ADMIN_KEY`: The admin key for both services. Use at least 15 characters.
   - `ENCRYPTION_KEY`: The key that encrypts the wallet secrets in the payment service database. The service requires at least 20 characters. We recommend 32 or more, for example the output of `openssl rand -hex 32`. Do not change it after the first start, because the service cannot decrypt the stored wallets with a different key.
   - `BLOCKFROST_API_KEY_PREPROD`: Your Blockfrost API key for the Preprod network.

   **Optional**

   - `PURCHASE_WALLET_PREPROD_MNEMONIC`: The mnemonic of the wallet that pays for agent requests. This wallet must have sufficient funds or you must top it up. If you do not set it, the seed generates a new mnemonic.
   - `SELLING_WALLET_PREPROD_MNEMONIC`: The mnemonic of the wallet that interacts with the smart contract. It only needs a small amount of funds for the Cardano network fees. If you do not set it, the seed generates a new mnemonic.
   - `COLLECTION_WALLET_PREPROD_ADDRESS`: The address that receives all payments after a successful purchase (not a refund). It does not need funds. We strongly recommend a hardware wallet for it. If you do not set it, the seed uses the selling wallet.

3. Start the services:

   ```bash
   docker compose up -d
   ```

   The first start builds two setup images from the release source code on GitHub. This takes some minutes. Later starts use the built images.

   On each start, the services do these steps:

   - `registry-setup` and `payment-setup` apply the database migrations and seed the databases. They seed only empty databases. Then they stop.
   - `registry-service` and `payment-service` start after their setup service completes successfully.

4. If you did not set the wallet mnemonics, read them from the payment setup log and store them securely now:

   ```bash
   docker logs masumi-payment-setup
   ```

   The log shows the generated mnemonics only after the first seed.

To see the status of the services, run `docker compose ps -a`. The two setup services show `Exited (0)` when they completed successfully.

## Services

- Registry Service: http://localhost:3000/docs (OpenAPI)
- Payment Service: http://localhost:3001/docs (OpenAPI) or http://localhost:3001/admin (admin dashboard)
- Registry database: `localhost:5432`
- Payment database: `localhost:5433`

If PostgreSQL already runs on your computer, port 5432 is not available. Change the host port of `postgres-registry` in `docker-compose.yml`, for example to `"5434:5432"`.

## Why the setup services exist

The published images contain only what the services need at runtime:

- The registry service image runs as a non-root user and does not include the Prisma schema, the migrations or the seed script.
- The payment service image does not include the workspace packages that the seed script imports.

Thus the setup services build the same release tag from source and run the migrations and the seed from there. The `payment-service` also starts with `node ./dist/index.js` directly, because the default image command (`pnpm run start`) tries to download a different pnpm version and fails in the container.

## Database Credentials

- Username: `postgres`
- Password: `postgres`
- Database: `postgres` (one PostgreSQL container for each service)

## Data Persistence

1. **Named volumes**:

   - The registry data is in the volume `masumi_postgres_data_registry`.
   - The payment data is in the volume `masumi_postgres_data_payment`.
   - The volumes stay when you remove the containers. The data stays after restarts and updates.

2. **Automatic restart**:

   - The services and the databases use `restart: unless-stopped`.
   - Docker restarts them after a crash and after a system reboot.

3. **Seeding**:
   - The seed runs only on empty databases. A restart keeps the existing data.
   - To reset the data, remove the volumes (see below) and start the services again.

## Managing Data

### Backup

To back up the databases:

```bash
docker exec masumi-postgres-registry pg_dumpall -U postgres > backup-registry.sql
```

```bash
docker exec masumi-postgres-payment pg_dumpall -U postgres > backup-payment.sql
```

### Restore

To restore from a backup:

```bash
cat backup-registry.sql | docker exec -i masumi-postgres-registry psql -U postgres
```

```bash
cat backup-payment.sql | docker exec -i masumi-postgres-payment psql -U postgres
```

## Updating to a New Release

1. Change the image tags of `registry-service` and `payment-service` in `docker-compose.yml`.
2. Change the tag in the `build.context` URL and in the `image` name of `registry-setup` and `payment-setup` to the same versions.
3. Rebuild and start the services. The setup services apply the new migrations.

   ```bash
   docker compose up -d --build
   ```

## Stopping the Services

To stop all services (the data stays):

```bash
docker compose down
```

To stop the services and remove the volumes (this deletes all data, including the encrypted wallets):

```bash
docker compose down -v
```
