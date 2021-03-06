service-pattern-go
-------

Hey! welcome, this is and example of simple clean architecture written in Go Lang with complete Dependency Injection along with Mocking example.

Inspired by [Manuel Kiessling go-cleanarchitecture](http://manuel.kiessling.net/2012/09/28/applying-the-clean-architecture-to-go-applications/) and [Joshua Partogi TDD training session](https://github.com/jpartogi/tennis-kata-laravel/)

It has simple dependencies:

 - [Chi (Router)](https://github.com/go-chi/chi)
 - [Jinzhu GORM (ORM)](https://github.com/jinzhu/gorm)
 - [Testify (Test & Mock framework)](https://github.com/stretchr/testify)

Get Started:

 - [Install](https://irahardianto.github.io/service-pattern-go/#install)
 - [Introduction](https://irahardianto.github.io/service-pattern-go/#introduction)
 - [Folder Structure](https://irahardianto.github.io/service-pattern-go/#folder-structure)
 - [Naming Convention](https://irahardianto.github.io/service-pattern-go/#naming-convention)
 - [Depency Injection](https://irahardianto.github.io/service-pattern-go/#dependency-injection)
 - [Mocking](https://irahardianto.github.io/service-pattern-go/#mocking)


----------

[Install](https://irahardianto.github.io/service-pattern-go/#install)
-------

Clone the source

    git clone https://github.com/irahardianto/service-pattern-go

Setup dependencies

    go get -u github.com/gorilla/mux
    go get -u github.com/jinzhu/gorm
    go get github.com/stretchr/testify

Setup sqlite data structure

    sqlite3 /var/tmp/gorm.db < setup.sql

Run the app

    go run main.go router.go servicecontainer.go

and visit

    http://localhost:8080/getPlayer/101


----------

[Introduction](https://irahardianto.github.io/service-pattern-go/#introduction)
-------
This is an example of Go clean architecture implementing Dependency Injection and Mocking for unit testing purposes to achieve safe, reliable and secure source code.

The idea of the pattern itself is to create decoupled systems that the implementation of lower level domain is not a concern of the implementor, and can be replaced without having concern of breaking implementor function.

The aim of the architecture is to produce a system that are:

 - Independent of frameworks. The system should be able to become an independent system, not bound into any framework implementation that cause the system to be bloated, instead those framework should be used as a tools to support the system implementation rather than limiting the system capabilities.
 - Highly testable. All codes are guilty and tests is the only way we can prove it otherwise, this means that our test coverage has to be able to cover as much layers as we can so we can be sure of our code reliability.
 - Independent of database. Business logic should not be bound to the database, the system should be able to swap Maria DB, Mongo DB, Dynamo DB without breaking the logic.
 - Independent of 3rd party library. No 3rd party library should be implemented directly to the system logic, we should abstract in away that our system can replace the library anytime we want.

Every implementation should only be by using interface, there should be no direct access from the implementor to implementation, that way we can inject its dependency and replace it with mock object during unit tests. For example:

 - PlayerController -> implement IPlayerService,  instead of direct PlayerService

        type PlayerController struct {
          PlayerService interface.IPlayerService
          PlayerHelper  helpers.PlayerHelper
        }

        func (controller *PlayerController) GetPlayer(res http.ResponseWriter, req *http.Request) {
        	playerId, _ := strconv.Atoi(req.FormValue("playerId"))
        	player := controller.PlayerService.FindById(playerId)
        	playerVM := controller.PlayerHelper.BuildPlayerVM(player)

          json.NewEncoder(res).Encode(playerVM)
        }

 - PlayerService -> implement IPlayerRepository, instead of direct PlayerRepository

       type PlayerService struct {
         PlayerRepository interfaces.IPlayerRepository
       }

       func (repository *PlayerService) FindById(playerId int) models.Player {

         player := repository.PlayerRepository.GetPlayerById(playerId)

         return player
       }

If you look into the implementation of these lines

      player := controller.PlayerService.FindById(playerId)

      player := repository.PlayerRepository.GetPlayerById(playerId)

Both are actually abstract implementation of the interface, not the real implementation itself.
So later on the Dependency Injection section, we will learn those interface will be injected with the implementation during the compile time. This way, we can switch the implementation of IPlayerService & IPlayerRepository during the injection with whatever implementation without changing the implementation logic.

Router that is used should only the one that **net/http** compatible, that way we can use **net/http/httptest** to unit test our controllers and be sure that we have proper implementation.

----------

[Folder Structure](https://irahardianto.github.io/service-pattern-go/#folder-structure)
-------
    /
    |- commands
    |- configurations
    |- controllers
    |- helpers
    |- infrastructures
    |- interfaces
    |- models
    |- repositories
    |- services
    |- viewmodels
    main.go
    router.go
    servicecontainer.go

The folder structure is created to accomodate seperation of concern principle, where every struct should have single responsibility to achieve decoupled system.

Every folder is a namespace of their own, and every file / struct under the same folder should only use the same namepace as their root folder.

**commands**

commands folder hosts all the structs under commands namespace, commands are intended to run console app that don't need interaction like crons and daemon alike.

**configurations**

configurations folder hosts all the structs under configurations namespace, the folder should hold configurations of the systems, the environment variables, etc.

**controllers**

controllers folder hosts all the structs under controllers namespace, controllers are the handler of all requests coming in, to the router, its doing just that, business logic and data access layer should be done separately.

controller struct implement services through their interface, no direct services implementation should be done in controller, this is done to maintain decoupled systems. The implementation will be injected during the compiled time.

**helpers**

controllers folder hosts all the structs under controllers namespace

**infrasctructures**

infrasctructures folder host all structs under infrasctructures namespace, infrasctructures consists of setup for the system to connect to external data source, it is used to host things like database connection configurations, MySQL, MariaDB, MongoDB, DynamoDB.

**interfaces**

interfaces folder hosts all the structs under interfaces namespace, interfaces as the name suggest are the bridge between different domain so they can interact with each other, in our case, this should be the only way for them to interact.

interface in Go is a bit different then you might already find in other language like Java or C#, while the later implements interface explicitly, Go implements interface implicitly. You just need to implement all method the interface has, and you're good to "Go".

In our system, our PlayerController implements IPlayerService to be able to interact with the implementation that will be injected. In our case, IPlayerService will be injected with PlayerService.

The same thing applies on PlayerService which implements IPlayerRepository to be able interact with the injected implementation. In our case, IPlayerRepository will be injected with PlayerRepository during the compile time.

PlayerRepository on the other hand, will be injected with infrasctructure configuration that has been setup earlier, this ensure that you can change the implementation of PlayerRepository, without changing the implementor which in this case PlayerService let alone break it. The same thing goes to PlayerService and PlayerController relationship, we can refactor PlayerService, we can change it however we want, without touching the implementor which is PlayerController.

**models**

models folder hosts all structs under models namespace, model is a struct reflecting our data object from / to database. models should only define data structs, no other functionalities should be included here.

**repositories**

repositories folder hosts all structs under repositories namespace, repositories is where the implementation of data access layer. All queries and data operation from / to database should happen here, and the implementor should be agnostic of what is the database engine is used, how the queries is done, all they care is they can pull the data according to the interface they are implementing.

**services**

services folder hosts all structs under services namespace, services is where the business logic lies on, it handles controller request and fetch data from data layer it needs and run their logic to satisfy what controller expect the service to return.

controller might implement many services interface to satisfy the request needs, and controller should be agnostic of how services implements their logic, all they care is that they should be able to pull the result they need according to the interface they implements.

**viewmodels**

viewmodels folder hosts all the structs under viewmodels namespace, viewmodels are model to be use as a response return of REST API call

**main.go**

main.go is the entry point of our system, here lies the router bindings it triggers ChiRouter singleton and call InitRouter to bind the router.

**router.go**

router.go is where we binds controllers to appropriate route to handle desired http request.

**servicecontainer.go**

servicecontainer.go is where the magic begins, this is the place where we injected all implementations of interfaces.


----------

[Naming Convention](https://irahardianto.github.io/service-pattern-go/#naming-convention)
-------
- Folders & Namespaces

  Folder naming must be :
  1. Descriptive
  2. Prefer short than long word

- Files

  Files naming must be :
  1. Use mixedCaps with first uppercase, eg : DateTime.go, not dateTime.go, String.go not string.go
  2. Interface function file name must be follow by "I" character, match with interface function declaration eg : IConnection, IRewardsService
  3. Descriptive, file name must be describe what it's functions.  
  4. Follow SRP (Single Responsibility Priciple), File name is the first time to look before reach the code inside, don't mixed String with DateTime.

- Struct

  There are two kind of struct in declaration, first declaring data model and last one wiring things. 

- Interface

  Interface name must be start with "I" character, for example "IConnection", "IRewardService".  

- Variable declaration

  Variable names should be short rather than long. This is especially true for local variables with limited scope. Prefer c to lineCount. Prefer i to sliceIndex. For other variable use mixedCaps (eg : GlobalVariable, privateVariable)

- Functions

  Function names must be as descriptive as possible, don't use long word if not necessary. Same as declaring variable, function name must be use mixedCaps

- Main modules

----------

[Commentary](https://irahardianto.github.io/service-pattern-go/#commentary)
------


----------

[Dependecy Injection](https://irahardianto.github.io/service-pattern-go/#dependency-injection)
-------


----------

[Mocking](https://irahardianto.github.io/service-pattern-go/#mocking)
-------

Cheers,
M. Ichsan Rahardianto.
