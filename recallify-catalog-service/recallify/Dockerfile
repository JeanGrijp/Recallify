# Multi-stage build for a Java 21 service (Maven)
# Stage 1: build the jar
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app
COPY pom.xml ./
# Copy sources separately to leverage Docker layer caching
COPY src ./src

# Skip tests in container build; adjust if you want to run them
RUN mvn -B -DskipTests package

# Stage 2: runtime image
FROM eclipse-temurin:21-jre

WORKDIR /app
EXPOSE 8080

# Copy the fat jar built in the previous stage
COPY --from=build /app/target/*.jar app.jar

# Use database URL from environment (set in docker-compose)
ENV DATABASE_URL=""

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
