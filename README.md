# CAP on SapMachine on Garden Linux Demo

Demo goal: Highligh thow easy it is to run JAVA apps based on SAP stack:
- **SAP Cloud Application Programming Model (CAP)** is a framework of languages, libraries, and tools for building enterprise-grade services and applications. (from https://cap.cloud.sap/docs/about/)
- **SapMachine** is a free, open-source, non-commercial, cross-platform, and fully-functional production-grade version of the Open Java Development Kit (OpenJDK). Built and supported by SAP, it provides a robust and reliable Java environment. (from https://sap.github.io/SapMachine/)
- **Garden Linux Container base image** is a slim base image maintained by SAP Garden Linux team and well integrated with SapMachine for Java use-cases.
- **Garden Linux OS as container runtime worker node orchestrated by Gardener**


## Setup 

Uses https://cap.cloud.sap/docs/java/getting-started

Expected state is that the cap prerequisites for Java local development are met and tooling is available
```bash
cds --version
java --version
mvn --version
```

## Demo init 

```bash
cds init demoapp --add java
mvn com.sap.cds:cds-maven-plugin:add -Dfeature=TINY_SAMPLE
```

## Demo run 
``` bash
mvn spring-boot:run
curl http://localhost:8080/odata/v4/CatalogService/
```

## Package 

```bash
mvn clean package spring-boot:repackage
mvn package spring-boot:repackage
java -jar ./srv/target/demoapp.jar
```

## Show from shell

```bash
curl http://localhost:8080
curl http://localhost:8080/odata/v4/CatalogService/Books | jq
```

## Build & run the image

```bash
podman build -t demoapp .
podman run --rm -ti -p 8080:8080 demoapp
```

As next steps of the demo:
1. tag & push the container to a registry
2. kubectl run the app in the cluster
3. kubectl port-forward the app and show it actually runs in the cluster - on garden linux VM managed by gardener in a GL container