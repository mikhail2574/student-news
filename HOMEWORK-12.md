# Homework 12 — Containerising avatar-service

Files changed: `avatar-service/Dockerfile`, `avatar-service/.dockerignore`, `docker-compose.yml`,
and `.github/workflows/ci.yml` (the `package` job now also builds the `avatar-service` image).

---

## Part 1 — The Dockerfile

```dockerfile
FROM maven:3.10-eclipse-temurin-27 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B -q clean package -DskipTests

FROM eclipse-temurin:27-jre-alpine
WORKDIR /app
RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app
RUN mkdir -p /data/avatars && chown -R app:app /data
COPY --from=build /app/target/avatar-service.jar app.jar
USER 10001:10001
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

### Line by line

**Build stage**

| Line | What it does and why |
|------|----------------------|
| `FROM maven:3.10-eclipse-temurin-27 AS build` | Starts the first stage from an image that has both Maven and a full JDK 27, the same Java version as `java.version` in `pom.xml`. It is named `build` so the second stage can copy from it; nothing from this heavy image ends up in the final one. |
| `WORKDIR /app` | Creates `/app` and makes it the current directory, so the following `COPY` and `RUN` commands use short relative paths. |
| `COPY pom.xml .` | Copies only the dependency list into the image. On its own it is a small, rarely changing layer, which is what makes the next line cacheable. |
| `RUN mvn -B -q dependency:go-offline` | Downloads every dependency and plugin from `pom.xml` into the image's local Maven repository. Because only `pom.xml` is present, this layer stays cached until `pom.xml` changes, so editing Java code does not download the internet again. `-B` is batch mode (no colours/progress), `-q` keeps the log short. |
| `COPY src ./src` | Copies the source code. It comes after the dependency layer on purpose: a source change invalidates only this layer and the ones after it. |
| `RUN mvn -B -q clean package -DskipTests` | Compiles the code and builds `target/avatar-service.jar` (the name comes from `<finalName>` in `pom.xml`). Tests are skipped here because CI already runs them in the "Avatar service · tests" job, and as root inside the build some of them would be skipped anyway. |

**Runtime stage**

| Line | What it does and why |
|------|----------------------|
| `FROM eclipse-temurin:27-jre-alpine` | Starts the final image from a JRE-only Alpine image: no compiler, no Maven, no source code. It is much smaller and has less software that could have vulnerabilities. |
| `WORKDIR /app` | Directory where the JAR will live and the process will start. |
| `RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app` | Creates a system group and user `app` with a fixed id 10001 (Alpine/busybox syntax). The service will run as this user instead of root; the fixed numeric id lets tools like Kubernetes check `runAsNonRoot`. |
| `RUN mkdir -p /data/avatars && chown -R app:app /data` | Creates the upload directory and gives it to `app`. The service runs as a normal user and cannot create a directory under `/` itself, and when an empty named volume is mounted here Docker copies this ownership into the volume. Without it `/health` returns 503 (see "Break it once"). |
| `COPY --from=build /app/target/avatar-service.jar app.jar` | Takes only the finished JAR from the build stage. It stays owned by root, so the app can read it but cannot overwrite its own code. |
| `USER 10001:10001` | From here on, and when the container runs, the process runs as `app`, not root. The numeric form works even without looking up `/etc/passwd`. |
| `EXPOSE 8090` | Documents that the service listens on 8090 (the default `PORT`). It does not publish the port; Compose does that with `ports:`. |
| `ENTRYPOINT ["java", "-jar", "/app/app.jar"]` | Starts the service. The exec (JSON) form makes `java` PID 1, so it gets `SIGTERM` directly from `docker stop` and can shut down cleanly. |

### `.dockerignore`

```
target/   data/   .idea/   .vscode/   *.iml   logs/   *.log   .git
```

`target/` is build output that is rebuilt inside the image anyway; `data/` is where local uploads go
when the service is run without Docker; `.idea/`, `.vscode/`, `*.iml` are IDE files; `logs/`, `*.log`
are logs; `.git` is history. None of them are needed to build, and leaving them out keeps the build
context small and stops them from invalidating the `COPY` cache or leaking into the image.

### What gets rebuilt

- **One Java file changes:** `FROM`, `WORKDIR`, `COPY pom.xml` and `dependency:go-offline` come from
  cache. `COPY src` and `mvn package` run again, and in the runtime stage only `COPY --from=build`
  is redone (user creation and `mkdir` stay cached). Dependencies are not downloaded again.
  (Checked with `docker compose build --progress=plain`: steps 2–4 of the build stage were `CACHED`,
  steps 5–6 ran again.)
- **`pom.xml` changes:** the cache breaks at `COPY pom.xml`, so `dependency:go-offline` downloads
  all dependencies again, then `COPY src`, `mvn package` and the final `COPY --from=build` run again.
  Only the base images and the runtime `addgroup`/`mkdir` layers are reused.

### Break it once

With `RUN mkdir -p /data/avatars && chown -R app:app /data` removed, after
`docker compose down -v && docker compose up --build -d avatar-service`:

```
$ curl http://localhost:8090/health
{"status":"error","detail":"upload directory /data/avatars is not writable: permission denied (the process runs as uid 10001, gid 10001)"}
```

HTTP status 503, and `docker compose ps` showed the container as `unhealthy`. Inside the container:

```
drwxr-xr-x 2 root root 4096 /data/avatars
```

Running the same broken image without a volume (`docker run`) gives
`{"status":"error","detail":"upload directory /data/avatars is missing"}`.

**Why the line is needed:** the service runs as uid 10001, which has no write access to `/`, so at
startup it cannot create `/data/avatars` itself. With a volume, Docker creates the missing mount
point, but owned by root, so the app still cannot write. When the directory already exists in the
image and belongs to `app`, Docker copies that ownership into the empty named volume on first use,
and the service can store pictures. After putting the line back: `{"status":"ok"}`.

---

## Part 2 — docker-compose.yml

```yaml
  avatar-service:
    build:
      context: ./avatar-service
    container_name: student-news-avatar-service
    environment:
      PORT: 8090
      UPLOAD_DIR: /data/avatars
      MAX_BYTES: 2097152
    ports:
      - "8090:8090"
    volumes:
      - avatar_data:/data/avatars
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "/dev/null", "http://localhost:8090/health"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s

volumes:
  postgres_data:
  avatar_data:
```

| Key | Why |
|-----|-----|
| `build.context: ./avatar-service` | Compose builds the image from `avatar-service/Dockerfile` with that folder as build context (filtered by its `.dockerignore`). |
| `container_name` | A fixed, readable name in the same style as the other services (`student-news-…`). |
| `environment.PORT` | Port the service listens on inside the container; must match the right side of `ports:` and the health check URL. |
| `environment.UPLOAD_DIR` | Where pictures are written; must be the path the volume is mounted at, otherwise files would be written to the container's own layer and lost. |
| `environment.MAX_BYTES` | Largest accepted upload (2 MiB). Set explicitly so the limit is visible and can be changed without rebuilding. |
| `ports: "8090:8090"` | Publishes container port 8090 on the host's port 8090, so the browser can reach it. |
| `volumes: avatar_data:/data/avatars` | Mounts the named volume over the upload directory, so the files live outside the container. |
| top-level `volumes: avatar_data:` | Declares the named volume; Compose creates it (`student-news_avatar_data`) and keeps it until `down -v`. |
| `healthcheck.test` | Runs busybox `wget` inside the container against `/health`. `wget` exits non-zero on a 503 or no answer, which marks the check as failed. The image has no `curl`. |
| `interval` / `timeout` / `retries` | Check every 10 s, give up on a check after 3 s, call the container `unhealthy` after 5 failures in a row. |
| `start_period` | Spring Boot takes a few seconds to start; failures during the first 20 s do not count. |

**Why the UI uses `localhost:8090`, not `avatar-service:8090`.** The React app is not running in the
frontend container: nginx only serves the static files, and the JavaScript then runs in the user's
browser on the host. The browser is not on the Compose network and cannot resolve the service name
`avatar-service`; that name only exists in Compose's internal DNS. From the host the service is
reachable through the published port, `localhost:8090`. (The backend, in contrast, talks to `db:5432`
because it runs inside the network.)

**What the named volume protects you from.** A container's own filesystem is thrown away when the
container is removed, which happens on `docker compose down`, on `up --build` with a new image, or
when the container is recreated. Without the volume every uploaded picture would disappear with it.
The named volume is managed by Docker, independent of any container, so the pictures survive
restarts, rebuilds and `down`, and are deleted only on purpose with `down -v` or `docker volume rm`.
It also avoids the permission/path problems of bind-mounting a host folder.

**What Docker does with the health check.** Docker runs the command periodically and stores the
result as the container's health status: `starting` → `healthy`, or `unhealthy` after `retries`
failures. It shows up in `docker compose ps` and `docker inspect`. Compose uses it for
`depends_on: condition: service_healthy` (other services can wait until the service is really
ready, like the backend waits for `db`), and `docker compose up --wait` waits for it. Plain Docker
does **not** restart an unhealthy container by itself; orchestrators (Swarm, or Kubernetes with its
own probes) act on it.

---

## Part 3 — Proof

### 1. `docker compose up --build -d` and `docker compose ps`

```
NAME                          IMAGE                         SERVICE          STATUS                    PORTS
student-news-avatar-service   student-news-avatar-service   avatar-service   Up 43 seconds (healthy)   0.0.0.0:8090->8090/tcp, [::]:8090->8090/tcp
student-news-backend          student-news-backend          backend          Up 37 seconds             0.0.0.0:8080->8080/tcp, [::]:8080->8080/tcp
student-news-db               postgres:16-alpine            db               Up 43 seconds (healthy)   127.0.0.1:5432->5432/tcp
student-news-frontend         student-news-frontend         frontend         Up 37 seconds             0.0.0.0:5173->8080/tcp, [::]:5173->8080/tcp
```

### 2. `./avatar-service/smoke-test.sh`

```
Smoke-testing avatar-service at http://localhost:8090
ok    GET /health -> 200
ok    POST /avatars/smoke-test-2008 -> 201
ok    GET /avatars/smoke-test-2008 -> 200
ok    downloaded image matches the upload byte for byte
ok    DELETE /avatars/smoke-test-2008 -> 204
ok    GET /avatars/smoke-test-2008 after delete -> 404
PASS: avatar-service at http://localhost:8090 works
```

### 3. News post with a picture

![News post with a picture](hw12-ui.png)

### 4. `docker compose exec avatar-service whoami`

```
app
```

(`id` → `uid=10001(app) gid=10001(app) groups=10001(app)`; `/data/avatars` is `app app`.)

### 5. `down` vs `down -v`

```
$ docker compose down && docker compose up -d
$ curl -o /dev/null -w '%{http_code}' http://localhost:8090/avatars/news-1
200            # 49711 bytes, the same picture; the post is still in the list

$ docker compose down -v && docker compose up -d
$ curl http://localhost:8090/avatars/news-1
{"error":"Not Found","detail":"no avatar stored for id news-1"}     # HTTP 404
$ curl http://localhost:8080/api/news
[]
```

`docker compose down` removes the containers and the network but keeps named volumes, so the new
`avatar-service` container mounts the same `avatar_data` volume and finds the picture (and Postgres
finds the post in `postgres_data`). `docker compose down -v` also removes the named volumes declared
in the file, so both the pictures and the database are deleted, and the next `up` starts with empty
volumes.

### 6. GitHub Actions

![GitHub Actions run passing](hw12-actions.png)
