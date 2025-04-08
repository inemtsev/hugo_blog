---
title: "Kotlin Dsl Getting Started"
date: 2022-09-04T14:23:25+07:00
draft: true
varTitle: "kotlin-dsl-getting-started"
varResizer: "https://aqzodowgen.cloudimg.io/fit/800x600/n/https://www.eventslooped.com/posts/img/"
tags: ["kotlin", "dsl"]
sitemap: 
    priority: 0.5
---

### Return to Blogging

After taking a break during the Covid years I decided to resume blogging. During the last few years I've had the pleasure of doing some work on Android, and I've much enjoyed coding in Kotlin. I've decided to try using Kotlin on the server-side for some smaller projects and found that the experience was quite positive. 

One of the more interesting features of Kotlin is the ability to create your own DSL (domain-specific-language). There are many use cases for this feature; a common one is to write builders in a readable, declarative way. In this post will build a small DSL to represent a response from the server. Kotlin, Ktor and Kotlin serializer will be used.

### The Goal

Suppose you would like your endpoint to return the following structure to represent a list of meetings in your day
```json
{
  "year": 15,
  "month": 12,
  "day": 2022,
  "isOutOfOffice": false,
  "events": [
    {
      "name": "Standup",
      "timeStart": 9,
      "timeEnd": 10,
      "people": []
    },
    {
      "name": "1-1",
      "timeStart": 11,
      "timeEnd": 12,
      "people": [
        {
          "name": "John Brown",
          "isOrganizer": true
        },
        {
          "name": "Chris Grossman",
          "isOrganizer": false
        },
        {
          "name": "Sally Wallace",
          "isOrganizer": false
        }
      ]
    }
  ]
}
```

The classic way would be to build up the complex response object in one of your services or mappers. But, there may be a better way. Let's try to obtain this structure by building up a DSL.

Without much effort we can create a small DSL that will allow us to write the following very succinctly:
```kotlin
val dayInTheOffice = day(2022, 12, 15, false) {
                event("Standup", 9, 10)
                event("1-1", 11, 12) {
                    person("John Brown", true)
                    person("Chris Grossman", false)
                    person("Sally Wallace", false)
                }
            }
```

There is nothing stopping you from doing logic, loops, etc - inside your DSL:
```kotlin
val dayInTheOffice = day(2022, 12, 15, false) {
                eventTimes.forEach { e ->
                    val people = eventPeople[e.nameOfEvent]
                    event(
                        meetingName = e.nameOfEvent,
                        startHour = e.startHour,
                        endHour = e.endHour
                    ) {
                        people?.let {
                            people.forEach { p -> person(name = p.name, isOrganizer = p.isOrganizer) }
                        }
                    }
                }
            }
```