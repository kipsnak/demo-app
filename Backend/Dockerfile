###############
# Build Stage #
###############

FROM docker.io/openjdk:17-jdk-alpine AS Builder
LABEL stage="Builder"
ENV APPHOME=/build/app

RUN mkdir -p ${APPHOME}
WORKDIR ${APPHOME}

COPY mvnw .
COPY .mvn .mvn
COPY pom.xml .
COPY src src

RUN ./mvnw package -DskipTests

###############
# Build Stage #
###############
FROM docker.io/openjdk:17-jdk-alpine
LABEL stage=Runtime
COPY --from=Builder /build/app/target/*.jar /app/runme.jar
EXPOSE 8080
ENTRYPOINT java -jar /app/runme.jar