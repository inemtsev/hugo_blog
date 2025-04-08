---
title: "Tinkering With Ktor 3: Practical uses of Ktor's plugin system"
date: 2024-05-11T12:49:31+07:00
draft: false
tags: ["kotlin","ktor"]
sitemap: 
    priority: 0.5
---

### Summary

Ktor isn't just a small and snappy framework, it features a simple, yet effective plugin system to extend the framework to your heart's content. My team uses this feature extensively, and I've used it in several of my side projects. Let's play with a few example and see what we can do.

### Simple example

Let's say, we want to see how long does it take for server to process a request until it responds. We can create a new plugin in `plugins/MeasureTimeToResponse.kt` with the following code:

```kotlin
val MeasureTimeToResponse = createApplicationPlugin(name = "MeasureTransformationBenchmarkPlugin") {
    val timeToResponseKey = AttributeKey<Long>("timeToResponseKey")

    onCall { call ->
        val callTime = System.currentTimeMillis()
        call.attributes.put(timeToResponseKey, callTime)
    }

    onCallRespond { call ->
        val callTime = call.attributes[timeToResponseKey]
        val responseTime = System.currentTimeMillis()

        println("Time to response: ${responseTime - callTime} ms") // or use some cool logger
    }
}
```

Finally, we enable this plugin by installing it in `Application.kt`

```kotlin
fun Application.module() {
    configureHTTP()
    configureSerialization()
    configureRouting()
    install(MeasureTimeToResponse) // yay
}
```

Voila! We now get this in our console for every request: `Time to response: 8 ms`

Ktor provides several hooks like: *onCall*, *onCallRespond* and others. We can even tap into application events like *ApplicationStarted* or *ApplicationStopped*. [See full list here.](https://ktor.io/docs/server-custom-plugins.html#handle-app-events)

### Going deeper

Imagine you have a very complicated http pipeline with various transformations. It may become difficult to imagine what the *final* response will look like. Well, we can log the final response after all the steps using the *ResponseBodyReadyForSend* hook:

```kotlin
val FinalResponseLogger = createApplicationPlugin(name = "FinalResponseLogger") {
    on(ResponseBodyReadyForSend) { call, finalResponse ->
        val responseContent = finalResponse as? TextContent
        responseContent?.let { println("Final response: ${responseContent.text}") }
    }
}
```

You might notice that we cast to TextContent here, the reason for this is that finalResponse is of type **OutgoingContent** here, which does not allow us to read the body (if you try to use toString, you will simply get the first few characters). However, we can cast this to **TextContent** for valid responses and this will give us access we need.

Install the plugin and test!
Now, we are logging our whole page in the response :)

<p>
<img src="https://eventslooped-images-thumb.s3.ca-central-1.amazonaws.com/resized-l-ktor-log-response.png" alt="ktor-final-response"/>
</p>

As always, [here is the github to final code.](https://github.com/inemtsev/ktor-tinker)
