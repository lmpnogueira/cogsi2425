FROM openjdk:17-jdk-slim
COPY build/libs/*.jar /app/
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]

