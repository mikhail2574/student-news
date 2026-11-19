# Homework 12 — avatar-service in Docker

## Part 1 — Dockerfile

| Line | What and why |
|------|--------------|
| `FROM maven:3.10-eclipse-temurin-27 AS build` | Build stage with Maven and JDK 27 (same Java as `pom.xml`); named so the final stage can copy from it. |
| `WORKDIR /app` | Working directory for the build. |
| `COPY pom.xml .` | Only the dependency list first, so the next layer can be cached. |
| `RUN mvn -B -q dependency:go-offline` | Downloads all dependencies in their own layer; reused until `pom.xml` changes. |
| `COPY src ./src` | Source after dependencies: a code change invalidates only the layers from here on. |
| `RUN mvn -B -q clean package -DskipTests` | Builds `target/avatar-service.jar`; tests already run in CI. |
| `FROM eclipse-temurin:27-jre-alpine` | Minimal runtime: JRE only on Alpine, no JDK/Maven/source. |
| `WORKDIR /app` | Where the JAR lives and the process starts. |
| `RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app` | Creates an unprivileged user with a fixed numeric id. |
| `RUN mkdir -p /data/avatars && chown -R app:app /data` | Upload directory owned by `app`; the app cannot create it itself, and an empty named volume copies this ownership. |
| `COPY --from=build /app/target/avatar-service.jar app.jar` | Only the JAR from the build stage; owned by root, so read-only for the app. |
| `USER 10001:10001` | The container runs as `app`, not root. |
| `EXPOSE 8090` | Documents the port the service listens on (publishing is done by Compose). |
| `ENTRYPOINT ["java", "-jar", "/app/app.jar"]` | Starts the service; exec form makes java PID 1 so it receives `SIGTERM` on stop. |

`.dockerignore` excludes `target/` (build output), `data/` (local uploads), `.idea/`, `.vscode/`, `*.iml` (IDE), `logs/`, `*.log` (logs) and `.git`: smaller context, no cache busting, nothing local in the image.

**One Java file changed:** dependency layers are cached; only `COPY src`, `mvn package` and the final `COPY --from=build` run again.
**`pom.xml` changed:** cache breaks at `COPY pom.xml`, so dependencies are downloaded again, then everything after it is rebuilt.

**Break it once** — without the `mkdir` line:
```
$ curl http://localhost:8090/health
{"status":"error","detail":"upload directory /data/avatars is not writable: permission denied (the process runs as uid 10001, gid 10001)"}
```
HTTP 503, container `unhealthy`. The app runs as uid 10001 and cannot create directories under `/`; Docker creates the missing mount point owned by root. With the directory in the image owned by `app`, the empty volume inherits that ownership and the app can write. Line restored → `{"status":"ok"}`.

## Part 2 — docker-compose.yml

| Key | Why |
|-----|-----|
| `build.context: ./avatar-service` | Build the image from that folder's Dockerfile. |
| `environment: PORT / UPLOAD_DIR / MAX_BYTES` | Listen port (matches `ports` and healthcheck), upload dir (must equal the volume path), max upload size (2 MiB), all visible and changeable without rebuilding. |
| `ports: "8090:8090"` | Publishes the service on the host for the browser. |
| `volumes: avatar_data:/data/avatars` + top-level `avatar_data:` | Pictures are stored in a Docker-managed named volume, outside the container. |
| `healthcheck` (`wget -q -O /dev/null http://localhost:8090/health`, interval 10s, timeout 3s, retries 5, start_period 20s) | No curl in the image; busybox `wget` fails on 503 or no answer. `start_period` gives Spring Boot time to start. |

- **Why `localhost:8090`:** the UI's JavaScript runs in the browser on the host, not in a container. The browser is not on the Compose network and cannot resolve `avatar-service`; it reaches the service through the published port.
- **Named volume protects from:** losing all pictures when the container is removed or recreated (`down`, `up --build`). Data is removed only on purpose (`down -v`).
- **Health check result:** Docker marks the container `starting` / `healthy` / `unhealthy` (shown in `docker compose ps`). Compose uses it for `depends_on: condition: service_healthy` and `up --wait`. Plain Docker does not restart an unhealthy container.

## Part 3 — Proof

**`docker compose ps`**
```
NAME                          SERVICE          STATUS                    PORTS
student-news-avatar-service   avatar-service   Up 43 seconds (healthy)   0.0.0.0:8090->8090/tcp
student-news-backend          backend          Up 37 seconds             0.0.0.0:8080->8080/tcp
student-news-db               db               Up 43 seconds (healthy)   127.0.0.1:5432->5432/tcp
student-news-frontend         frontend         Up 37 seconds             0.0.0.0:5173->8080/tcp
```

**`./avatar-service/smoke-test.sh`**
```
ok    GET /health -> 200
ok    POST /avatars/smoke-test-2008 -> 201
ok    GET /avatars/smoke-test-2008 -> 200
ok    downloaded image matches the upload byte for byte
ok    DELETE /avatars/smoke-test-2008 -> 204
ok    GET /avatars/smoke-test-2008 after delete -> 404
PASS: avatar-service at http://localhost:8090 works
```

**UI with a picture**

![UI](hw12-ui.png)

**`docker compose exec avatar-service whoami`** → `app` (uid 10001)

**`down` vs `down -v`:** after `down` + `up -d` the picture is still there (`GET /avatars/news-1` → 200); after `down -v` + `up -d` it is gone (404, news list empty). `down` removes containers and network but keeps named volumes; `down -v` also deletes the volumes, so pictures and the database are lost.

**GitHub Actions**

![Actions](hw12-actions.png)
