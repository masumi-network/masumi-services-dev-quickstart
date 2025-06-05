# Masumi Services Docker Compose Setup

This repository contains a Docker Compose configuration to run both the Masumi Registry Service and Masumi Payment Service together and configure the databases

## Prerequisites

- Docker and Docker Compose installed
- A Blockfrost API key (get one from [blockfrost.io](https://blockfrost.io))

## Setup Instructions

For detailed setup instructions, please refer to the guide: https://docs.masumi.network/get-started/installation/option-1-recommended-using-masumi-services-docker-compose-setup

## Services

- Registry Service: Available at http://localhost:3000/docs (for Open-API)
- Payment Service: Available at http://localhost:3001/docs (for Open-API) or http://localhost:3001/admin (for an admin dashboard)
- PostgreSQL: Available at localhost:5432

## Database Credentials

- Username: postgres
- Password: postgres
- Databases: masumi_registry, masumi_payment

## Data Persistence

The setup includes several features to ensure data persistence:

1. **Named Volume**:

   - Database data is stored in a named volume `masumi_postgres_data`
   - This volume persists even when containers are removed
   - Data survives container restarts and updates

2. **Automatic Restart**:

   - All services are configured with `restart: unless-stopped`
   - Services will automatically restart if they crash
   - Services will restart when the system reboots

3. **Database Initialization**:
   - The `init-multiple-dbs.sh` script only runs on first startup
   - Existing data is preserved during container restarts
   - Seeding only occurs on empty databases

## Database Migration and Seeding

The services automatically handle database setup:

1. The PostgreSQL database is initialized with the required databases
2. Prisma migrations are run automatically
3. The seeding script checks if each database is empty:
   - If empty, it runs the seed script
   - If not empty, it skips seeding, therefore preserving existing data. If you want to reset, remove the volumes and start the services again.
4. The services start and is ready to use

## Managing Data

### Backup

To backup your database:

```bash
docker exec masumi-postgres pg_dumpall -U postgres > backup.sql
```

### Restore

To restore from a backup:

```bash
cat backup.sql | docker exec -i masumi-postgres psql -U postgres
```

## Stopping the Services

To stop all services (data is preserved):

```bash
docker compose down
```

To stop and remove volumes (this will delete all data):

```bash
docker compose down -v
```
