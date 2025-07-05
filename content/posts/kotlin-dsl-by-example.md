---
title: "Kotlin Dsl by Example"
date: 2025-06-08T19:44:33+07:00
draft: true
---

### Summary

One of the more interesting features of Kotlin is the ability to create your own DSL (domain-specific-language). There are many use cases for this:
- **UI elements:** [Jetpack compose](https://developer.android.com/compose), [kotlinx.html](https://github.com/Kotlin/kotlinx.html)...
- **Configurations:** [Ktor configuration](https://ktor.io/docs/server-configuration-code.html#embedded-custom)
- **Builder pattern:** In fact, anywhere we can use the builder pattern, we can build a DSL around it. This way we can provide a clear and expressive syntax while tucking away validation/transformation/etc logic.

### An Example

Let's say we wanted to build a structure to represent our CI pipeline, because maybe we want to generate our CI pipeline configuration dynamically. We also want to use a builder and do some validation on the values that the user can provide for this structure.

Let's start with the root of our DSL structure:

```kotlin
// Main entry point
fun pipeline(init: Pipeline.() -> Unit): Pipeline {
    val pipeline = Pipeline()
    pipeline.init()
    return pipeline
}
```

This function will already allow us to use `pipeline { }` syntax. Let's break it down, the **init** parameter of our function
is actually called a [function type with receiver](https://kotlinlang.org/docs/lambdas.html#function-literals-with-receiver), in other words it's a *lambda* that can be called in the context of a *specific type instance*, in this case Pipeline type.
DSL definitions can become verbose, so let's use Kotlin's [apply](https://kotlinlang.org/api/core/kotlin-stdlib/kotlin/apply.html) and single expression syntax to make it more consise:

```kotlin
fun pipeline(init: Pipeline.() -> Unit) = Pipeline().apply(init)
```

In Kotlin, lambda parameters of a function can be brough outside of a paranthesis (see: [Trailing lambda](https://kotlinlang.org/docs/lambdas.html#passing-trailing-lambdas)), like so:

```kotlin
val product = items.fold(1) { acc, e -> acc * e }
// this is the same as writing
val product = items.fold(1, { acc, e -> acc * e })
```

This fact combined with our pipeline function already allows us to use the syntax:

```kotlin
val myProjectPipeline = pipeline { // pipeline function creates a Pipeline instance internally and runs this lambda on it
    // what can we do here?
}
```

### Let's dig deeper

Armed with these patterns, let's build a more complete example. Our CI Pipeline, probably needs some stages and some triggers, so we can start with this structure:

```kotlin
class Pipeline {
    val stages = mutableListOf<Stage>()
    val triggers = mutableListOf<Trigger>()

    fun stages(init: StagesBuilder.() -> Unit) {
        val builder = StagesBuilder().apply(init) // we can add custom logic inside of our builders
        stages.addAll(builder.stages)
    }

    fun triggers(init: TriggersBuilder.() -> Unit) {
        val builder = TriggersBuilder().apply(init) // we can add custom logic inside of our builders
        triggers.addAll(builder.triggers)
    }
}
```

We are now capable of using the following DSL syntax:

```kotlin
pipeline {
    stages {
        // populate stages here
    }
    triggers {
        // populate triggers here
    }
}
```

### Complete triggers

Our pipeline needs to have triggers like OnPush, OnPullRequest, OnSchedule and OnTag. Let's add this structure to represent these:

```kotlin
sealed class Trigger {
    data class OnPush(val branch: String) : Trigger()
    data class OnPullRequest(val targetBranch: String) : Trigger()
    data class OnSchedule(val cron: String) : Trigger()
    data class OnTag(val pattern: String) : Trigger()
}
```

Finally, we need to provide a Builder:

```kotlin
class TriggersBuilder {
    val triggers = mutableListOf<Trigger>()

    fun onPullRequest(targetBranch: String = "main") {
        triggers.add(Trigger.OnPullRequest(targetBranch))
    }

    fun onPush(branch: String) {
        triggers.add(Trigger.OnPush(branch))
    }

    fun onSchedule(cron: String) {
        if (!cron.matches(Regex("""^(\*|[0-5]?\d)(\s+(\*|[01]?\d|2[0-3])){4}$"""))) { // add some validation here
            throw IllegalArgumentException("Invalid cron expression: $cron")
        }
        triggers.add(Trigger.OnSchedule(cron))
    }

    fun onTag(pattern: String = "*") {
        triggers.add(Trigger.OnTag(pattern))
    }
}
```

### Complete stages (Advanced example)

Our pipeline stage, needs a name, jobs to run, environment to target and prerequisite step(s). We can model these via:

```kotlin
data class Stage (
    val name: String,
    val jobs: List<Job>,
    val environment: String? = null,
    val requires: List<String> = emptyList()
)
```

Since, we can have multiple stages, let's create a **StagesBuilder** first (note the plural):

```kotlin
class StagesBuilder {
    val stages = mutableListOf<Stage>()

    fun stage(name: String, init: StageBuilder.() -> Unit) { // expose a lamba to be applied on StageBuilder
        if(name.isBlank()) throw IllegalArgumentException("Stage name must not be blank")

        val builder = StageBuilder(name)
        builder.init() // apply the passed lambda onto the builder
        stages.add(builder.build()) // add result of builder into our internal collection
    }
}
```

This simple class/function validates the name of the stage and exposes a lambda to build the contents of that specific stage.

We want to run as many **Jobs** per stage as we want and even parallelize them. Finally, it would be great to indicate that a stage depends on another stage to complete. One way to model this using **StageBuilder** (note the singular):

```kotlin
class StageBuilder(private val name: String) {
    private val jobs = mutableListOf<Job>() // higher type can accept all job types
    private val requires = mutableListOf<String>()
    var environment: String? = null

    fun requires(stageName: String) requires.add(stageName)

    fun parallel(init: ParallelJobBuilder.() -> Unit) {
        val builder = ParallelJobBuilder()
        builder.init()
        jobs.add(Job.ParallelJob(name, builder.jobs, environment, requires))
    }

    fun deploy(init: DeployJobBuilder.() -> Unit) {
        val builder = DeployJobBuilder()
        builder.init()
        jobs.add(builder.build())
    }

    fun job(name: String, init: JobBuilder.() -> Unit) {
        val builder = JobBuilder()
        builder.init()
        jobs.add(builder.build())
    }

    fun build(): Stage {
        return Stage(
            name,
            jobs,
            environment,
            requires
        )
    }
}

```

Let's see what this allows us to create:

```kotlin
    val myProjectPipeline = pipeline {
        triggers {
            onPullRequest("main")
            onPush("develop")
            onSchedule("0 0 * * *") // Daily at midnight
            onTag("v*")
        }

        stages {
            stage("Build") {
                job("compile") {
                    run("./gradlew build")
                    run("println('Build complete')")
                }
            }
            stage("Test") {
                parallel {
                    job("unit-tests") {
                        run("./gradlew test")
                        artifact("test-results.xml")
                    }
                    job("integration-tests") {
                        run("./gradlew integrationTest")
                        artifact("integration-test-results.xml")
                    }
                }
            }
            stage("Deploy") {
                requires("Build")
                deploy {
                    to("PROD")
                    healthCheck("http://localhost:8080/health")
                    replicas = 2
                }
            }
        }
    }

    val pipelineJson = Json { prettyPrint = true }.encodeToString(myProjectPipeline)
    println(pipelineJson)

```


