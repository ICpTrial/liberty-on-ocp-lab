# Stage 1. Build Java Application
FROM maven:3.8-openjdk-11 as build
COPY . /tmp/src
WORKDIR /tmp/src
RUN mvn clean package

# Stage 2. Build Liberty Custom Image
FROM ibmcom/websphere-liberty:full-java11-openj9-ubi

LABEL \
  org.opencontainers.image.authors="Your Name" \
  org.opencontainers.image.source="Your Repository URL"

USER root
RUN dnf update -y 
RUN dnf install -y glibc-locale-source \
  && localedef -i ja_JP -c -f UTF-8 -A /usr/share/locale/locale.alias ja_JP.UTF-8 \
  && echo LANG=ja_JP.UTF-8 > /etc/locale.conf \
  && dnf remove -y glibc-locale-source \
  && dnf clean all \
  && ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime

USER default

COPY --chown=1001:0 --from=build /tmp/src/DefaultApplication-ear/target/liberty/wlp/usr/servers/DefaultApplicationServer/ /config
COPY --chown=1001:0 --from=build /tmp/src/DefaultApplication-ear/target/liberty/wlp/usr/shared/resources/derby-*.jar /opt/ibm/wlp/usr/shared/resources

RUN configure.sh
