# Étape 1 : Build Java
FROM gradle:8.14-jdk21 AS build

WORKDIR /app
COPY . .

RUN gradle build --no-daemon -x test

# Étape 2 : Runtime non-root
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=build /app/build/libs/*.jar app.jar

RUN adduser -D -u 1001 appuser \
    && chown -R appuser:appuser /app

USER 1001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
