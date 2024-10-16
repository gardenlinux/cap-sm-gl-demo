# CAP on SapMachine on Garden Linux Demo

## Demo goal
Highligh thow easy it is to run JAVA apps based on SAP stack
- **SAP Cloud Application Programming Model (CAP)** is a framework of languages, libraries, and tools for building enterprise-grade services and applications. (from https://cap.cloud.sap/docs/about/)
- **SapMachine** is a free, open-source, non-commercial, cross-platform, and fully-functional production-grade version of the Open Java Development Kit (OpenJDK). Built and supported by SAP, it provides a robust and reliable Java environment. (from https://sap.github.io/SapMachine/)
- **Garden Linux Container base image** is a slim base image maintained by SAP Garden Linux team and well integrated with SapMachine for Java use-cases.
- **Garden Linux OS as container runtime worker node orchestrated by Gardener**

## Demo Steps

1. Bootstrap locally a cap project
2. Package the app in SapMachine on Garden Linux container and demo that it works locally
3. Publish the container to a public image registry
4. Schedule the new app in a Gardener managed Kubernetes Cluster runing Garden Linux worker node (why not on Iron Core?)
5. Show the app works with kubectl port-forward


## Demo execution flow

### Local dev setup (before the demo)

Uses https://cap.cloud.sap/docs/java/getting-started

Expected state is that the cap prerequisites for Java local development are met and tooling is available
```bash
cds --version
java --version
mvn --version
```

### Demo init (app init)

```bash
cds init demoapp --add java
mvn com.sap.cds:cds-maven-plugin:add -Dfeature=TINY_SAMPLE
```

### Demo run (app local run)
``` bash
mvn spring-boot:run
curl http://localhost:8080/odata/v4/CatalogService/
```

### Package (app build)

```bash
mvn clean package spring-boot:repackage
mvn package spring-boot:repackage
java -jar ./srv/target/demoapp.jar
curl http://localhost:8080
curl http://localhost:8080/odata/v4/CatalogService/Books | jq
```

### Publishing the app

```bash
podman build -t gcr.io/demoapp .
podman run --rm -ti -p 8080:8080 gcr.io/demoapp 
podman push gcr.io/demoapp
```

### Scheduling the app

```bash
gardenctl ...
kubectl run gcr.io/demoapp
kubectl port-forward demoapp
curl http://localhost:8080
curl http://localhost:8080/odata/v4/CatalogService/Books | jq
```