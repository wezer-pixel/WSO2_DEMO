# Use a consistent, slim Java runtime as the base image
FROM openjdk:11-jre-slim

# The build context will be the service directory (e.g., backends/soap-service/)
# The Maven build runs in this directory, so the artifact will be in the target/ subdirectory.
ARG ARTIFACT_PATH=target/*.war

# Copy the built artifact from the build context into the container
COPY ${ARTIFACT_PATH} app.war

# Expose the default port for Spring Boot applications
EXPOSE 8080

# Command to run the application when the container starts
ENTRYPOINT ["java", "-jar", "/app.war"]