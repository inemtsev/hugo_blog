---
title: "Golang Microservices Part 2 - PostgreSQL with gorm"
date: 2019-12-29T23:57:36+07:00
draft: false
tags: ["golang","microservices", "ORM"]
sitemap: 
    priority: 0.5
---

### Setting up GORM

[Continuing from the app we created in Part 1](/posts/golang-microservices-grpc-communication)

PostgreSQL is one of the most popular database choices today because it is free and has most of the features that the big guns like Microsoft SQL Server and Oracle offer. To connect our microservice to PostgreSQL, we can simplify our life by using an ORM for GO called [GORM](https://gorm.io/)... GO-ORM get it? 

<iframe src="https://giphy.com/embed/fGRLn3m6XvmHeLNOzn" width="480" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/HallmarkChannel-fGRLn3m6XvmHeLNOzn">via GIPHY</a></p>

<p>Let's upgrade the gRPC microservice we created in Part 1 to use an actual database instead of just cache. </p>

### First, let's create our persistence model

We will need to install the **github.com/jinzhu/gorm** package. So run:

> go get github.com/jinzhu/gorm

The **BasketballPlayer** model contains all of the data *we* want to persist. For things like ID and other technicallities we can embed another type, let's call it **BasePersistenceModel**. You are free to use the default gorm.Model, but you will get several columns you may or may not need like *updated_at*, *created_at* and *deleted_at*. You will also not be able to change the type of the column ID. This is why I like to create my own base model. 


{{< highlight go >}}
type BasePersistenceModel struct {
	ID	uint64 `gorm:"type:int;primary_key"`
}

type BasketballPlayer struct {
	BasePersistenceModel //gorm.Model can be used if you don't mind ID being an integer
	FirstName string
	LastName string
	Age int32
	PhotoUrl string
	PointsPerGame int32
	AssistsPerGame int32
	ReboundsPerGame int32
}
{{< / highlight >}}

You may ask, why **int32** is used here instead of the default **int**? Simply, because our generated gRPC model uses int32 and I did not feel like doing any conversion. [What is the difference?](https://stackoverflow.com/questions/21491488/what-is-the-difference-between-int-and-int64-in-go)

We have a small problem here, our *gRPC model* is not the same as our *persistence model*. Therefore, we will need to be able to convert between these two. Let's add two methods to help us do this:

{{< highlight go >}}
func (em *BasketballPlayer) GetgRPCModel() basketBallPlayer.Player {
	return basketBallPlayer.Player{
		Id:                   strconv.FormatUint(em.ID, 10),
		FirstName:            em.FirstName,
		LastName:             em.LastName,
		Age:                  em.Age,
		PhotoUrl:             em.PhotoUrl,
		PointsPerGame:        em.PointsPerGame,
		AssistsPerGame:       em.AssistsPerGame,
		ReboundsPerGame:      em.ReboundsPerGame,
	}
}

func (em *BasketballPlayer) From(gRPCModel basketBallPlayer.Player) {
		u, e := strconv.ParseUint(gRPCModel.Id, 10, 64)
		if e != nil {
			panic("incorrect ID from gRPC")
		}

		em.ID = u
		em.FirstName = gRPCModel.FirstName
		em.LastName = gRPCModel.LastName
		em.Age = gRPCModel.Age
		em.PhotoUrl = gRPCModel.PhotoUrl
		em.PointsPerGame = gRPCModel.PointsPerGame
		em.AssistsPerGame = gRPCModel.AssistsPerGame
		em.ReboundsPerGame = gRPCModel.ReboundsPerGame
}
{{< / highlight >}}

Putting this all together, we have BasketballPlayer.go below, which we can put in our **models folder**:

```go
package models

import (
	basketBallPlayer "basic-gRPC-proto"
	"strconv"
)

type BasePersistenceModel struct {
	ID	uint64 `gorm:"type:int;primary_key"`
}

type BasketballPlayer struct {
	BasePersistenceModel //gorm.Model can be used if you don't mind ID being an integer
	FirstName string
	LastName string
	Age int32
	PhotoUrl string
	PointsPerGame int32
	AssistsPerGame int32
	ReboundsPerGame int32
}

func (em *BasketballPlayer) GetgRPCModel() basketBallPlayer.Player {
	return basketBallPlayer.Player{
		Id:                   strconv.FormatUint(em.ID, 10),
		FirstName:            em.FirstName,
		LastName:             em.LastName,
		Age:                  em.Age,
		PhotoUrl:             em.PhotoUrl,
		PointsPerGame:        em.PointsPerGame,
		AssistsPerGame:       em.AssistsPerGame,
		ReboundsPerGame:      em.ReboundsPerGame,
	}
}

func (em *BasketballPlayer) From(gRPCModel basketBallPlayer.Player) {
	u, e := strconv.ParseUint(gRPCModel.Id, 10, 64)
	if e != nil {
		panic("incorrect ID from gRPC")
	}

	em.ID = u
	em.FirstName = gRPCModel.FirstName
	em.LastName = gRPCModel.LastName
	em.Age = gRPCModel.Age
	em.PhotoUrl = gRPCModel.PhotoUrl
	em.PointsPerGame = gRPCModel.PointsPerGame
	em.AssistsPerGame = gRPCModel.AssistsPerGame
	em.ReboundsPerGame = gRPCModel.ReboundsPerGame
}
```

### Finally, let's update our main.go

**First**, we need to add proper package references to **main.go**. Let's add the following: 

```go
"github.com/jinzhu/gorm"
_ "github.com/jinzhu/gorm/dialects/postgres"
```

The second line adds support for the database of your choosing. Don't forget to add it, or you will get a cryptic error that isn't easy to understand. [Here you will find a list of supported databases by gorm.](http://gorm.io/docs/connecting_to_the_database.html) 

**Second**, we need to add add a few package-level variables and a way to initialize them:

```go
var db *gorm.DB
const dbPath = "host=34.84.26.219 user=postgres password=somecoolpasswordhere"

func init() {
	var e error
	db, e = gorm.Open("postgres", dbPath)
	defer db.Close()

	if e != nil {
		panic("failed to connect to database")
	}

	db.AutoMigrate(&models.BasketballPlayer{})  //this will create a table in your database if it isn't there already
}
```

**Lastly**, let's update our handler so that it checks cache for data first and if not found, tries to fetch data from database:

```go
func (*server) GetBasketballPlayer(ctx context.Context, r *basketBallPlayer.PlayerRequest) (*basketBallPlayer.PlayerResponse, error) {
	id := r.GetId()

	fmt.Println("Checking cache...")
	playerVal, found := c.Get(id)
	if found {
		player := playerVal.(*basketBallPlayer.Player)
		return &basketBallPlayer.PlayerResponse{
			Result: player,
		}, nil
	} else {
		fmt.Println("Checking database...")
		db, e := gorm.Open("postgres", dbPath)
		defer db.Close()
		if e != nil {
			panic(fmt.Sprintf("failed to connect to database: %v", e))
		}

		var p models.BasketballPlayer
		if e = db.First(&p, id).Error; e != nil {
			return nil, fmt.Errorf("could not find player with id: %v", id)
		} else {
			gRPCResult := p.GetgRPCModel()
			c.Set(id, &gRPCResult, cache.DefaultExpiration)

			return &basketBallPlayer.PlayerResponse{
				Result:	&gRPCResult,
			}, nil
		}
	}
}
```

Update: I improved caching slightly from Part 1, the code now stores pointers in cache instead of copying full values. 

Putting all the pieces into our app from Part 1 we get:

```go
package main

import (
	basketBallPlayer "basic-gRPC-proto"
	"basic-gRPC-server/models"
	"context"
	"fmt"
	"github.com/jinzhu/gorm"
	_ "github.com/jinzhu/gorm/dialects/postgres"
	"log"
	"net"
	"time"

	"github.com/patrickmn/go-cache"
	"google.golang.org/grpc"
)

type server struct{}

var c *cache.Cache
var db *gorm.DB
const dbPath = "host=34.84.26.219 user=postgres password=somecoolpasswordhere"

func init() {
	var e error
	db, e = gorm.Open("postgres", dbPath)
	defer db.Close()

	if e != nil {
		panic("failed to connect to database")
	}

	db.AutoMigrate(&models.BasketballPlayer{})
}

func (*server) GetBasketballPlayer(ctx context.Context, r *basketBallPlayer.PlayerRequest) (*basketBallPlayer.PlayerResponse, error) {
	id := r.GetId()

	fmt.Println("Checking cache...")
	playerVal, found := c.Get(id)
	if found {
		player := playerVal.(*basketBallPlayer.Player)
		return &basketBallPlayer.PlayerResponse{
			Result: player,
		}, nil
	} else {
		fmt.Println("Checking database...")
		db, e := gorm.Open("postgres", dbPath)
		defer db.Close()
		if e != nil {
			panic(fmt.Sprintf("failed to connect to database: %v", e))
		}

		var p models.BasketballPlayer
		if e = db.First(&p, id).Error; e != nil {
			return nil, fmt.Errorf("could not find player with id: %v", id)
		} else {
			gRPCResult := p.GetgRPCModel()
			c.Set(id, &gRPCResult, cache.DefaultExpiration)

			return &basketBallPlayer.PlayerResponse{
				Result:	&gRPCResult,
			}, nil
		}
	}
}

func main() {
	fmt.Println("Starting gRPC micro-service...")
	c = cache.New(60*time.Minute, 70*time.Minute)
	c.Set("1", &basketBallPlayer.Player{
		Id:              "1",
		FirstName:       "James",
		LastName:        "LeBron",
		Age:             33,
		PhotoUrl:        "https://ak-static.cms.nba.com/wp-content/uploads/headshots/nba/1610612747/2019/260x190/2544.png",
		PointsPerGame:   26,
		AssistsPerGame:  11,
		ReboundsPerGame: 8,
	}, cache.DefaultExpiration)

	l, e := net.Listen("tcp", ":50051")
	if e != nil {
		log.Fatalf("Failed to start listener %v", e)
	}

	s := grpc.NewServer()
	basketBallPlayer.RegisterPlayerServiceServer(s, &server{})

	if e := s.Serve(l); e != nil {
		log.Fatalf("failed to serve %v", e)
	}
}
```

I added one row into the database for testing purposes: 

Sending a request to our API for player with **Id** of **1**, gives us:
{{< figure src="result1.png" alt="cache only" loading="lazy" >}}

Sending another request for a player with **Id** of **2** (this one is from our database):
{{< figure src="result2.png" alt="cache and db only" loading="lazy" >}}

Now, let's take a look at our Gin API. I made a third request for player with Id of 2 to see if we get the expected speed-up from cache. And we do indeed:
{{< figure src="result3.png" alt="cache only" loading="lazy" >}}

Voila! It works as expected!
{{< figure src="JSON-result.png" alt="json result" loading="lazy" >}}

<p>Go Warriors! :)</p>
<iframe src="https://giphy.com/embed/fe6GhFBexWZeMgt5nV" width="480" height="270" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/nba-emotion-stephen-curry-fe6GhFBexWZeMgt5nV">via GIPHY</a></p>