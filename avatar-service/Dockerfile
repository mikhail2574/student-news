# ---- Stage 1: build the JAR with the full JDK and Maven -------------------
# Same Java major version as java.version in pom.xml (27).
FROM maven:3.10-eclipse-temurin-27 AS build
WORKDIR /app
# Only pom.xml first, so the dependency download below is its own layer and is
# reused from cache as long as pom.xml does not change.
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
# The source comes after the dependencies: editing Java code only invalidates
# the layers from here on.
COPY src ./src
# Tests already run in CI ("Avatar service · tests"); the image only needs the JAR.
RUN mvn -B -q clean package -DskipTests

# ---- Stage 2: run it on a small JRE-only Alpine image ---------------------
FROM eclipse-temurin:27-jre-alpine
WORKDIR /app
# Unprivileged user with a fixed numeric id (Kubernetes can check runAsNonRoot).
RUN addgroup -S -g 10001 app && adduser -S -u 10001 -G app app
# The upload directory must exist and belong to the app user: the app cannot
# create /data itself, and an empty named volume copies this ownership.
RUN mkdir -p /data/avatars && chown -R app:app /data
# Only the JAR from the build stage; it stays owned by root, so it is read-only for the app.
COPY --from=build /app/target/avatar-service.jar app.jar
USER 10001:10001
EXPOSE 8090
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
