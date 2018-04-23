# Theory
We want to make sure the structure of our app is easily extendable and doesn’t demotivate us when we open the project :)
## Clean Architecture principles
To achieve this we are following the [Clean Architecture principles by Uncle Bob](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html). You can read the detailed description of ideas in the article above, while the main idea is to separate the code into **business logic**, **data storage and retrieval** from the outer world and **presentation** of the previous two things.

The main source of inspiration for us is the article [Architecting Android… The clean way? by Fernando Cejas](https://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/). Please make sure you read it before planning the architecture on your project.

![Clean Architecture scheme](https://8thlight.com/blog/assets/posts/2012-08-13-the-clean-architecture/CleanArchitecture-8d1fe066e8f7fa9c7d8e84c1a6b0e2b74b2c670ff8052828f4a7e73fcbbc698c.jpg)

*The main picture in every Google search result for Clean Architecture - why not put it here, too?*

![CLean Architecture in Android by Fernando Cejas](https://fernandocejas.com/assets/migrated/clean_architecture1.png)

*The structure from Fernando Cejas’ article*

The main idea in short is this: the app consists of the layers, and only outer layers can access inner ones, not vice versa. 
In the heart of the application lies the **Domain Logic layer**. It contains the operations that the user performs while in the app (we call them *Use Cases*) and the business rules that apply to those operations (*Entities*, but usually they sound more like *OrderPriceCalculator, GameFlow*, etc); for more details see the Domain section.

After that the things that connect the Domain logic with the real world go: the **Presentation Logic layer** that includes *Gateways, Repositories, Presenters*, etc. All these classes are middle-men between the device, network, databases and the abstract *Entity* and *Use Case* classes. This layer was split by Fernando Cejas into 2 parts: **Presentation** and **Data**. The first one contains the UI presentation classes (*Presenters / ViewModel* / others), the second one contains everything for accessing the data (*UserGateway, OrdersRepository*). For more info see the corresponding sections Presentation and Data.

The most important rule we have to remember is the *Dependency Rule*: **source code dependencies can only point inwards**.

In our case Domain is the most important guy. Other 2 layers are dependent on it (both Data and Presentation). This means that both Data and Presentation can use classes from Domain, but Domain knows nothing about Data or Presentation.

In [this article](https://habrahabr.ru/company/mobileup/blog/335382/) a description of the main difficult points is given, please make sure you read it before planning the architecture on your project.

## Domain layer
The Domain layer is the most interesting and at the same time abstract one. The most important rule for the Android apps: **there should be absolutely NO Android dependencies in Domain**. This is the inner layer, so it knows nothing about what’s happening in the outer world.

Here are some examples of what you would want to consider as Domain:
* Logic of calculating a product’s price;
* The state of the game / quiz / any other operation that takes more than 1 screen to complete and is not persisted between sessions;
* Building a list of recommendations based on the information we have about the user;
* Checking if all needed settings are set before requesting information about the next screen, etc.

Classes examples:
* *GetUserUseCase*,
* *SendMessageUseCase*,
* *SendUserAnswerUseCase*,
* *OrderPriceCalculator*,
* *GameFlow*,
* *RecommendationsBuilder*,
* *SettingsChecker*.

This layer is the most difficult to detect, it comes with practice. It’s also pretty hard to give a lot of suggestions here - this layer is determined by the app entirely, as it has its Domain logic. 
One possible way to detect Domain is thinking like this: if my app was also on iOS and Web, which parts/logic were common among platforms?
## Data part
The Data part of the app is responsible for saving information and retrieving it from the outer world. Here’s a list of things that usually go to this layer:
* Shared Preferences;
* Database (SQLite, Realm, Room);
* Accessing information in KeyStore;
* Network APIs - *getUser()*, *saveNewUserName()*, *postOrder()*, *login()* / *signUp()*, etc;
* Socket connections and others.
The Data classes usually implement interfaces from the Domain. This layer is outer for the Domain, so it can use the Domain’s classes. 
## Presentation part
For the Presentation layer we usually use the MVP pattern. Here’s a nice diagram of the structure:
![MVP](https://cdn-images-1.medium.com/max/1024/1*wXX8ipcYFeMqQBENMqM8xA.png)
MVP allows us to separate the presentation logic from the presentation itself. This separation helps with testing, makes the code readable, easy to understand and easier to maintain in future.
There are two most popular options of MVP, [Passive View and Supervising Controller](https://msdn.microsoft.com/en-us/library/ff709839.aspx). We are more inclined to use Passive View in general, meaning that the presenter takes responsibility for all actions and updates.

Some useful sources on MVP:
* [A nice article about MVP](http://engineering.remind.com/android-code-that-scales/);
* [Small examples of MVP structure](https://android.jlelse.eu/presentation-model-and-passive-view-in-mvp-the-android-way-fdba56a35b1e);
* [An article about Passive View by Martin Fowler](https://martinfowler.com/eaaDev/PassiveScreen.html).

In our case the *Presenters* use the Domain’s *Use Cases* and, if needed, other classes from that layer. Since Data and Presentation parts are in the same layer, we technically *can* call them directly in each other - but **we never do that**. The reason for this is separation of concerns: logic, not presentation decides where to get/put the data.

# Practice
## How to use?
In your project you can either write everything from scratch or use some boilerplate code (like [this one](https://github.com/bufferapp/android-clean-architecture-boilerplate)). It’s up to you, but please keep in mind that boilerplate can sometimes harm flexibility.
Here are some samples to get you started:
* [Fernando Cejas’ sample](https://github.com/android10/Android-CleanArchitecture);
* [sample by Anton Makhrov](https://github.com/uptechteam/CleanArchExample) for his [article on Clean Architecture](https://blog.uptech.team/clean-architecture-in-android-with-kotlin-rxjava-dagger-2-2fdc7441edfc);
* [Example from google samples](https://github.com/googlesamples/android-architecture/tree/todo-mvp-clean/todoapp);
* [Clean Architecture sample from Buffer](https://github.com/bufferapp/android-clean-architecture-boilerplate)
## When to use?
* When we create the project from scratch.
* When we can refactor the existing project.
* When we add new features that allow us to do so (separate from the existing codebase, small enough to try it out, etc)
