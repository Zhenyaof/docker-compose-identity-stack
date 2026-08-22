# Docker Commands — Identity Stack (Postgres + Keycloak + Curl)

Written for **Windows `cmd.exe`** (run these from inside
`C:\Users\Zhenya\docker-compose-identity-stack`).

## 1. One-time setup — create the persistent data folder

```cmd
if not exist data\postgres mkdir data\postgres
```

This is the custom, non-default location the Postgres bind mount
points to.

## 2. Start the stack

```cmd
docker compose up -d
```

Compose starts `postgres` first, waits until its healthcheck passes,
then starts `keycloak`, waits until *its* healthcheck passes, and
only then starts `curl` (which then just idles forever).

## 3. Check status / health of every service

```cmd
docker compose ps
```

You should see `healthy` next to postgres and keycloak, and `running`
(Up) for curl once both are healthy.

## 4. Watch it come up live

```cmd
docker compose logs -f
```

Logs won't be JSON — they use the `local` logging driver, rotated
automatically (10MB/5MB per file, max 20 files kept, oldest ones
dropped).

## 5. Verify curl really waited for both healthchecks

```cmd
docker inspect --format="{{.State.StartedAt}}" postgres
docker inspect --format="{{.State.StartedAt}}" keycloak
docker inspect --format="{{.State.StartedAt}}" curl-client
```

`curl-client`'s StartedAt should be later than both of the others.

## 6. Test connectivity from inside the curl container

```cmd
docker exec -it curl-client curl -sf http://keycloak:8080/health/ready
```

## 7. Open Keycloak login page (the only thing exposed to the host)