# Stage 1: Build the application using a Maven container
FROM maven:3.8.5-openjdk-11 AS build

WORKDIR /app
# Copy pom.xml and .mvn to leverage Docker layer caching
COPY .mvn/ .mvn
COPY mvnw .
RUN chmod +x mvnw
COPY pom.xml .
# Download dependencies
RUN ./mvnw dependency:go-offline

# Copy the rest of the source code and build the application
COPY src/ src/
RUN ./mvnw clean package -DskipTests

# Stage 2: Create the final, smaller image
FROM openjdk:11-jre-slim
WORKDIR /app

# Copy the built JAR from the build stage
COPY --from=build /app/target/*.jar app.jar

# Expose the port the application runs on (default for Spring Boot is 8080)
EXPOSE 8080

# Command to run the application
ENTRYPOINT ["java","-jar","app.jar"]