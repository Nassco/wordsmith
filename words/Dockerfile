FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn verify

FROM eclipse-temurin:21-alpine AS jre-build
RUN jlink \
    --add-modules java.base,java.logging,java.sql,java.naming,jdk.httpserver \
    --strip-debug \
    --no-man-pages \
    --no-header-files \
    --compress=zip-9 \
    --output /jre-minimal

FROM alpine:3.21
ENV JAVA_HOME=/jre-minimal
ENV PATH="${JAVA_HOME}/bin:${PATH}"
COPY --from=jre-build /jre-minimal /jre-minimal
COPY --from=build /app/target/words.jar /app/words.jar
COPY --from=build /app/target/dependency /app/dependency
EXPOSE 8080
CMD ["java", "-Xmx128m", "-Xms64m", "-jar", "/app/words.jar"]