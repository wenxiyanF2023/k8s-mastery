FROM openjdk:8-jdk-alpine
FROM maven:3.8.4-jdk-8-slim AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

FROM openjdk:8-jdk-alpine
WORKDIR /app
COPY --from=build /app/target/sentiment-analysis-web-0.0.1-SNAPSHOT.jar ./app.jar
ENV SA_LOGIC_API_URL http://localhost:5000
EXPOSE 8080
CMD ["java", "-jar", "app.jar", "--sa.logic.api.url=${SA_LOGIC_API_URL}"]