---
title: "Tinkering With Ktor: Using HTML DSL for page templates"
date: 2024-03-30T19:02:33+07:00
draft: true
tags: ["kotlin","ktor","DSL"]
sitemap: 
    priority: 0.5
---

<p>
<img src="https://eventslooped-images-thumb.s3.ca-central-1.amazonaws.com/resized-l-tinkering-template.jpg" alt="templates">
Photo by <a href="https://unsplash.com/@pankajpatel?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Pankaj Patel</a> on <a href="https://unsplash.com/photos/computer-screen-displaying-website-home-page-Ylk5n_nd9dA?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash">Unsplash</a>
</p>

### Summary

There are a bunch of templating engines available for Ktor, each one with its own strengths and weaknesses. But, I don't like that I have to add yet another engine to the runtime (especially when I plan to run React or Vue anyway) so why not try using a simple feature of Kotlin to build our html? Kotlin DSL (Domain Specific Language) capabilities, while having a steep learning curve if you never used them before can be used structures that closely mimic html. This is not a new idea, there have been several attempts at this and the most popular one is the [kotlinx.html](https://github.com/Kotlin/kotlinx.html) library. Ktor team already created a convenient way to plug this library into your Ktor project, called `ktor-server-html-builder`. The kotlinx.html library gives us the basic components of HTML and we can extend that further if we want. Let's see what can be accomplished with these.

### Getting Started

Add this dependency to your `build.gradle.kts`:

{{< highlight kotlin >}}
implementation("io.ktor:ktor-server-html-builder:$ktor_version")
{{< / highlight >}}

Now, you can add this kind of route to render a basic html page, nothing fancy:

{{< highlight kotlin >}}
get("/index") {
            call.respondHtml(HttpStatusCode.OK) {
                attributes["lang"] = "en"
                head {
                    title { +"Let's tinker with Ktor" }
                    meta { charset = "UTF-8" }
                }
                body {
                    h1 { +"Hello World!" }
                    p { +"Let's tinker with Ktor" }
                }
            }
        }
{{< / highlight >}}

All the html "elements" here are simply functions that populate the outer block. The element "head" for example is a function and the body of this element is its lambda parameter. 

If we simplify the definition of head function, it can look something like this:

{{< highlight kotlin >}}
fun HTML.head(contents: HEAD.() -> Unit) {
    // create the instance which we will populate with lambda
    val head = HEAD()

    // contents lambda acts on HEAD class instance (as per definition), we can simply call it here
    head.contents()
    
    // where children is some mutable list in HTML
    this.children.add(head)
}
{{< / highlight >}}

The real code found in **kotlinx.html** is a bit more abstract, but this should give you some clues on how these DSLs are constructed.

If we now try to run our `fun main()` and hit our `/index` endpoint we will get a simple html page with this source:

{{< highlight html >}}
<!DOCTYPE html>
<html lang="en">
  <head>
    <title>Let's tinker with Ktor</title>
    <meta charset="UTF-8">
  </head>
  <body>
    <h1>Hello World!</h1>
    <p>Let's tinker with Ktor</p>
  </body>
</html>
{{< / highlight >}}

### Basic setup

We can already render a basic page, but what if we wanted to create a template out of it, so you can reuse some common code across different pages? Thankfully, the library already has a placeholder feature to help us with this.

Let's create another file **CompanyTemplate.kt** with the following basic template:

{{< highlight kotlin >}}

class CompanyTemplate : Template<HTML> {
    val titleText = Placeholder<TITLE>()
    val headScripts = Placeholder<HEAD>()
    val bodyContent = Placeholder<BODY>()
    val header = Placeholder<BODY>()
    val footer = Placeholder<BODY>()

    override fun HTML.apply() {
        attributes["lang"] = "en"
        head {
            link { rel = "stylesheet"; type = "text/css"; href = "/css/my-company-styles.css" }
            title { insert(titleText) }
            meta { charset = "UTF-8" }
            insert(headScripts)
        }
        body {
            insert(header)
            insert(bodyContent)
            insert(footer)
        }
    }
}

{{< / highlight >}}

Take note of the generic type of **Placeholder type**, it needs to match where that piece of code is going to be placed. For example, `Placeholder<BODY>` will need to be placed within the `body {}` function using the `insert()` function.

To make use of this template we have to use the call.respondHtmlTemplate() function in this way:

{{< highlight kotlin >}}

get("/index2") {
            call.respondHtmlTemplate(CompanyTemplate()) {
                titleText { +"Let's tinker with Ktor" }
                headScripts {
                    script(src = "/js/my-company-scripts.js") {}
                }
                header {
                    h1 { +"Welcome to our company" }
                }
                bodyContent {
                    p { +"Let's tinker with Ktor" }
                }
                footer {
                    p { +"© 2022 Our Company" }
                }
            }
        }

{{< / highlight >}}

You can see that the 5 sections populated here, match the 5 placeholders we defined in the CompanyTemplate class. We can take this a step further and hide away the code in the above snippet into a separate file and create a **ViewModel** to carry this data.

Create a ViewModel to represent requirements of the template:

{{< highlight kotlin >}}

data class CompanyTemplateViewModel(
    val titleText: String,
    val header: String,
    val bodyContent: String,
    val footer: String,
    val headScripts: List<String>
)

{{< / highlight >}}

Now, let's refactor our template populating code into a Provider. Since we are using functions to build these structures we can use loops and other logical operators freely:

{{< highlight kotlin >}}

object CompanyTemplateProvider {
    fun getTemplate(viewModel: CompanyTemplateViewModel): CompanyTemplate {
        val template =
            CompanyTemplate().apply {
                titleText { +viewModel.titleText }
                headScripts {
                    for (script in viewModel.headScripts)
                        script(src = script) {}
                }
                header {
                    h1 { +viewModel.header }
                }
                bodyContent {
                    p { +viewModel.bodyContent }
                }
                footer {
                    p { +viewModel.footer }
                }
            }

        return template
    }
}

{{< / highlight >}}

Finally, let's update our usage in route logic:

{{< highlight kotlin >}}

get("/index2") {
            val vm = CompanyTemplateViewModel(
                titleText = "Let's tinker with Ktor",
                header = "Welcome to our company",
                bodyContent = "Let's tinker with Ktor",
                footer = "© 2022 Our Company",
                headScripts = listOf("/js/my-company-scripts.js")
            )


            call.respondHtmlTemplate(CompanyTemplateProvider.getTemplate(vm)) {}
        }

{{< / highlight >}}

Voila! Now we can reuse this template neatly across various routes.

As always, [here is the github to final code.](https://github.com/inemtsev/ktor-tinker)