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

### Build optimized JRE (jlink)

```bash
# For convenience, cd into the build directory
cd srv/target
# Unpack the jar so we can analyze the Java modules needed by our app's dependencies
jar xf demoapp.jar
# Identify the Java modules needed by our app and it's dependencies
jdeps --ignore-missing-deps -q --recursive --multi-release 21 --print-module-deps --class-path 'BOOT-INF/lib/*' demoapp.jar > deps.info
# Build an optimized version of the SAPMachine JRE with just the dependencies our app needs
jlink --add-modules $(cat deps.info) --strip-debug --no-header-files --no-man-pages --output ./tinysapmachine
# Show that our smaller JRE can run our app
./tinysapmachine/bin/java -jar demoapp.jar
```

With this we slimmed down the ~150 MiB JRE to about ~86 MiB (values might be different on your system, depending on your operating system and which dependencies your application has).

### Build a bare container image

A bare container image is a container image with all parts of the operating system stripped out that you don't need.
To get started with building your bare container image, build an container image with the stripped JRE first like in this example `Containerfile`:

```Dockerfile
FROM ghcr.io/sap/sapmachine:21-jdk-headless-gl-1592 as build
RUN apt-get update -q && apt-get install -y binutils
RUN mkdir /usr/src/demoapp
COPY . /usr/src/demoapp
WORKDIR /usr/src/demoapp
COPY srv/target/demoapp.jar demoapp.jar
RUN jar xf demoapp.jar
RUN jdeps --ignore-missing-deps -q  \
    --recursive  \
    --multi-release 21  \
    --print-module-deps  \
    --class-path 'BOOT-INF/lib/*'  \
    demoapp.jar > deps.info
RUN jlink \
    --add-modules $(cat deps.info) \
    --strip-debug \
    --no-header-files \
    --no-man-pages \
    --output /tinysapmachine

FROM ghcr.io/gardenlinux/gardenlinux:1592

EXPOSE 8080

COPY --from=build /tinysapmachine /jre
COPY --from=build /usr/src/demoapp/demoapp.jar /

CMD ["/jre/bin/java", "-jar", "/demoapp.jar"]
```

The procedure to build your bare container image is as follows:

```bash
# Download the latest release of the unbase_oci shell script
wget https://github.com/gardenlinux/unbase_oci/releases/download/latest/unbase_oci
# make sure it is executable
chmod +x unbase_oci

# Pull the base image which is needed by unbase_oci to compute the diff
# This needs to be same image you use in the last 'FROM' statement in your Containerfile
SHA_GL=$(podman pull -q ghcr.io/gardenlinux/gardenlinux:1592)

# Build your app's container image and save the image sha sum
SHA_MYAPP=$(podman build -q -t myapp .)

# Let unbase_oci do its magic. You will get an image called myapp:latest_bare that is much smaller
./unbase_oci --ldd-dependencies podman:"$SHA_GL" podman:"$SHA_MYAPP" podman:myapp:latest_bare
```

Now if you run `podman images` you should be able to see both your original container image next to your bare image:

```
podman images
REPOSITORY        TAG          IMAGE ID      CREATED             SIZE
localhost/myapp   latest_bare  dfc66ec39aa9  6 seconds ago       149 MB
localhost/myapp   latest       ae50705013bf  About a minute ago  256 MB
```

As we can see, unbase oci cut our image size almost in half.

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
