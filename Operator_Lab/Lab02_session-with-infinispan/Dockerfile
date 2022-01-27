# Stage 1. Build Java Application
FROM maven:3.8-openjdk-11 as build
COPY . /tmp/src
WORKDIR /tmp/src
RUN mvn clean package \
  && cd infinispan \
  && mvn versions:use-latest-releases -DallowMajorUpdates=false \
  && mvn dependency:copy-dependencies -DoutputDirectory=target \
  && rm -f target/jboss-transaction-api*.jar

# Stage 2. Build Liberty Image
FROM ibmcom/websphere-liberty:full-java11-openj9-ubi

LABEL \
  org.opencontainers.image.authors="Your Name" \
  org.opencontainers.image.source="Your Repository URL"

COPY --chown=1001:0 --from=build /tmp/src/src/main/liberty/config /config
COPY --chown=1001:0 --from=build /tmp/src/target/guide-sessions.war /config/apps
COPY --chown=1001:0 --from=build /tmp/src/infinispan/target/*.jar /opt/ibm/wlp/usr/shared/resources/infinispan/

RUN configure.sh
