# MagnetPlay - Quick Start

## Clone

```bash
git clone https://github.com/SebastianVi1/magnetPlay-deployment.git
cd magnetPlay-deployment
git submodule update --init --recursive
```

## Development

```bash
docker compose up --build -d
```

Access: Frontend at http://localhost:5173, Backend at http://localhost:8080

## Production

```bash
docker compose -f docker-compose.prod.yml --env-file ./.env.prod up --build -d
```

Access: Frontend at http://localhost (port 80)

### Build with specific tag

```bash
export TAG='v1.0.0'
docker compose -f docker-compose.prod.yml --env-file ./.env.prod build --no-cache
docker compose -f docker-compose.prod.yml --env-file ./.env.prod up -d
```

## Update & Redeploy

### Dev

```bash
git pull origin master
git submodule update --remote --merge --recursive
docker compose up --build --force-recreate -d
```

### Prod

```bash
git pull origin master
git submodule update --remote --merge --recursive
export TAG='v1.0.1'
docker compose -f docker-compose.prod.yml --env-file ./.env.prod build --no-cache
docker compose -f docker-compose.prod.yml --env-file ./.env.prod up --force-recreate -d
```
