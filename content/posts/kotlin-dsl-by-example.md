---
title: "Kotlin DSL by Example"
date: 2025-06-05T15:30:00+07:00
draft: false
tags: ["kotlin","dsl", "functional"]
sitemap: 
    priority: 0.5
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
        // add some validation here
        if (!cron.matches(Regex("""^(\*|[0-5]?\d)(\s+(\*|[01]?\d|2[0-3])){4}$"""))) { 
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

Let's create a few types of jobs that can run in our pipeline: Basic, Parallel and Deployment. These can be modelled with:

```kotlin
sealed class Job {
    abstract val name: String

    data class BasicJob(
        override val name: String,
        val commands: List<String>,
        val artifacts: List<String> = emptyList(),
        val publishedResults: String? = null
    ): Job()

    data class ParallelJob(
        override val name: String,
        val jobs: List<Job>,
        val environment: String? = null,
        val requires: List<String> = emptyList(),
        val condition: String? = null
    ): Job()

    data class DeployJob(
        override val name: String,
        val target: String,
        val replicas: Int = 1,
        val healthCheck: String? = null,
        val rollbackOnFailure: Boolean = true
    ): Job()
}
```

These also needs some builders, you can customize them to your business' needs:

```kotlin
    class JobBuilder(private val name: String) {
        private val commands = mutableListOf<String>()
        private val artifacts = mutableListOf<String>()
        var publishedResults: String? = null

        fun run(command: String) {
            commands.add(command)
        }

        fun artifact(artifact: String) {
            artifacts.add(artifact)
        }

        fun build(): Job.BasicJob {
            return Job.BasicJob(
                name,
                commands,
                artifacts,
                publishedResults
            )
        }
    }

    class ParallelJobBuilder {
        // base type collection that can hold all types of jobs
        val jobs = mutableListOf<Job>()

        fun job(name: String, init: JobBuilder.() -> Unit) {
            val builder = JobBuilder(name)
            builder.init()
            jobs.add(builder.build())
        }
    }

    class DeployJobBuilder {
        var target = ""
        var replicas = 1
        var healthCheck: String? = null
        var rollbackOnFailure = true

        fun to(target: String) {
            require(target.contains("PROD") || target.contains("QA") || target.contains("UAT")) {
                "Target must be either 'PROD' or 'QA' or 'UAT"
            }
            this.target = target
        }

        fun replicas(count: Int) {
            require(count > 0) { "Replicas count must be greater than 0" }
            this.replicas = count
        }

        fun healthCheck(endpoint: String) {
            require(endpoint.contains("http://") || endpoint.contains("https://")) {
                "Health check endpoint must start with http:// or https://"
            }
            this.healthCheck = endpoint
        }

        fun build(): Job.DeployJob {
            return Job.DeployJob(
                "Deploy to $target",
                target,
                replicas,
                healthCheck,
                rollbackOnFailure
            )
        }
    }
```

Now that we have the job types, let's compose the stages. Since we can have multiple stages, let's create a **StagesBuilder** first (note the plural):

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
    private val jobs = mutableListOf<Job>() // base type can accept all job types
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

    // Adding kotlinx serializer and annotations to our project allows for
    val pipelineJson = Json { prettyPrint = true }.encodeToString(myProjectPipeline)
    println(pipelineJson)

```

Running the above example gives:

```json
{
    "stages": [
        {
            "name": "Build",
            "jobs": [
                {
                    "type": "org.example.pipeline.Job.BasicJob",
                    "name": "compile",
                    "commands": [
                        "./gradlew build",
                        "println('Build complete')"
                    ]
                }
            ]
        },
        {
            "name": "Test",
            "jobs": [
                {
                    "type": "org.example.pipeline.Job.ParallelJob",
                    "name": "Test",
                    "jobs": [
                        {
                            "type": "org.example.pipeline.Job.BasicJob",
                            "name": "unit-tests",
                            "commands": [
                                "./gradlew test"
                            ],
                            "artifacts": [
                                "test-results.xml"
                            ]
                        },
                        {
                            "type": "org.example.pipeline.Job.BasicJob",
                            "name": "integration-tests",
                            "commands": [
                                "./gradlew integrationTest"
                            ],
                            "artifacts": [
                                "integration-test-results.xml"
                            ]
                        }
                    ]
                }
            ]
        },
        {
            "name": "Deploy",
            "jobs": [
                {
                    "type": "org.example.pipeline.Job.DeployJob",
                    "name": "Deploy to PROD",
                    "target": "PROD",
                    "replicas": 2,
                    "healthCheck": "http://localhost:8080/health"
                }
            ],
            "requires": [
                "Build"
            ]
        }
    ],
    "triggers": [
        {
            "type": "org.example.pipeline.Trigger.OnPullRequest",
            "targetBranch": "main"
        },
        {
            "type": "org.example.pipeline.Trigger.OnPush",
            "branch": "develop"
        },
        {
            "type": "org.example.pipeline.Trigger.OnSchedule",
            "cron": "0 0 * * *"
        },
        {
            "type": "org.example.pipeline.Trigger.OnTag",
            "pattern": "v*"
        }
    ]
}
```

Let's test validation, updating the deploy's environment parameter to "Grandma's computer in the basement" gives us an error at runtime:

```kotlin
    deploy {
        to("Grandma's computer in the basement")
        healthCheck("http://localhost:8080/health")
        replicas = 2
    }
```

**Error:** *Exception in thread "main" java.lang.IllegalArgumentException: Target must be either 'PROD' or 'QA' or 'UAT*

Similarly, messing with our cron notation will give us an error:

```kotlin
    triggers {
        onPullRequest("main")
        onPush("develop")
        onSchedule("0 0") // Daily at midnight
        onTag("v*")
    }
```
**Error:** *Exception in thread "main" java.lang.IllegalArgumentException: Invalid cron expression: 0 0*

Full, runnable code [available here](https://github.com/inemtsev/pipeline-demo)
