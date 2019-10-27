---
title: "KrakenD: Writing Plugins using Golang"
date: 2019-10-19T18:59:02+07:00
draft: false
imgpad: 14
---

### Summary
[KrakenD](https://www.krakend.io) has caught my attention as a Gateway solution because of its claim at incredible performance and extensibility. It is Open Source and can be tailored to your needs or can be extended in a convenient way with its support for [Golang Plugins](https://golang.org/pkg/plugin/). 

### Plugin, what?
Go plugins are a fairly new addition to Golang (support starting with Go 1.8), but they provide a very innovative way to add functionality to a Go application. Go plugins are soft-linked, meaning that they are picked up by the main appication at runtime. Since plugins need to be part of the **main** package, their main() function is not run; they are initialized once and cannot be closed. Once small caveat of plugins at the time of writing, is that they cannot be compiled on a Windows OS; Linux and MacOS work just fine. For the following examples I will use Linux to show how to setup KrakenD and add a plugin. 

### Benefits of using Go Plugins with KrakenD
KrakenD has a small but very active and growing community; features are constantly being added and extended. If we wanted to add our own functionality to the pipeline and extend KrakenD, we could certainly do that. But, if a new version of KrakenD was to be released we would have to manually merge these changes into our own version of KrakenD. Wouldn't it be easier if we could simply drop a plugin file next to KrakenD binary and receive benefits upon app restart? This is the perfect case for Golang plugins. 

### How can you get started
I will not dive into how to modify the [KrakenD](https://www.krakend.io) core, it is open source after all and sky is the limit. Instead, I will clone/download the binary source from [KrakenD-CE Github](https://github.com/devopsfaith/krakend-ce). What is KrakenD-CE and how does it differ from KrakenD? KrakenD-CE is a Community Edition of KrakenD, it includes many additional middlewares for added functionality that creators thought were useful. KrakenD on its own is more barebones. **You can think of KrakenD-CE as KrakenD + Extra Stuff.** 

Assuming you have Go Binary installed and KrakenD folder located in your PATH environment location, compile in command-line via command:
> make build

If you have not installed Go Binaries, get them here: [https://golang.org/doc/install](https://golang.org/doc/install)
If you receive an error "make command not found" install build essentials using one of the following commands, depending on your distro.
> sudo yum install build-essential
> sudo apt-get install build-essential

### Configuring a basic setup
Once you have KrakenD built, let's take a look at the settings. These are contained in **krakend.json**. Most of the functionality you will need from a Gateway is configurable from this file. I suggest using the online [KrakenDesigner](https://www.krakend.io/docs/configuration/overview/) to generate your own configuration file. I created a basic configuration like this. Here I simply take the GitHub username as a parameter and call api.github.com as my downstream service to obtain a user's profile in json format. 

{{< gist inemtsev cc0ca70682d198abf00c0b1a8f246d57 "krakend.json" >}}

You can test your configuration via the following command with debugging:
> ./krakend run -c krakend.json --debug

### Creating the plugin

Let's create a sample plugin that makes a call to Github API gets another user's data and attaches the response as a header to the downstream service. 

{{< gist inemtsev cc0ca70682d198abf00c0b1a8f246d57 "headerModPlugin.go" >}}

Some things to notice here.
 - func main() is not necessary here because it will be ignored, but my IDE was complaining so it's there for decoration :)
 - attachuserid is the parameter we will pass in to the configuration in our krakend.json

Lastly, let's configure our plugin. Add the following lines to the root of your configuration: 

{{ < highlight json >}}
 "plugin": {
    "pattern": ".so",
    "folder": "./"
  },
  "extra_config": {
    "github_com/devopsfaith/krakend/transport/http/server/handler": {
      "name": "headerModPlugin",
      "attachuserid": "rsc"
   }
  }
{{ < / highlight >}}

Your final configuration should look something [like this.](https://gist.githubusercontent.com/inemtsev/cc0ca70682d198abf00c0b1a8f246d57/raw/b246c503d76bcb38d1c88a7d20ef0073f442bb8a/krakend_with_plugin.json)

Notice the **name** parameter that needs to match the one in our plugin and also notice the **attachuserid** parameter that we defined inside our plugin. I am passing in the username of **Russ Cox**, one of the big names behind Golang. 

If your configuration is correct, when you run KrakenD in debug mode you will see the message: injecting plugin headerModPlugin. :)

