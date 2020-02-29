# Getting Started Building Docker Images with Gradle

This guide demonstrates how you can use Gradle and the [Gradle Docker plugin](https://github.com/bmuschko/gradle-docker-plugin) to define a Docker image, create a container based on that image, and run a containerized application locally.

We won't be creating a new Java application here: instead, we'll take a pre-made trivial Spring Boot application and containerize it.

## What you'll need
* About 20 (?) minutes
* A text editor or IDE
* A terminal application
* JDK 8 or later
* Docker Desktop for [Mac](https://docs.docker.com/docker-for-mac/install/) or [Windows](https://docs.docker.com/docker-for-windows/install/)

You do not need to install Gradle separately: the sample Spring Boot application that we'll use contains a Gradle wrapper that will discover and get a Gradle distribution automatically.

## Get a sample Java application
To get a sample Spring Boot application to containerize, [clone this repository](https://github.com/gorohoroh/tuneit-gradle-docker) from GitHub.

If cloning is not an option, [download the sample application](tuneit-gradle-docker-master.zip) in an archived form, and unpack the archive.

Open the sample application in your text editor or IDE of choice. At this point, the `build.gradle` file should look  like this:

```
plugins {
    id 'java'
    id 'application'
    id 'org.springframework.boot' version '2.2.5.RELEASE'
}

repositories {
    mavenCentral()
}

dependencies {
    compile 'org.springframework.boot:spring-boot-starter-web:2.2.5.RELEASE'
}

application {
    mainClassName = 'com.tuneit.App'
}

jar {
    manifest {
        attributes 'Main-Class': 'com.tuneit.App'
    }

    from {
        configurations.compile.collect { it.isDirectory() ? it : zipTree(it) }
    }
}
```
 
## Apply the Gradle Docker plugin
Add the `docker-spring-boot-application` (or remote-api?) plugin to the `plugins` section of your build script file: 

```
plugins {
    id 'java'
    id 'application'
    id 'org.springframework.boot' version '2.2.5.RELEASE'
    id 'com.bmuschko.docker-spring-boot-application' version '6.1.4'
}
```

## Import image and container types
As you will be using types from Gradle Docker plugin that we have just applied, add these import statements after the `plugins` section:

```
import com.bmuschko.gradle.docker.tasks.image.*
import com.bmuschko.gradle.docker.tasks.container.*
```

## Create a task to generate a Dockerfile
In order for Docker to know how to configure your image, it needs a file called `Dockerfile`. Let's create a Gradle task that generates the file:

```
task createDockerFile(type: Dockerfile) {
    from 'openjdk:8-jre-alpine'
    copyFile jar.archiveFileName.get(), '/app/test_service.jar'
    entryPoint 'java'
    defaultCommand '-jar', '/app/test_service.jar'
    exposePort 8080
}
```

When you execute this task, Gradle creates a `Dockerfile` under `build/docker`:

```
FROM openjdk:8-jre-alpine
COPY tuneit-gradle-docker-1.0.jar /app/test_service.jar
ENTRYPOINT ["java"]
CMD ["-jar", "/app/test_service.jar"]
EXPOSE 8080
```

TODO elaborate on these instructions.

## Copy jar to docker build dir
When we build our Docker image, it will need to contain a JAR file with our application and all its dependencies. To make this file available, create a task that copies the assembled JAR file into `build/docker`:

```
task syncJar(type: Copy) {
    dependsOn assemble
    from jar.destinationDirectory
    into "$buildDir/docker"
}

```

## Set Java compilation options to match base image specs
When we use our generated `Dockerfile` to build an image, it will be based on a distribution that uses JRE 8 as a runtime environment. In order to successfully run our application in this environment, let's make sure we generate Java classes compatible with Java 8:

```
compileJava {
    sourceCompatibility = JavaVersion.VERSION_1_8
    targetCompatibility = JavaVersion.VERSION_1_8
}
``` 

## Build image
We are now ready to build our Docker image. To do this, add the following task:
```
task buildImage(type: DockerBuildImage) {
    dependsOn createDockerFile, syncJar
    inputDir = createDockerFile.getDestDir()
    images = ["yourdockerhubusername/tuneit-gradle-docker:1.0"]
}
```

Make sure you have Docker running on your machine, and then execute the task:

```
❯ ./gradlew buildImage
```

This task will take outputs of the tasks we created earlier, and create a local Docker image. To verify that the image has been created, run the following Docker command:

```
❯ docker images
```

If all went well, you should see an entry describing the new image at the top of the list of available images:

![A Docker image has been created](img/terminal_docker_image_created.png)

## Create container
Now that we have an image, we can create an actual container based on that image.

First, let's define the name for our container as a local variable: we'll need to use it from more than a single task. Near the top of the build file, just before the `repositories` extension, add this line:

```
def ourContainerName = "ournewcontainer"
```

Add the following task:

```
task createContainer(type: DockerCreateContainer) {
    dependsOn buildImage
    targetImageId buildImage.getImageId()
    containerName = "ournewcontainer"
    hostConfig.portBindings = ['8080:8080']
}
```

Execute the task:

```
❯ ./gradlew createContainer
```

To verify that a container has been created, run the following command in the terminal:

```
❯ docker container ls --all
```

You should see an entry describing the new container at the top of the list of available images:

![A Docker container has been created](img/terminal_docker_container_created.png)

## Start container

If we start container and the container using the same name is already running, this won't work.

Because of that, we should:
* Create task `stopContainer`;
* Create task `removeContainer` that depends on `stopContainer`;
* Modify task `createContainer` to depend on `removeContainer`.

As a result, every time we want to start a container, Gradle will first make sure to:
* Check if a container using the same name is already running;
* If it is running, stop and remove the container.

```gradle
task stopContainer(type: DockerStopContainer) {
    targetContainerId("$ourContainerName")
}

task removeContainer(type: DockerRemoveContainer, group: dockerGroupName) {
    dependsOn stopContainer
    targetContainerId("$ourContainerName")
}

...

task startContainer(type: DockerStartContainer) {
    dependsOn createContainer
    targetContainerId("$ourContainerName")
}

```

Modify `createContainer` to depend on `removeContainer':

```gradle
task createContainer(type: DockerCreateContainer) {
- dependsOn buildImage
+ dependsOn buildImage, removeContainer
```

Move `createContainer` to be the penultimate task in `build.gradle`.

(Alternatively, merge "Create container" and "Start container" to suggest all tasks in one listing.)