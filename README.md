# docker-compose-gradle-plugin
[![Build Status](https://travis-ci.org/avast/docker-compose-gradle-plugin.svg?branch=master)](https://travis-ci.org/avast/docker-compose-gradle-plugin) [![Download](https://api.bintray.com/packages/avast/maven/docker-compose-gradle-plugin/images/download.svg) ](https://bintray.com/avast/maven/docker-compose-gradle-plugin/_latestVersion)

Simplifies usage of [Docker Compose](https://www.docker.com/docker-compose) for local development and integration testing in [Gradle](https://gradle.org/) environment.

`composeUp` task starts the application and waits till all exposed TCP ports are open (so till the application is ready). Then it reads assigned host and ports of particular containers and stores them into `dockerCompose.servicesInfos` property.

`composeDown` task stops the application and removes the containers.

## Why to use Docker Compose?
1. I want to be able to run my application on my computer, and it must work for my colleagues as well. Just execute `docker-compose up` and I'm done.
2. I want to be able to test my application on my computer - I don't wanna wait till my application is deployed into dev/testing environment and acceptance/end2end tests get executed. I want to execute these tests on my computer - it means execute `docker-compose up` before these tests.

## Why this plugin?
You could easily ensure that `docker-compose up` is called before your tests but there are few gotchas that this plugin solves:

1. If you execute `docker-compose up -d` (_detached_) then this command returns immediately and your application is probably not able to serve requests at this time. This plugin waits till all exported TCP ports are open.
2. It's recommended not to assign fixed values of exposed ports in `docker-compose.yml` (i.e. `8888:80`) because it can cause ports collision on integration servers. If you don't assign a fixed value for exposed port (use just `80`) then the port is exposed to random free port. This plugin reads assigned ports (and even IP addresses of containers) and stores them into `dockerCompose.servicesInfo` map.

# Usage
The plugin must be applied on project that contains `docker-compose.yml` file. It supposes that [Docker Engine](https://www.docker.com/docker-engine) and [Docker Compose](https://www.docker.com/docker-compose) are installed and available in `PATH`.

```gradle
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath "com.avast.gradle:docker-compose-gradle-plugin:$versionHere"
    }
}

apply plugin: 'java'
apply plugin: 'docker-compose'

dockerCompose.isRequiredBy(test) // hooks 'dependsOn composeUp' and 'finalizedBy composeDown'

dockerCompose {
    // stopContainers = false // useful for debugging
}

test.doFirst {
    // get information about container of service 'web' (declared in docker-compose.yml)
    def webInfo = dockerCompose.servicesInfos.web
    // pass host and exposed TCP port 80 as Java System properties
    systemProperty 'myweb.host', webInfo.host
    systemProperty 'myweb.port', webInfo.ports[80]
    // exposes "WEB_HOST" and "WEB_TCP_80" environment variables
    dockerCompose.exposeAsEnvironment(test)
    // exposes "web.host" and "web.host.80" system properties
    dockerCompose.exposeAsSystemProperties(test)
}
```

# Tips 
* `dockerCompose.servicesInfos` contains information about running containers so you must access this property after `composeUp` task is finished. So `doFirst` of your test task is perfect place where to access it.
* Check [ServiceInfo.groovy](/src/main/groovy/com/avast/gradle/dockercompose/ServiceInfo.groovy) to see what you can know about running containers.
* If some Dockerfile needs an artifact generated by Gradle then you can declare this dependency in a standard way, like `composeUp.dependsOn project(':my-app').distTar`
* You can call `dockerCompose.isRequiredBy(anyTask)` for any task, for example for your custom `integrationTest` task.
* All properties in `dockerCompose` have meaningfull default values so you don't have to touch it. If you are interested then you can look at [ComposeExtension.groovy](/src/main/groovy/com/avast/gradle/dockercompose/ComposeExtension.groovy) for reference.
