---
title: "Tinkering With Ktor: Setting up a simple server with Docker"
date: 2024-01-07T13:24:35+07:00
draft: false
tags: ["kotlin","ktor"]
sitemap: 
    priority: 0.5
---

### Summary

After doing a few small projects on Android, I've come to enjoy Kotlin's convenient syntax that strikes a beautiful balance between functional and imperative programming. After doing some research, it looks like Kotlin has been gaining popularity on the server-side as well and for good reasons, it's so darn productive. I'll be doing a small series of posts on how to use Ktor framework, for my own learning and hopefully for yours as well :)

### Getting started

I'll assume the reader already has JVM 17 or higher installed (my machine has 17.0.7 installed) as well as Docker.

JetBrains IntelliJ *Ultimate* has a template for getting started with Ktor, but let's use the free option and start with the [Ktor project generator here.](https://start.ktor.io/)

Head over to the project generator and:
1. Fill in your project details (name, package, etc.)
2. Add these essential plugins:
   - Routing: for handling HTTP requests
   - kotlinx.serialization: for JSON serialization/deserialization

The generator will create a zip file with everything you need to get started.

### Quick test

Open the extracted folder with IntelliJ, open Application.Kt file and run `fun main()`
If your environment is setup correctly, you should get this console result:

<p>
{{< figure src="resized-l-ktor-running.png" alt="Ktor running" preset="medium" loading="lazy" >}}
</p>


### Set up our Dockerfile
At the root of your new project, create a file called Dockerfile (no extension).

Let's use a multi-stage build, in the first stage we build our app and in the second stage we package the runtime with our finalized application.

For the build stage we can use the [Gradle base image with JDK 17.](https://hub.docker.com/_/gradle/tags?page=1&name=17) Then we build a FatJar (Jar with our app's dependencies included in the package) of our application using Gradle. Our first stage then looks like this:

```dockerfile
FROM gradle:jdk17 AS build
COPY --chown=gradle:gradle . /home/gradle/src
WORKDIR /home/gradle/src
RUN gradle buildFatJar --no-daemon
```

Note that we named this stage **build**. We will need this later.

The second stage will include the runtime and our packaged app from the first stage. I found a linux with [JRE image here](https://hub.docker.com/_/eclipse-temurin/tags?page=1&name=17) and used that as my base image for this stage. In most cases you need the JRE (Java Runtime Environment) here, not JDK (Java Development Kit). You can see the details about the image on the website. The one I am using for this example is based on the Ubuntu distribution of Linux, a very common setup.

We copy the FatJar from the previous stage by referring to its name "build", expose the port 8080 and set the start of the app. Giving us the resulting second stage:

```dockerfile
FROM eclipse-temurin:17.0.10_7-jre
COPY --from=build home/gradle/src/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Finally, run `docker build -t cool-app .`, this builds your container with the tag cool-app. Use `docker run -p 8080:8080 cool-app` to run container.

Voila, your app can run on any system with just docker installed.

By default you can use http://0.0.0.0:8080 to test.

<p>
{{< figure src="resized-l-tinkering-containers.jpg" alt="containers" preset="medium" loading="lazy" >}}
</p>

You can see the final result in this [repository.](https://github.com/inemtsev/ktor-tinker)