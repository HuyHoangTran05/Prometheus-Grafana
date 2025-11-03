# syntax=docker/dockerfile:1

# ---------- Build stage ----------
FROM eclipse-temurin:17-jdk AS build
WORKDIR /workspace

# Copy Gradle wrapper and build files first for better layer caching
COPY gradlew gradlew.bat ./
COPY gradle ./gradle
COPY build.gradle settings.gradle ./

# Copy the rest of the source
COPY . .

# Normalize line endings for Windows checkouts and make wrapper executable, then warm up Gradle
RUN sed -i 's/\r$//' gradlew && chmod +x gradlew && ./gradlew --version

# Build the Spring Boot fat jar (skip tests for faster image builds)
RUN ./gradlew clean bootJar -x test

# ---------- Runtime stage ----------
FROM eclipse-temurin:17-jre AS runtime
WORKDIR /app

# Copy the built jar
COPY --from=build /workspace/build/libs/*.jar app.jar

# Expose the app port
EXPOSE 8080

# Allow optional JVM opts via JAVA_OPTS
ENV JAVA_OPTS=""

# Start the application
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
