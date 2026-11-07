# syntax=docker/dockerfile:1

# Base images are pinned by version and digest for reproducible builds; Dependabot bumps both.

# ---- Build stage ------------------------------------------------------------
# Builds the executable jar. Quality gates (Checkstyle, SpotBugs, tests, JaCoCo) already run in CI
# via `./mvnw verify` before an image is ever built here, so this stage only packages the artifact.
FROM eclipse-temurin:25.0.5_8-jdk-noble@sha256:95e350ad1eb738f943e141640f7e3d797e483359a3b33c3864255bfd1ad71f8a AS build
WORKDIR /workspace

# Copied and resolved before the source tree, so this layer is only invalidated when a dependency
# actually changes rather than on every code change.
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN chmod +x mvnw && ./mvnw --batch-mode --no-transfer-progress dependency:go-offline

# Checkstyle (bound to the `validate` phase, still run below despite -DskipTests) reads its config
# from here.
COPY config config
COPY src src
RUN ./mvnw --batch-mode --no-transfer-progress -DskipTests clean package \
  && cp target/littlewords-*.jar littlewords.jar \
  && java -Djarmode=tools -jar littlewords.jar extract --layers --destination extracted

# ---- Runtime stage ------------------------------------------------------------
FROM eclipse-temurin:25.0.5_8-jre-noble@sha256:a98128a57de944a72bd31fbc79b898edd5a5864d3652071d5bbd2160ab7d7dce AS runtime
WORKDIR /app

# Unprivileged user with a fixed numeric id, so a runtime can check it is not root without a passwd lookup.
RUN groupadd --system --gid 10001 littlewords && useradd --system --uid 10001 --gid 10001 littlewords
USER 10001:10001

# One layer per Spring Boot layer, least to most frequently changed: a code change only ships the last one.
COPY --from=build /workspace/extracted/dependencies/ ./
COPY --from=build /workspace/extracted/spring-boot-loader/ ./
COPY --from=build /workspace/extracted/snapshot-dependencies/ ./
COPY --from=build /workspace/extracted/application/ ./

# 8080 serves the API and /livez, /readyz; 8081 serves the actuator and must never be routed publicly.
EXPOSE 8080 8081

# Probes the main port, so a stuck API fails even if the management port answers. No curl in this
# image, so bash speaks HTTP over /dev/tcp. The start period covers JVM startup and Flyway migrations.
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD ["bash", "-c", "exec 3<>/dev/tcp/127.0.0.1/8080 && printf 'GET /readyz HTTP/1.0\\r\\nHost: localhost\\r\\n\\r\\n' >&3 && read -r status <&3 && [[ $status == *' 200 '* ]]"]

# Heap takes 60 % of the container memory limit; the rest is for metaspace, threads and buffers.
# An OutOfMemoryError exits the JVM, so the restart policy brings back a clean one instead of a half-dead one.
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=60", "-XX:+ExitOnOutOfMemoryError", "-jar", "littlewords.jar"]
