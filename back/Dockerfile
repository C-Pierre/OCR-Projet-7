FROM gradle:8-jdk17-alpine AS build

WORKDIR /app
COPY . .

RUN chmod +x gradlew \
    && ./gradlew build --no-daemon

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=build /app/build/libs/microcrm-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8080

USER 1001

ENTRYPOINT ["java","-jar","/app/app.jar"]
