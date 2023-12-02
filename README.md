# Inversion of Control with Unity

Unity is a good game development tool and honestly, I like most of its features. However, the programming framework is awkward to use on big projects where code maintainability is of fundamental importance. As you may have guessed, the following considerations in this article do not apply to simple projects or projects developed by one or two coders. For these specific cases, Unity is good enough. The arguments in this article apply to relatively complex projects developed by a medium-big team.

After years of analysing the intrinsic issues of the Unity framework, I realised that they all share the same root: the way Unity injects dependencies inside entities, or better saying, how Unity makes this natural process very difficult.

A dependency can be defined as an object needed by another object to execute its code. If object A needs object B to execute its code, B is a dependency for A. When B is passed into A, we say that the dependency B is injected into object A.

The problem
Unity code framework is designed around Bilas’ Data-Driven GameObject System1. A component-based entity framework brings many benefits: pushes the users to favour composition over inheritance, keeps the classes small, clean and focused on a single responsibility, and promotes modularity. This is in a perfect world.

In Unity, the Entities are called GameObjects and the Components are based on MonoBehaviour implementations. Components add “behaviours” to Entities and theoretically, they should be perfectly encapsulated. The Entity Component structure relies on the fact that components define the whole entity’s logic. In order to use a Gameobjects framework properly, the following should be followed:

MonoBehaviours must operate within the GameObjects they are attached to. This is needed to ensure that encapsulation is not broken, as it shouldn’t be needed to. Components, by design, must handle only the Entity they are attached to.
MonoBehaviours are modular and single-responsibility classes reusable between GameObjects.
The behaviour of a GameObject should be extended by adding MonoBehaviours instead of adding code to an existing MonoBehaviour. One component must add only one behaviour to encourage modularity.
GameObjects should not be created without a view such as a mesh or a collider or anything that is actually an entity in the game (i.e.: Monobehaviours shouldn’t be managers)
MonoBehaviours can know other components on the same GameObject using GetComponent, however, they shouldn’t know MonoBehaviours from other GameObjects.
The first three points are obvious, even to those who have never heard about Entity Component frameworks. The Components design allows adding logic to an entity without using inheritance. We can say that instead to specialize the behaviour of an Entity vertically (usually specializing classes through polymorphism), Components allow expanding behaviours horizontally. This is not the place to talk about it, but in case you didn’t know, composition over inheritance is a well-known practice of which benefits have been proved extensively. The great thing about an Entity Component framework is then that composition comes naturally, without any particular effort. The smaller the component, the easier will be to reuse it, that’s why giving a single responsibility to each component will promote modularity.

The fourth point may seem an arbitrary rule, but Unity entities by default implement the Transform class, implying that the object must be placed in the world. What’s the point of having a Transform by default if it’s not used?

Last point starts to uncover the real issue. MonoBehaviours are meant to add behaviours on the GameObject that they are attached to and consequentially Unity can’t provide a clean way to let GameObects know each other without resorting to awkward solutions. The other problem is that Unity asks the user to use MonoBehaviours even when the logic to write is not related to an Entity at all. For example: let’s say there is an object called MonsterManager in the game world which must know the monsters that are still alive. When a monster dies, how can this object know it?

Currently, there are 2 simple solutions to this problem:

MonsterManager is a Monobehaviour created inside a GameObject that has no view. All the monsters will look for the MonsterManager class using the GameObject.Find or Object.FindObjectOfType function inside the Start or Awake methods and add themselves into the MonsterManager. MonsterManager could potentially listen to a Monster event to know when it will die.
MonsterManager is a Singleton and it’s used directly inside the Monster Monobehaviour. When a monster dies, a public MonsterManager function is called.
both cases are needed to solve the same problem: Inject dependencies. In the case A, MonsterManager is a dependency for the Monster objects. The Monster objects cannot add themselves to the MonsterManager if they don’t know the MonsterManager object. In the second case, the dependency is solved through the use of a global variable. You may have heard that global variables and singletons are very bad to use and you may not have fully understood why yet. For the moment, let’s focus on the practical aspects of these solutions:

What’s wrong with GameObject.Find
Aside from being quite a slow function, GameObject.Find is one example of how awkward the Unity Framework is for big projects development. What happens if someone in the team decides to rename the GameObject? Should GameObjects renaming or deletion be forbidden? GameObject.Find can lead to several run-time errors that cannot be caught at compiling time. This scenario could be really hard to manage when dozens of GameObjects are searched through this function. GameObject.Find and similar functions should be definitively abolished by the Unity framework (in favour of better alternatives).

What’s wrong with the Object.FindObjectOfType
How should the MonsterManager object be injected inside the Monster class then? There is actually another solution: calling the function Object.FindObjectOfType. However, what is FindObjectOfType? Object.FindObjectOfType could be seen as a wrong implementation of the Service Locator Pattern, with the difference that it is not possible to abstract the interface of the service from its implementation. This is another problem of the Unity framework, which I will now briefly hint at, but that would need another entire article, Unity discourages the use of interfaces. The interface is among the most powerful concepts at the core of every well-designed code. Instead of promoting the usage of interfaces and therefore abstraction, Unity pushes the coder to use Monobehaviour, even when the use of Monobehaviour is not necessary.

What’s wrong with the Singleton Pattern
Singleton is a controversial argument since the pattern has been invented. I am personally against the use of Singleton; every single Singleton class I have encountered so far, led only to design problems hard to fix through refactoring. However, if you ask me why I am against Singletons, I will not answer you with the usual answers (they break encapsulation, hide dependencies, make it impossible to write proper unit tests, bind the code to the implementation of service instead of its abstraction, make refactoring really awkward, usually create memory leaks), but with the hindsight of the practice: your code will become unmantainable. This is because the use of Singletons comes without any design constriction that, while apparently makes the coder life easier, will make it a hell later on, when the code turns into an incomprehensible blob without a structured flow (spaghetti code).

Projects developed by one of two coders wouldn’t experience the problem at first or ever (for example, if the project doesn’t need to be maintained over time), but mid-size project developers will pay the consequences of wild designs when they realise that refactoring became a very costly operation. Think about it, you can use Singletons everywhere, without limits, without rules. What would keep a coder from making a total mess out of it? Common sense? Let’s not fool ourselves, common sense doesn’t apply when release deadlines approach fast.

If this is not enough to dismiss Singletons think of this: what is a Singleton if not just another way to inject dependencies? When you don’t have any other choice than singletons to inject dependencies, you can be sure that your design smells!

Furthermore, singletons must rely on public functions, which may change states inside the Singleton itself. When the singleton is used recklessly in several parts of the code, it becomes very hard to track when and how these states changed. Refactoring becomes tremendously risky because changing the order of the calls of the public functions of a singleton could mean finding the singleton itself in an unexpected state. The biggest problem about Singleton is then the fact that they cannot be designed with Inversion of Control in mind. You are always forced to write functions like IsSingletonReady() or DisableThisSpecificBehaviour() or AccessToThisGlobalData(), leaving the control of the internal states to external entities there are hard to track, especially due to the Singleton being fundamentally a globally accessible variable.

Communication between objects is the hardest OOP problem to solve and the main reason why dependencies are injected is to let objects communicate with each other. Communication is impossible without any form of injection. Therefore while in some rare cases, the use of a Singleton could be acceptable, you can’t work with a framework that won’t let you use any other kind of injection. While it’s feasible, it’s totally unreasonable to solve all kinds of communication problems through singletons.

The solution
There are several ways to resolve dependencies: using the Service Locator pattern, injecting dependencies manually through constructors/setters, exploiting reflection capabilities through an IoC container and, obviously, using Singletons. Singletons aside, I use the Service Locator Pattern ONLY when it’s used to inject actual stateless services. Manual injection would be the preferable solution, but since Unity doesn’t have an entry point where dependencies can be created and injected, manual injections become quite hard to achieve without a good understanding of the Unity limitations.

The other solution available is to use IoC containers. Another article is needed to explain what an IoC container is, therefore I will give a first simple definition: An IoC container is a…container that creates and contains the dependencies that must be injected inside the application objects. It is called Inversion Of Control because the design is thought in such a way that the dependencies are never created by the user, but they are lazily created by the container when they are required.

There are several IoC containers out there, a lot written in c# as well, but practically none work in Unity (at the time of this article writing, that was 2012, ) and most of all, they are damn complicated*. For these reasons, I decided to create an IoC container for Unity, trying to keep it as simple as possible. Actually creating a basic IoC container is pretty straightforward and everybody can do it, my implementation has just a few tweaks that make it simple to use with Unity projects.

In my previous article, I briefly introduced the problems related to injecting dependencies using the Unity engine framework. I did not give an in-depth explanation because only those who understood the problem by themselves, can really understand why a solution is needed. If this is not the case, one may think that all this theory is irrelevant as long as “things get done”. However, In a big team/complex project environment is actually important how things get done too. Getting things done for the sake of finishing them as soon as possible, may lead to some tragic consequences. The truth is that keeping your code maintainable is as important as getting things done since, for large projects, the long-term cost of maintenance is higher than the cost of developing new features.

Cjg9a5ZWgAAPTcS[1]
Completing a feature in two days and then spending five days fixing the relative bugs found in production, due to the complexity of the solution adopted, won’t make anyone look good. Breaking good design practices to complete a feature in record time and then forcing someone else to spend several days to refactor it in order to extend its functionalities, won’t make anyone look good either.

I spent a long time during my carrier understanding how to write maintainable code and I eventually realised that the most important condition is to write simple code, so simple that doesn’t even need to be commented to be clearly understood by another coder. Writing simple code isn’t an easy task, often, in fact, those who think to get things done easily end up writing very convoluted (over-engineered) code, hard to read, maintain and refactor. I also realised that the only way to be able to write simple code for each possible problem is to deeply understand all the aspects involved in good code design.

These aspects are fundamentally explained in the SOLID principles, principles that I will mention several times in my next articles. Being able to inject dependencies efficiently plays a key role and that’s why I eventually decided to implement an IoC container.

Before discussing the example I built on purpose to show the features of my IoC container, I want to share my experience with other IoC containers I used before writing mine. As a game developer, the Inversion of Control concept was unknown to me, although I surely used IoC without knowing what it was and what it was called.

My experience with other IoC containers
When I was introduced to the concept of Dependency Injection, the IoC containers were already extensively used to ease our code tasks. That’s why, when I shifted from traditional C++ OOP to event-based programming in Actionscript, I started to use Robotlegs. Robotlegs was an IoC Container for AS3 language.

After I used it for a couple of small personal projects, I decided to abandon it. This is because eventually, I ended up realising that manual Dependency Injection was a better practice. However, after some time, when I moved to C#, I started to experiment with Ninject and after discussing the practices with its author, I realised that I was not appreciating the use of an IoC container because I did not understand its principles *1.

To understand how to use an IoC container, I need to introduce some concepts first:

Composition Root
Composition Root is a key concept to understand. The Composition Root is where the IoC container must be initialized before everything else start to use it. Since an application could use several contextualised containers, it could have more composition roots as well. The Composition Root is found in every application that has a main entry point, however, Unity doesn’t really have a single entry point that allows objects to be initialized and this is actually quite simply the real cardinal issue of the whole Unity code design. Since it’s of such significant importance, I will come back to this concept several times in my next articles. I continue using the concept of Composition Root even with Svelto.ECS.

Dependency Graph
Once objects are used inside other objects, they start to form a graph of dependencies. Let’s say that there is a class A and a class B, then there is a class C that uses A and a class D that uses B and C. D cannot work without B and C, C cannot work without A. This waterfall of dependencies is called Dependency Graph.

How to use an IoC Container
If, for the moment, we don’t consider objects that must be created dynamically after the start-up phase, all the dependencies can be solved right at the beginning of the application, within the Composition Root context.

When I started to use IoC containers, my first mistake was to misunderstand their use. I didn’t realise that the container would have taken the responsibility of creating dependencies away from me. I just couldn’t imagine a design where I would stop using the new keyword to create dependencies. If we didn’t use an IoC container, this would mean that all the dependencies would be created and initially passed only from the Composition Root. With an IoC container instead, the creation and injection of dependencies work in a different way, as I will soon show to you. If you haven’t grasped the meaning of this paragraph yet, don’t worry, I will return to this with more examples in my next articles.

Let’s take a look at what a Composition Root could look like using an IoC container with the following example:

    void SetupContainer()
    {
        container = new IoCContainer();
        
        //interface is bound to a specific instance
        container.Bind<IGameObjectFactory>().AsSingle(new GameObjectFactory(container));
        //interface is bound to a specific implementation, the same instance will be used once created
        container.Bind<IMonsterCounter>().AsSingle<MonsterCountHolder>();
        container.Bind<IMonsterCountHolder>().AsSingle<MonsterCountHolder>();
        //once the dependency is requested, a new instance will be created
        container.Bind<WeaponPresenter>().ToFactory(new MultiProvider<WeaponPresenter>());
        container.Bind<MonsterPresenter>().ToFactory(new MultiProvider<MonsterPresenter>());
        container.Bind<MonsterPathFollower>().ToFactory(new MultiProvider<MonsterPathFollower>());
        //once requested, the same instance will be used
        container.BindSelf<UnderAttackSystem>();
        container.BindSelf<PathController>();
    }
the container object is not used in any other part of the example, except for Factories. the Factory pattern is a special case and I will explain it later.

What is happening? Inside the SetupContainer method, all dependency types are registered into the container. With the method Bind, I am telling the container that every time a dependency of a known type is found, it must be solved with an instance of the type bound to it. The dependencies are solved lazily (that means only when needed) through the metatag [Inject].

The application flow starts from this method:

    void StartGame()
    {
        //tickEngine could be added in the container as well
        //if needed to other classes!
        UnityTicker tickEngine = new UnityTicker(); 
        
        tickEngine.Add(container.Inject(new MonsterSpawner()));
        tickEngine.Add(container.Build<UnderAttackSystem>());
    }
After the MonsterSpawner is built, it will have the dependency IMonsterCounter injected because it has been declared as

[Inject] public IMonsterCounter     monsterCounter      { set; private get; }
and because IMonsterCounter has been previously registered and bound to a valid implementation (MonsterCountHolder) inside the container. In its turn, if the MonsterCounterHolder class has some other tagged dependencies, they would be injected as well and so on…

In a complicated project scenario, this system will eventually give the impression that the [Inject] metatag becomes a sort of magic keyword able to solve automatically our dependencies. Of course, there is nothing magic and, differently than Singleton containers, our dependencies will be correctly injected ONLY if they are part of the Dependency Graph.

Compared to the use of a Singleton, an IoC container gives the same flexibility, but without risking wildly using static classes everywhere (and theoretically avoid memory leakings). Dependencies will be correctly injected only if the object that uses them is part of the Dependency Graph. Dependencies can be also injected through interfaces, which allows changing implementations without changing the code that uses them. In fact, it would be enough to swap the MonsterCounterHolder with another type that implements IMonsterCounter to have new behaviours without changing code in all the places where the dependencies are used.

In reality, you can perform more fancy tasks with an IoC container, as logic can be injected to change how the container decides to solve a dependency, but this is not relevant for this article (just check how many options a modern IoC container provides).

IoC container and the keyword new
It is called Inversion Of Control because the flow of the objects creation is inverted. The control of the creation is not anymore on the user, but on the framework that creates and injects objects for us. The most powerful benefit behind this concept is that the code will not depend on the implementation of the classes anymore, but always on their abstractions (if interfaces are used). Relying on the abstraction of the classes gives many benefits that can be fully appreciated when practices like refactoring and unit testing are regularly used. all that said, one could ask: if I should not use new, how is it possible to create objects dynamically, while the application runs, like for example spawning objects in the game world? As we have seen, the answer is to use Factories. Factories, in my design, can use the container to inject dependencies into the object that they create. Factories will be also injected as dependencies, so they can be easily used whenever are needed:

public [IoC.Inject] IBulletFactory bulletFactory { set; private get }

void OnBulletMustBeCreated()
{
    add(bulletFactory.Create());
}
Remembering that factories are the only classes that can use the Container outside the Composition Root, the IBulletFactory Create function would look like this:

public IBullet Create()
{
  return container.Inject(new Bullet());
}
In this way the Bullets will not just be created, but all their dependencies regularly injected, in run-time.

If you wonder why your classes should not be able to create objects on their own, remember that this is not overengineering, this is separation of concerns. Creating bullets and handling them are two different responsibilities.

IoC container and Unity
The use of an IoC container, similar to the one I am showing in this article, will help to shift the code patterns from intensive use of Monobehaviour to the use of normal classes that implement interfaces.

A MonoBehaviour, in its pure form, is a useful tool and its use must be encouraged when appropriate. Still, our Monobehaviours may need dependencies that must be injected. If the MonoBehaviours are created dynamically through a factory, as it should happen most of the time, then the dependencies are solved automatically. On the other hand, if MonoBehaviours are created implicitly by Unity (Gameobject on the scene), their dependencies cannot be solved within the Composition Root because of the nature of the Unity Framework.

Although in my example the turrets are not created dynamically, their dependencies are automatically solved by the IoC Container during the Awake time and before any Start function is called. The trick is to use a GameObject with the IoC container MonoBehaviour as the root of all the GameObjects that need dependencies injected. The IoC container then looks for the Monobehaviours children of the Context GameObject, and will fulfil their dependencies.

This is a good reason to use an IoC container in Unity, however, the same Context trick can be used with other patterns (which I proved with the Svelto.ECS)

IoC Container and MVP
IoC container becomes also very useful when patterns like the Model-View-Presenter (MVC and MVVM included) are used. Explaining the importance of using patterns like MVP to develop GUIs is not part of the scope of this article. However, you must know that MVP is great to separate the logic from data from their visual representation. With Unity, A GameObject would just handle the visual aspect of the entity, while a Presenter would handle the logic. This would look more or less like this:

class View:Monobehaviour, IView
{
  [Inject] public IPresenter presenter { private get; set; }

  void Start()
  {
    presenter.SetView(this); //from now on the presenter can use the public function of this Monobehaviour
  }
}
IoC Container and hierarchical containers
A proper IoC container library should support hierarchical containers, without this fundamental feature, using an IoC library extensively can quickly degenerate into spaghetti code as I will explain later in my next articles. It’s bad practice to just bind every possible dependency in a single container, as this won’t promote separation of concerns. For example, if we create dependencies for an MVP based GUI, these dependencies could possibly make sense only for the GUI layer of the application. This would mean that the GUI should be handled by a specialised GUI context. Hierarchical containers are powerful because, while they encapsulate the dependencies needed for a specific context, they will also inherit dependencies from parent containers.

IoC Container and Unit Tests
Often the IoC container concept is associated with Unit Tests, however, there isn’t any direct link between the two practices although using an IoC container will help in writing testable code.

Unit tests must test exclusively the code of the testing class and never its dependencies. An IoC container will make it very simple to inject harmless stubs of dependencies that cannot ever break or affect the functionality of the code to test. This can happen thanks to the use of interfaces and the mockup is just one different (often empty) implementation used for the scope of the tests only.

Conclusion
before to conclude, I want to quote two answers I found on StackOverflow that could help to solve some other doubts:

from http://stackoverflow.com/a/2551161

The important thing to realize here is that you can (and should) write your code in a DI-friendly, but container-agnostic manner.

This means that you should always push the composition of dependencies to a point where you can’t possibly defer it any longer. This is called the Composition Root and is often placed in near the application’s entry point.

If you design your application in this way, your choice of DI Container (or no DI Container) revolves around a single place in your application, and you can quickly change strategy.

You can choose to use Poor Man’s DI if you only have a few dependencies, or you can choose to use a full-blown DI Container. Used in this fashion, you will have no dependency on any particular DI Container, so the choice becomes less crucial in terms of maintainability.

A DI Container helps you manage complextity, including object lifetime. Used like described here, it doesn’t do anything you couldn’t write in hand, but it does it better and more succinctly. As such, my threshold for when to start using a DI Container would be pretty low.

I would start using a DI Container once I get past a few dependencies. Most of them are pretty easy to get started with anyway.

and from http://stackoverflow.com/a/2066827

Pure encapsulation is an ideal that can never be achieved. If all dependencies were hidden then you wouldn’t have the need for DI at all. Think about it this way, if you truly have private values that can be internalized within the object, say for instance the integer value of the speed of a car object, then you have no external dependency and no need to invert or inject that dependency. These sorts of internal state values that are operated on purely by private functions are what you want to encapsulate always.

But if you’re building a car that wants a certain kind of engine object then you have an external dependency. You can either instantiate that engine — for instance new GMOverHeadCamEngine() — internally within the car object’s constructor, preserving encapsulation but creating a much more insidious coupling to a concrete class GMOverHeadCamEngine, or you can inject it, allowing your Car object to operate agnostically (and much more robustly) on for example an interface IEngine without the concrete dependency. Whether you use an IOC container or simple DI to achieve this is not the point — the point is that you’ve got a Car that can use many kinds of engines without being coupled to any of them, thus making your codebase more flexible and less prone to side effects.

DI is not a violation of encapsulation, it is a way of minimizing the coupling when encapsulation is necessarily broken as a matter of course within virtually every OOP project. Injecting a dependency into an interface externally minimizes coupling side effects and allows your classes to remain agnostic about implementation.
