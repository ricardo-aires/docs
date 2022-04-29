# Non-Official Confluent Docker Images on ARM

Currently the [Official Confluent Docker Images](https://hub.docker.com/u/confluentinc) are only made available for `linux/amd64` architecture.

When running in the Apple M1 ARM-based chip, these images need to be emulated and in this case break:

- [Docker Desktop for Apple silicon known-issues](https://docs.docker.com/desktop/mac/apple-silicon/#known-issues)
- [Confluent common-docker ARM Support](https://github.com/confluentinc/common-docker/issues/117)

To help my team maintain the current labs, some of the images and versions from confluent where build on ARM using the official git repositories from Confluent:

- [confluentinc/common-docker](https://github.com/confluentinc/common-docker)
- [confluentinc/kafka-images](https://github.com/confluentinc/kafka-images)
- [confluentinc/schema-registry-images](https://github.com/confluentinc/schema-registry-images)
- [confluentinc/ksql-images](https://github.com/confluentinc/ksql-images)
- [confluentinc/control-center-images](https://github.com/confluentinc/control-center-images)

## ARM Images

Currently the images available are:

| component                         | arm image                                                                                                         | official image                                                                                                  | versions         |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ---------------- |
| Zookeeper                         | [ricardo7aires/cp-zookeeper](https://hub.docker.com/r/ricardo7aires/cp-zookeeper)                                 | [confluentinc/cp-zookeeper](https://hub.docker.com/r/confluentinc/cp-zookeeper)                                 | `6.1.0`, `7.1.1` |
| Kafka (Community Version)         | [ricardo7aires/cp-kafka](https://hub.docker.com/r/ricardo7aires/cp-kafka)                                         | [confluentinc/cp-kafka](https://hub.docker.com/r/confluentinc/cp-kafka)                                         | `6.1.0`, `7.1.1` |
| Enterprise Kafka Distribution     | [ricardo7aires/cp-server](https://hub.docker.com/r/ricardo7aires/cp-server)                                       | [confluentinc/cp-server](https://hub.docker.com/r/confluentinc/cp-server)                                       | `6.1.0`, `7.1.1` |
| Kafka Connect (Community Version) | [ricardo7aires/cp-kafka-connect](https://hub.docker.com/r/ricardo7aires/cp-kafka-connect)                         | [confluentinc/cp-kafka-connect](https://hub.docker.com/r/confluentinc/cp-kafka-connect)                         | `6.1.0`, `7.1.1` |
| Enterprise Kafka Connect          | [ricardo7aires/cp-server-connect](https://hub.docker.com/r/ricardo7aires/cp-server-connect)                       | [confluentinc/cp-server-connect](https://hub.docker.com/r/confluentinc/cp-server-connect)                       | `6.1.0`, `7.1.1` |
| Schema Registry                   | [ricardo7aires/cp-schema-registry](https://hub.docker.com/r/ricardo7aires/cp-schema-registry)                     | [confluentinc/cp-schema-registry](https://hub.docker.com/r/confluentinc/cp-schema-registry)                     | `6.1.0`, `7.1.1` |
| KSQL-DB Server                    | [ricardo7aires/cp-ksqldb-server](https://hub.docker.com/r/ricardo7aires/cp-ksqldb-server)                         | [confluentinc/cp-ksqldb-server](https://hub.docker.com/r/confluentinc/cp-ksqldb-server)                         | `6.1.0`, `7.1.1` |
| KSQL-DB cli                       | [ricardo7aires/cp-ksqldb-cli](https://hub.docker.com/r/ricardo7aires/cp-ksqldb-cli)                               | [confluentinc/cp-ksqldb-cli](https://hub.docker.com/r/confluentinc/cp-ksqldb-cli)                               | `6.1.0`, `7.1.1` |
| Control Center                    | [ricardo7aires/cp-enterprise-control-center](https://hub.docker.com/r/ricardo7aires/cp-enterprise-control-center) | [confluentinc/cp-enterprise-control-center](https://hub.docker.com/r/confluentinc/cp-enterprise-control-center) | `6.1.0`, `7.1.1` |

Because they are being build from the source of Confluent they are compatible with the demos provided by them and charts/operators. Compatible, not the same as supported. These are for lab purpose, not production.

## Build Process

### Requirements

Used at the time of build:

- macOS Monterey 12.3.1
- [Docker Desktop for Mac](https://docs.docker.com/desktop/mac/install/) - Apple Chip 4.7.1 (77678)
- socat v1.7.4.3
- maven v3.8.5
- tox v3.25.0

> socat, maven and tox installed via [brew](http://brew.sh).

### Pre-Steps

After the software required is installed, we need to set some maven mirrors to avoid build errors, if maven was installed via brew this is done by editing `/opt/homebrew/Cellar/maven/3.8.5/libexec/conf/settings.xml` and adding in the mirror list

```xml
<mirror>
    <id>confluent</id>
    <mirrorOf>confluent</mirrorOf>
    <name>Nexus public mirror</name>
    <url>http://packages.confluent.io/maven/</url>
</mirror>
```

The kafka Docker build is based on the Maven plugin `dockerfile-maven-plugin`, which doesn't support ARM. As a workaround we need to expose the docker daemon sock in a TCP port:

```bash
socat TCP-LISTEN:2375,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock
```

Run the above command and leave that window open while doing the builds. On the windows performing the build make sure to change the variable for the docker host:

```bash
export DOCKER_HOST=tcp://127.0.0.1:2375
```

### Build Base image

1. Clone the [confluentinc/common-docker](https://github.com/confluentinc/common-docker) git repository
1. Change to the desired version

    ```bash
    git checkout tags/v7.1.1
    ```

1. Add the next repository on the root `pom.xml`

    ```xml
    <repositories>
        <repository>
            <id>confluent</id>
            <url>https://packages.confluent.io/maven/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
    ```

1. Beware of the versions hardcoded in the same `pom.xml`, some may not be available for the ARM, some required changes in time of writing:

    ```xml
    <ubi.openssl.version>1.1.1k-6.el8_5</ubi.openssl.version>
    <ubi.zulu.openjdk.version>11.0.15</ubi.zulu.openjdk.version>
    ```

    > On older versions these last step the version are hardcoded on the `base/Dockerfile.ubi8`

1. On the file `base/tox.ini` remove the versions of `pytest`, `pytest-xdist` and `pytest-cov` to avoid conflicts.
1. On older tags you may need to update pip inside the `base/Dockerfile.ubi8`, you can check the latest to reference, but you can add the next in the `RUN` statement:

```Dockerfile
...
RUN microdnf --nodocs install yum \
...
    && alternatives --set python /usr/bin/python3 \
    && python3 -m pip install --upgrade "pip${PYTHON_PIP_VERSION}" "setuptools${PYTHON_SETUPTOOLS_VERSION}" \
...
```

1. Run the build

```bash
mvn clean package \
  -Pdocker -DskipTests \
  -Ddocker.registry=local/
```

> The `pom.xml` has a flag to validate if the Yum/Dnf package manager detects that there is security update available and by default it will fail if it has. One can disabled or take action when detected. When I run there was an update for the `zlib` hence I changed the Dockerfile to updated it.

### Build Zookeeper, Kafka and Kafka Connect

1. Clone the [confluentinc/kafka-images](https://github.com/confluentinc/kafka-images) git repository
1. Change to the desired version

    ```bash
    git checkout tags/v7.1.1
    ```

1. Add the next repository on the root `pom.xml`

    ```xml
    <repositories>
        <repository>
            <id>confluent</id>
            <url>https://packages.confluent.io/maven/</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
    ```

1. Run the build

```bash
mvn clean package \
  -Pdocker -DskipTests \
  -DCONFLUENT_PACKAGES_REPO='https://packages.confluent.io/rpm/7.1' \
  -DCONFLUENT_VERSION=7.1.1 \
  -Ddocker.registry=local/
```

The same steps can be done on other repos in order to build some images, remember to do the common first:

- [confluentinc/schema-registry-images](https://github.com/confluentinc/schema-registry-images) to build the schema registry image
- [confluentinc/ksql-images](https://github.com/confluentinc/ksql-images) to build the ksqldb images
- [confluentinc/control-center-images](https://github.com/confluentinc/control-center-images) to build the control center image

