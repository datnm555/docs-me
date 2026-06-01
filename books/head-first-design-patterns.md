# Head First Design Patterns — Summary

> **Eric Freeman & Elisabeth Robson** — *Head First Design Patterns: Building Extensible & Maintainable Object-Oriented Software*, 2nd edition (O'Reilly, 2020). First edition 2004. Java-based, but the lessons translate cleanly to C#, Kotlin, TypeScript, and Swift.

> **Why this book matters:** it is the friendliest, most-quoted introduction to the **Gang of Four** patterns. The "Head First" style — conversational, visual, repetitive on purpose — turns abstract pattern definitions into stories you remember years later. Each chapter is built around one concrete running example.

> **This summary covers:** the design principles the book teaches throughout, every chapter's pattern (intent, running example, key code, take-aways, pitfalls), the compound-pattern chapter (MVC), and the leftover-patterns appendix.

> 🇻🇳 Vietnamese version: [`head-first-design-patterns-vi.md`](./head-first-design-patterns-vi.md)

---

## Quick Reference (What · Why · When · Where)

- **What** — A digest of Freeman & Robson's friendly tour of the 14 GoF patterns Head First covers in depth (Strategy, Observer, Decorator, Factory, Singleton, Command, Adapter, Facade, Template Method, Iterator, Composite, State, Proxy, Compound/MVC) plus the 9 OO design principles the book accumulates chapter by chapter.
- **Why** — Gives you the patterns *plus* the running stories that make them memorable; bridges the gap between abstract pattern definitions (GoF) and recognizing the pattern in real code (your codebase).
- **When** — First time learning design patterns; before a system-design interview; when you want a canonical example to anchor a code-review argument ("this is Decorator, not Adapter — here's why").
- **Where** — Pair with `dot-net/docs/design-pattern/` (.NET implementations of all 23), `principles/` (the 9 OO principles in depth), and `oop/` (the pillars the book assumes).

---

## Table of Contents

1. [About the Book](#1-about-the-book)
2. [The "Head First" Approach](#2-the-head-first-approach)
3. [The Core OO Design Principles](#3-the-core-oo-design-principles)
4. [Chapter 1 — Strategy Pattern](#4-chapter-1--strategy-pattern)
5. [Chapter 2 — Observer Pattern](#5-chapter-2--observer-pattern)
6. [Chapter 3 — Decorator Pattern](#6-chapter-3--decorator-pattern)
7. [Chapter 4 — Factory Method & Abstract Factory](#7-chapter-4--factory-method--abstract-factory)
8. [Chapter 5 — Singleton Pattern](#8-chapter-5--singleton-pattern)
9. [Chapter 6 — Command Pattern](#9-chapter-6--command-pattern)
10. [Chapter 7 — Adapter & Facade Patterns](#10-chapter-7--adapter--facade-patterns)
11. [Chapter 8 — Template Method Pattern](#11-chapter-8--template-method-pattern)
12. [Chapter 9 — Iterator & Composite Patterns](#12-chapter-9--iterator--composite-patterns)
13. [Chapter 10 — State Pattern](#13-chapter-10--state-pattern)
14. [Chapter 11 — Proxy Pattern](#14-chapter-11--proxy-pattern)
15. [Chapter 12 — Compound Patterns (Model-View-Controller)](#15-chapter-12--compound-patterns-model-view-controller)
16. [Chapter 13 — Better Living with Patterns](#16-chapter-13--better-living-with-patterns)
17. [Appendix — Leftover Patterns](#17-appendix--leftover-patterns)
18. [Big-Picture Take-Aways](#18-big-picture-take-aways)
19. [Reading Order Recommendations](#19-reading-order-recommendations)
20. [Compared to the Original GoF Book](#20-compared-to-the-original-gof-book)
21. [References](#21-references)

---

## 1. About the Book

* **Title:** *Head First Design Patterns: Building Extensible & Maintainable Object-Oriented Software*
* **Authors:** Eric Freeman & Elisabeth Robson (originally co-authored with Bert Bates & Kathy Sierra in the 1st edition).
* **Publisher:** O'Reilly Media.
* **Editions:** 1st (2004), 2nd (2020 — refreshed for Java 8+ with lambdas, plus a new section on Java's `Observable` deprecation).
* **Length:** ~672 pages, but the format is unusual — heavy on diagrams, dialogues, and exercises.
* **Audience:** developers comfortable with basic Java/OO who want to *internalize* the GoF patterns rather than just recognize them.
* **License of examples:** the source code is open and famously imitated across the industry.

The book covers **14 patterns in depth** (Strategy, Observer, Decorator, Factory Method, Abstract Factory, Singleton, Command, Adapter, Facade, Template Method, Iterator, Composite, State, Proxy) plus the **MVC compound pattern** and a brief tour of the remaining GoF patterns in the appendix.

---

## 2. The "Head First" Approach

The book is a deliberate teaching experiment. The format relies on cognitive-science principles to make abstract ideas stick:

* **Conversational tone** — the prose talks *to* you, not at you.
* **Bizarre visuals & humor** — the brain remembers anomalies.
* **Running stories per chapter** — SimUDuck, Starbuzz Coffee, the Gumball Machine, the Home Theater… each chapter has a single, memorable example.
* **Bullet Points (the boxed summaries)** — at the end of every chapter, a short list distilling the lesson.
* **Sharpen Your Pencil exercises** — fill-in-the-blank UML, "guess the pattern", code-completion challenges.
* **Code Up Close** — annotated code showing exactly which lines implement which part of the pattern.
* **There Are No Dumb Questions** sidebars — anticipating reader doubts.
* **Pattern formality at the end** — only *after* the example is built does the book reveal the canonical GoF intent, structure, and consequences.

**The reading order matters.** Each chapter introduces principles that the next chapters build on. By Chapter 6 (Command), you have a working vocabulary of *encapsulate what varies*, *favor composition*, *program to an interface*, and *loose coupling*.

---

## 3. The Core OO Design Principles

The book accumulates a checklist of **OO Design Principles** as it goes. By the end of the book you have collected:

| #   | Principle                                                                            | Introduced in            |
| --- | ------------------------------------------------------------------------------------ | ------------------------ |
| 1   | **Identify the aspects of your application that vary and separate them from what stays the same.** | Chapter 1 — Strategy     |
| 2   | **Program to an interface, not an implementation.**                                  | Chapter 1 — Strategy     |
| 3   | **Favor composition over inheritance.**                                              | Chapter 1 — Strategy     |
| 4   | **Strive for loosely coupled designs between objects that interact.**                | Chapter 2 — Observer     |
| 5   | **Classes should be open for extension, but closed for modification.** (Open-Closed) | Chapter 3 — Decorator    |
| 6   | **Depend upon abstractions. Do not depend upon concrete classes.** (Dependency Inversion) | Chapter 4 — Factory   |
| 7   | **Principle of Least Knowledge — talk only to your immediate friends.** (Law of Demeter) | Chapter 7 — Facade   |
| 8   | **Don't call us, we'll call you.** (The Hollywood Principle)                          | Chapter 8 — Template Method |
| 9   | **A class should have only one reason to change.** (Single Responsibility Principle) | Chapter 9 — Iterator     |

These nine principles are the **backbone** of the book. Almost every pattern is presented as "the realization of principles X, Y, Z".

> **Mental model:** patterns are *applied principles*. If you internalize the nine principles above, you can usually reinvent the corresponding pattern on demand.

---

## 4. Chapter 1 — Strategy Pattern

> *"Welcome to Design Patterns."*

### The story — SimUDuck

You inherit a duck simulator. Originally, every `Duck` inherits from a base class and has `quack()` and `swim()`. To "add flying ducks", a developer adds `fly()` to the base class. Suddenly **rubber ducks fly** and **decoy ducks fly**. Inheritance pushed unwanted behavior into subclasses.

The lesson: **identify what varies, encapsulate it, and inject it via composition.**

### The pattern

> **Strategy:** define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from the clients that use it.

### The canonical code

```java
public interface FlyBehavior {
    void fly();
}

public class FlyWithWings implements FlyBehavior {
    public void fly() { System.out.println("I'm flying!"); }
}

public class FlyNoWay implements FlyBehavior {
    public void fly() { System.out.println("I can't fly"); }
}

public class FlyRocketPowered implements FlyBehavior {
    public void fly() { System.out.println("Rocket-powered flying"); }
}

public interface QuackBehavior {
    void quack();
}

public class Quack implements QuackBehavior {
    public void quack() { System.out.println("Quack"); }
}

public class MuteQuack implements QuackBehavior {
    public void quack() { System.out.println("<< Silence >>"); }
}

public abstract class Duck {
    FlyBehavior flyBehavior;
    QuackBehavior quackBehavior;

    public void performFly()   { flyBehavior.fly(); }
    public void performQuack() { quackBehavior.quack(); }

    // Composition: behavior can be swapped at runtime
    public void setFlyBehavior(FlyBehavior fb)   { this.flyBehavior = fb; }
    public void setQuackBehavior(QuackBehavior qb){ this.quackBehavior = qb; }

    public abstract void display();
    public void swim() { System.out.println("All ducks float"); }
}

public class MallardDuck extends Duck {
    public MallardDuck() {
        flyBehavior   = new FlyWithWings();
        quackBehavior = new Quack();
    }
    public void display() { System.out.println("I'm a real Mallard duck"); }
}

// Usage — swap behavior at runtime
Duck model = new ModelDuck();
model.performFly();                     // "I can't fly"
model.setFlyBehavior(new FlyRocketPowered());
model.performFly();                     // "Rocket-powered flying"
```

### Key points

* What varies → behavior. Encapsulate it behind an interface.
* What stays → the structure of `Duck` itself.
* **HAS-A** (composition) is more flexible than **IS-A** (inheritance).
* Behaviors are first-class objects you can pass around.

### Pitfalls

* Strategy ≠ static utility. The whole point is **runtime swappability**.
* Don't pre-extract a strategy for variation you don't have. Wait for the second concrete behavior.

---

## 5. Chapter 2 — Observer Pattern

> *"Keeping your objects in the know."*

### The story — Weather Station

A device collects temperature, humidity, and pressure. Multiple "displays" (current conditions, statistics, forecast) need updates whenever readings change. Without a pattern, the device class hardcodes every display. Adding a new display means editing the core.

### The pattern

> **Observer:** define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.

### The canonical code

```java
public interface Subject {
    void registerObserver(Observer o);
    void removeObserver(Observer o);
    void notifyObservers();
}

public interface Observer {
    void update(float temp, float humidity, float pressure);
}

public interface DisplayElement { void display(); }

public class WeatherData implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private float temperature, humidity, pressure;

    public void registerObserver(Observer o) { observers.add(o); }
    public void removeObserver(Observer o)   { observers.remove(o); }

    public void notifyObservers() {
        for (Observer o : observers) o.update(temperature, humidity, pressure);
    }

    public void measurementsChanged() { notifyObservers(); }

    public void setMeasurements(float t, float h, float p) {
        temperature = t; humidity = h; pressure = p;
        measurementsChanged();
    }
}

public class CurrentConditionsDisplay implements Observer, DisplayElement {
    private float temperature, humidity;
    private Subject weatherData;

    public CurrentConditionsDisplay(Subject weatherData) {
        this.weatherData = weatherData;
        weatherData.registerObserver(this);
    }

    public void update(float temp, float humidity, float pressure) {
        this.temperature = temp;
        this.humidity = humidity;
        display();
    }

    public void display() {
        System.out.println("Current: " + temperature + "F, " + humidity + "% humidity");
    }
}
```

### Key points

* **Loose coupling** — the subject knows only that observers implement `Observer`.
* Subjects and observers can be added/removed at runtime.
* Two flavors: **push** (subject sends data) and **pull** (observer queries subject). The book uses push and discusses the trade-off.
* The 2nd edition adds a note that Java's `java.util.Observable` is **deprecated since Java 9** — write your own subject interface (as above).

### Pitfalls

* **Order of notification is not guaranteed.** Don't rely on it.
* **Memory leak**: if observers never unregister, the subject keeps them alive.
* In multi-threaded code, the observer list needs synchronization or a copy-on-write list.

> **C# parallel:** the `event` keyword, `IObservable<T>`/`IObserver<T>` (Rx), MediatR `INotification`. In modern .NET you rarely roll your own subject interface.

---

## 6. Chapter 3 — Decorator Pattern

> *"Decorating objects."*

### The story — Starbuzz Coffee

A coffee shop sells beverages — DarkRoast, Espresso, HouseBlend, Decaf — plus condiments (Mocha, Soy, Whip, Steamed Milk). The naive design is class explosion: `DarkRoastWithMochaAndWhip`, `EspressoWithMochaAndSoy`, …

Each condiment changes price; each combo is unique. Subclassing every combination is impossible.

### The pattern

> **Decorator:** attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

### Principle introduced — Open/Closed

> **Classes should be open for extension, but closed for modification.**

You extend functionality by *wrapping* objects, never by editing the original code.

### The canonical code

```java
public abstract class Beverage {
    String description = "Unknown Beverage";
    public String getDescription() { return description; }
    public abstract double cost();
}

public abstract class CondimentDecorator extends Beverage {
    Beverage beverage;
    public abstract String getDescription();
}

public class Espresso extends Beverage {
    public Espresso() { description = "Espresso"; }
    public double cost() { return 1.99; }
}

public class Mocha extends CondimentDecorator {
    public Mocha(Beverage beverage) { this.beverage = beverage; }
    public String getDescription() { return beverage.getDescription() + ", Mocha"; }
    public double cost()           { return 0.20 + beverage.cost(); }
}

public class Whip extends CondimentDecorator {
    public Whip(Beverage beverage) { this.beverage = beverage; }
    public String getDescription() { return beverage.getDescription() + ", Whip"; }
    public double cost()           { return 0.10 + beverage.cost(); }
}

// Usage — compose at runtime
Beverage beverage = new Espresso();          // Espresso $1.99
beverage = new Mocha(beverage);              // Espresso, Mocha $2.19
beverage = new Mocha(beverage);              // Espresso, Mocha, Mocha $2.39
beverage = new Whip(beverage);               // Espresso, Mocha, Mocha, Whip $2.49

System.out.println(beverage.getDescription() + " $" + beverage.cost());
```

### Key points

* Decorators have **the same supertype** as the object they decorate.
* They **wrap** the object — multiple decorators stack arbitrarily deep.
* Behavior is added **dynamically** at construction time.
* Java's `java.io` library (`BufferedReader` wrapping `FileReader` wrapping `Reader`) is **the canonical real-world example**.

### Pitfalls

* **Decorators add many small classes** — recognize that the I/O API is the price you pay for compositional flexibility.
* The wrapped object's type is not preserved — relying on `instanceof` after wrapping breaks.
* Avoid decorators that add *new* methods to the interface; they won't be visible through the base type.

> **C# parallel:** ASP.NET Core middleware *is* the Decorator pattern around `HttpContext`; `DelegatingHandler` in `HttpClient`; Scrutor's `Decorate<T>()` helper.

---

## 7. Chapter 4 — Factory Method & Abstract Factory

> *"Baking with OO goodness."*

### The story — Pizza Store

A pizza store has `orderPizza(String type)`. Each `if (type.equals("cheese")) new CheesePizza()` block grows whenever you add a topping. Worse, you franchise the store: **New York** and **Chicago** styles need different ingredients but the same workflow.

### Two patterns covered

#### Factory Method

> Define an interface for creating an object, but let subclasses decide which class to instantiate.

```java
public abstract class PizzaStore {
    public Pizza orderPizza(String type) {
        Pizza pizza = createPizza(type);
        pizza.prepare();
        pizza.bake();
        pizza.cut();
        pizza.box();
        return pizza;
    }
    // Factory method — defers to subclass
    protected abstract Pizza createPizza(String type);
}

public class NYPizzaStore extends PizzaStore {
    protected Pizza createPizza(String type) {
        return switch (type) {
            case "cheese"   -> new NYStyleCheesePizza();
            case "veggie"   -> new NYStyleVeggiePizza();
            case "clam"     -> new NYStyleClamPizza();
            case "pepperoni"-> new NYStylePepperoniPizza();
            default          -> null;
        };
    }
}

public class ChicagoPizzaStore extends PizzaStore {
    protected Pizza createPizza(String type) {
        return switch (type) {
            case "cheese" -> new ChicagoStyleCheesePizza();
            // ...
            default       -> null;
        };
    }
}
```

#### Abstract Factory

> Provide an interface for creating **families** of related or dependent objects without specifying their concrete classes.

When the regional pizza varies not only in *which Pizza class* but in **which dough, sauce, cheese, veggies** to use:

```java
public interface PizzaIngredientFactory {
    Dough createDough();
    Sauce createSauce();
    Cheese createCheese();
    Veggies[] createVeggies();
    Pepperoni createPepperoni();
    Clams createClam();
}

public class NYPizzaIngredientFactory implements PizzaIngredientFactory {
    public Dough createDough()   { return new ThinCrustDough(); }
    public Sauce createSauce()   { return new MarinaraSauce(); }
    public Cheese createCheese() { return new ReggianoCheese(); }
    public Veggies[] createVeggies() { return new Veggies[]{ new Garlic(), new Onion(), new Mushroom() }; }
    public Pepperoni createPepperoni() { return new SlicedPepperoni(); }
    public Clams createClam()    { return new FreshClams(); }
}

public class CheesePizza extends Pizza {
    PizzaIngredientFactory ingredientFactory;
    public CheesePizza(PizzaIngredientFactory ingredientFactory) {
        this.ingredientFactory = ingredientFactory;
    }
    void prepare() {
        dough  = ingredientFactory.createDough();
        sauce  = ingredientFactory.createSauce();
        cheese = ingredientFactory.createCheese();
    }
}
```

### Principle introduced — Dependency Inversion

> **Depend upon abstractions. Do not depend upon concrete classes.**

The book lists three guidelines (not laws):

* No variable should hold a reference to a concrete class.
* No class should derive from a concrete class.
* No method should override an implemented method of any of its base classes.

These are *guidelines*. Strive to follow them as much as practical.

### Key points

* **Factory Method** uses **subclassing** to defer choice of concrete type.
* **Abstract Factory** uses **composition** to deliver a *family* of related types.
* Often paired: Factory Method **inside** Abstract Factory.
* Both **encapsulate object creation** — the most common source of code that violates OCP.

### Pitfalls

* **Simple Factory** (a static factory method that switches on type) is **not** the GoF Factory Method — the book calls it a "programming idiom", not a true pattern.
* Don't add a factory for every `new` call. Add one when you have a *real* choice between concrete types or when construction is *complex*.

---

## 8. Chapter 5 — Singleton Pattern

> *"One-of-a-kind objects."*

### The story — Chocolate Boiler

A chocolate factory has a single, expensive `ChocolateBoiler`. If two threads each create one and start it at the same time, the factory floods. Only **one instance** must ever exist.

### The pattern

> **Singleton:** ensure a class has only one instance, and provide a global point of access to it.

### Evolving the implementation

**Classic (NOT thread-safe):**

```java
public class Singleton {
    private static Singleton uniqueInstance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (uniqueInstance == null) {
            uniqueInstance = new Singleton();
        }
        return uniqueInstance;
    }
}
```

**Synchronized (safe, slow):**

```java
public static synchronized Singleton getInstance() {
    if (uniqueInstance == null) {
        uniqueInstance = new Singleton();
    }
    return uniqueInstance;
}
```

**Eager initialization:**

```java
public class Singleton {
    private static Singleton uniqueInstance = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return uniqueInstance; }
}
```

**Double-checked locking (Java 5+, needs `volatile`):**

```java
public class Singleton {
    private volatile static Singleton uniqueInstance;
    private Singleton() {}
    public static Singleton getInstance() {
        if (uniqueInstance == null) {
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

**Enum singleton (recommended in the 2nd ed., per Effective Java):**

```java
public enum Singleton {
    UNIQUE_INSTANCE;

    public void doSomething() { /* ... */ }
}
// Usage:
Singleton.UNIQUE_INSTANCE.doSomething();
```

### Key points

* Singleton fights two enemies: **multiple instances** and **concurrency races**.
* The enum form is the *simplest* correct form in Java; it handles serialization and reflection attacks for free.
* The 2nd edition explicitly cautions: **Singleton is often abused.** It hides global state, makes testing hard, and creates tight coupling.

### Pitfalls

* **Singletons make testing painful** — you can't substitute fakes.
* **Singletons hide dependencies.** A class that asks `MyService.getInstance()` lies about its needs.
* **Classloader / process boundaries** can create *multiple* "singletons" in JEE / multi-process scenarios.

> **Modern alternative:** use a DI container's **singleton lifetime**. You get one instance per application, full testability, and explicit dependencies. The `Singleton pattern as written` is rarely the right answer in 2025+ .NET / Spring / NestJS code.

---

## 9. Chapter 6 — Command Pattern

> *"Encapsulating invocation."*

### The story — The Universal Remote Control

A 7-button remote: each button can be programmed to control a different device (lights, fan, garage door, stereo). The remote must not know the API of each device. **Undo** is also required.

### The pattern

> **Command:** encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

### The canonical code

```java
public interface Command {
    void execute();
    void undo();
}

public class Light {
    public void on()  { System.out.println("Light on"); }
    public void off() { System.out.println("Light off"); }
}

public class LightOnCommand implements Command {
    Light light;
    public LightOnCommand(Light light) { this.light = light; }
    public void execute() { light.on(); }
    public void undo()    { light.off(); }
}

public class LightOffCommand implements Command {
    Light light;
    public LightOffCommand(Light light) { this.light = light; }
    public void execute() { light.off(); }
    public void undo()    { light.on(); }
}

public class NoCommand implements Command {        // Null Object — avoids null checks
    public void execute() {}
    public void undo()    {}
}

public class RemoteControl {
    Command[] onCommands  = new Command[7];
    Command[] offCommands = new Command[7];
    Command undoCommand;

    public RemoteControl() {
        Command noCommand = new NoCommand();
        for (int i = 0; i < 7; i++) {
            onCommands[i] = noCommand;
            offCommands[i] = noCommand;
        }
        undoCommand = noCommand;
    }

    public void setCommand(int slot, Command on, Command off) {
        onCommands[slot]  = on;
        offCommands[slot] = off;
    }

    public void onButtonWasPushed(int slot) {
        onCommands[slot].execute();
        undoCommand = onCommands[slot];
    }
    public void offButtonWasPushed(int slot) {
        offCommands[slot].execute();
        undoCommand = offCommands[slot];
    }
    public void undoButtonWasPushed() { undoCommand.undo(); }
}

// Macro command
public class MacroCommand implements Command {
    Command[] commands;
    public MacroCommand(Command[] commands) { this.commands = commands; }
    public void execute() { for (Command c : commands) c.execute(); }
    public void undo()    { for (Command c : commands) c.undo(); }
}
```

### Key points

* A command **encapsulates** receiver + action + parameters.
* The invoker (remote control) doesn't know what the command does.
* **Macro commands** are compositions of commands — pattern recursion.
* **Null Object** (`NoCommand`) avoids `null` checks everywhere.
* Commands enable **queuing**, **logging**, and **undo** by treating actions as data.

### Pitfalls

* For trivial single-action calls, a command object is overkill. Use a lambda instead.
* `undo()` is harder than `execute()` — many operations are not truly reversible.
* In modern code, **lambdas** are commands without ceremony.

> **C# parallel:** MediatR's `IRequest<T>` + `IRequestHandler`; CQRS commands; `Action`/`Func` delegates.

---

## 10. Chapter 7 — Adapter & Facade Patterns

> *"Being adaptive."*

Two patterns share a chapter because they both **wrap** something — but their *intent* differs.

### Adapter

> **Adapter:** convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

#### Running example — Ducks and Turkeys

```java
public interface Duck   { void quack(); void fly(); }
public interface Turkey { void gobble(); void fly(); }

public class WildTurkey implements Turkey {
    public void gobble() { System.out.println("Gobble gobble"); }
    public void fly()    { System.out.println("I'm flying a short distance"); }
}

public class TurkeyAdapter implements Duck {
    Turkey turkey;
    public TurkeyAdapter(Turkey turkey) { this.turkey = turkey; }
    public void quack() { turkey.gobble(); }
    public void fly() {
        for (int i = 0; i < 5; i++) turkey.fly();  // turkeys fly less than ducks
    }
}

// Usage
Turkey turkey = new WildTurkey();
Duck duckAdapter = new TurkeyAdapter(turkey);
duckAdapter.quack();  // gobble
duckAdapter.fly();    // flies 5 times
```

#### Two flavors of Adapter

* **Object Adapter** (composition — most common, what the example shows above).
* **Class Adapter** (multiple inheritance — not possible in Java/C#, included for completeness).

### Facade

> **Facade:** provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

#### Running example — Home Theater

To watch a movie, you must turn on the projector, dim the lights, lower the screen, fire up the amplifier, set surround sound, start the streaming player, and pop popcorn — seven devices, in order.

```java
public class HomeTheaterFacade {
    Amplifier amp;
    Tuner tuner;
    StreamingPlayer player;
    Projector projector;
    TheaterLights lights;
    Screen screen;
    PopcornPopper popper;

    public HomeTheaterFacade(Amplifier amp, Tuner tuner, StreamingPlayer player,
                             Projector projector, Screen screen, TheaterLights lights,
                             PopcornPopper popper) {
        this.amp = amp; this.tuner = tuner; this.player = player;
        this.projector = projector; this.screen = screen;
        this.lights = lights; this.popper = popper;
    }

    public void watchMovie(String movie) {
        System.out.println("Get ready to watch a movie...");
        popper.on();
        popper.pop();
        lights.dim(10);
        screen.down();
        projector.on();
        projector.wideScreenMode();
        amp.on();
        amp.setStreamingPlayer(player);
        amp.setSurroundSound();
        amp.setVolume(5);
        player.on();
        player.play(movie);
    }

    public void endMovie() {
        System.out.println("Shutting movie theater down...");
        popper.off();
        lights.on();
        screen.up();
        projector.off();
        amp.off();
        player.stop();
        player.off();
    }
}
```

### Principle introduced — Principle of Least Knowledge

> **Talk only to your immediate friends.**

A method should only invoke methods that belong to:
* the object itself
* objects passed in as parameters
* objects the method creates or instantiates
* components of the object (fields)

Not chains like `getA().getB().doSomething()` (the "train wreck").

### Key points

* **Adapter** *changes* an interface to match what the client expects.
* **Facade** *simplifies* an interface to a subsystem; it doesn't transform — it consolidates.
* Both wrap, but the **intent** differs (adapt vs. simplify).
* Facade reduces coupling (Law of Demeter) — clients talk only to the facade.

### Pitfalls

* A facade should be **easy to bypass**. If clients *must* use the facade, you have a controller, not a facade.
* Adapters that try to do too much logic become **God Adapters**. Keep them thin.

---

## 11. Chapter 8 — Template Method Pattern

> *"Encapsulating algorithms."*

### The story — Caffeine Beverages

Starbuzz prepares Coffee and Tea using nearly identical recipes:

* **Coffee:** boil water → brew coffee grinds → pour in cup → add sugar and milk.
* **Tea:** boil water → steep tea bag → pour in cup → add lemon.

The **structure** is the same: boil → brew/steep → pour → condiments. Only steps 2 and 4 differ.

### The pattern

> **Template Method:** define the skeleton of an algorithm in an operation, deferring some steps to subclasses. Template Method lets subclasses redefine certain steps of an algorithm without changing the algorithm's structure.

### The canonical code

```java
public abstract class CaffeineBeverage {
    // Template method — final so subclasses cannot break the algorithm
    public final void prepareRecipe() {
        boilWater();
        brew();
        pourInCup();
        addCondiments();
    }

    abstract void brew();           // subclass step
    abstract void addCondiments();  // subclass step

    void boilWater()  { System.out.println("Boiling water"); }
    void pourInCup()  { System.out.println("Pouring into cup"); }
}

public class Tea extends CaffeineBeverage {
    public void brew()           { System.out.println("Steeping the tea"); }
    public void addCondiments()  { System.out.println("Adding Lemon"); }
}

public class Coffee extends CaffeineBeverage {
    public void brew()           { System.out.println("Dripping coffee through filter"); }
    public void addCondiments()  { System.out.println("Adding Sugar and Milk"); }
}
```

### Optional hooks

A **hook** is a method declared in the abstract class with a default (often empty) implementation, which subclasses *may* override to influence the algorithm:

```java
public abstract class CaffeineBeverageWithHook {
    public final void prepareRecipe() {
        boilWater();
        brew();
        pourInCup();
        if (customerWantsCondiments()) {
            addCondiments();
        }
    }
    abstract void brew();
    abstract void addCondiments();
    void boilWater() { ... }
    void pourInCup() { ... }

    // Hook with default
    boolean customerWantsCondiments() { return true; }
}
```

### Principle introduced — The Hollywood Principle

> **Don't call us, we'll call you.**

The base class controls the algorithm; subclasses *plug in* steps without ever calling up the chain themselves. This is **inversion of control** at the class-design level — the same idea that powers frameworks like Spring, ASP.NET Core, and Angular.

### Key points

* Template Method = inheritance for *fixed structure with variable steps*.
* Subclasses fill in `abstract` steps and *optionally* override hooks.
* Mark the template method `final` to prevent subclasses from breaking the skeleton.
* The Hollywood Principle generalizes to **frameworks**: the framework calls your code.

### Pitfalls

* Inheritance couples subclasses to the base class. If the base changes the algorithm, every subclass is affected.
* For runtime flexibility, **Strategy** (composition) usually beats Template Method (inheritance).
* Don't put too much in the template — leave room for subclass intelligence.

> **C# parallel:** `BackgroundService.ExecuteAsync` is a template method; `WebApplicationBuilder` lifecycle hooks; ASP.NET Core controller filters.

---

## 12. Chapter 9 — Iterator & Composite Patterns

> *"Well-managed collections."*

### The story — Merging Menus

A new diner-coffeehouse merges three menus: PancakeHouse (uses `ArrayList`), DinerMenu (uses an array), CafeMenu (uses `Hashtable`). Each chef refuses to change. The waitress needs to iterate over **all** menus uniformly.

### Iterator

> **Iterator:** provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.

```java
public interface Iterator<T> {
    boolean hasNext();
    T next();
}

public class DinerMenuIterator implements Iterator<MenuItem> {
    MenuItem[] items;
    int position = 0;
    public DinerMenuIterator(MenuItem[] items) { this.items = items; }
    public boolean hasNext() {
        return position < items.length && items[position] != null;
    }
    public MenuItem next() { return items[position++]; }
}

public class PancakeHouseMenu {
    List<MenuItem> menuItems;
    public Iterator<MenuItem> createIterator() {
        return menuItems.iterator();
    }
}

public class Waitress {
    PancakeHouseMenu pancakeHouseMenu;
    DinerMenu dinerMenu;
    public Waitress(PancakeHouseMenu p, DinerMenu d) {
        pancakeHouseMenu = p; dinerMenu = d;
    }
    public void printMenu() {
        printMenu(pancakeHouseMenu.createIterator());
        printMenu(dinerMenu.createIterator());
    }
    private void printMenu(Iterator<MenuItem> iterator) {
        while (iterator.hasNext()) {
            MenuItem item = iterator.next();
            System.out.println(item.getName() + ", $" + item.getPrice());
        }
    }
}
```

### Principle introduced — Single Responsibility

> **A class should have only one reason to change.**

`PancakeHouseMenu` should manage its menu items, not also know how to iterate. **High cohesion** = one well-defined responsibility per class.

### Composite

> **Composite:** compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.

The cafe wants a **submenu** for desserts inside the diner menu. Iterating menus that contain submenus that contain menu items requires recursion. Composite lets a Menu *contain* both MenuItems and other Menus.

```java
public abstract class MenuComponent {
    public void add(MenuComponent component)    { throw new UnsupportedOperationException(); }
    public void remove(MenuComponent component) { throw new UnsupportedOperationException(); }
    public MenuComponent getChild(int i)        { throw new UnsupportedOperationException(); }
    public String getName()                     { throw new UnsupportedOperationException(); }
    public String getDescription()              { throw new UnsupportedOperationException(); }
    public double getPrice()                    { throw new UnsupportedOperationException(); }
    public boolean isVegetarian()               { throw new UnsupportedOperationException(); }
    public void print()                         { throw new UnsupportedOperationException(); }
}

public class MenuItem extends MenuComponent {
    String name, description;
    boolean vegetarian;
    double price;

    public MenuItem(String name, String description, boolean vegetarian, double price) {
        this.name = name; this.description = description;
        this.vegetarian = vegetarian; this.price = price;
    }
    public String getName()        { return name; }
    public String getDescription() { return description; }
    public double getPrice()       { return price; }
    public boolean isVegetarian()  { return vegetarian; }

    public void print() {
        System.out.print("  " + getName());
        if (isVegetarian()) System.out.print("(v)");
        System.out.println(", " + getPrice() + " -- " + getDescription());
    }
}

public class Menu extends MenuComponent {
    List<MenuComponent> menuComponents = new ArrayList<>();
    String name, description;

    public Menu(String name, String description) {
        this.name = name; this.description = description;
    }

    public void add(MenuComponent component) { menuComponents.add(component); }
    public void remove(MenuComponent component) { menuComponents.remove(component); }
    public MenuComponent getChild(int i) { return menuComponents.get(i); }
    public String getName() { return name; }

    public void print() {
        System.out.println("\n" + getName() + ", " + description);
        System.out.println("---------------------");
        for (MenuComponent c : menuComponents) c.print();   // recurses transparently
    }
}
```

### Key points

* Iterator decouples collection storage from traversal.
* Composite represents **part-whole hierarchies** — leaves and composites share an interface.
* The classic Composite "violation" of SRP: a single interface for both leaf and composite *intentionally* mixes responsibilities — the book defends it as a **transparency vs. safety** trade-off.

### Pitfalls

* Iterator's biggest risk: **fail-fast** behavior when the collection is modified mid-iteration (`ConcurrentModificationException`).
* Composite makes leaves *look* like they have child operations. Throwing `UnsupportedOperationException` for leaves is one approach; checking type is another. The book accepts the trade-off.

> **C# parallel:** `IEnumerable<T>` / `IEnumerator<T>` — every `foreach` is the Iterator pattern; Roslyn syntax trees are a Composite; Blazor render trees.

---

## 13. Chapter 10 — State Pattern

> *"The state of things."*

### The story — Gumball Machine

A gumball machine has four states: **NoQuarter**, **HasQuarter**, **Sold**, **SoldOut**. Each state allows or rejects four actions: `insertQuarter`, `ejectQuarter`, `turnCrank`, `dispense`.

The naive design uses a giant state field + `switch` everywhere. Adding a **WinnerState** (10% chance of two gumballs) requires editing every action method in the machine. **Open/Closed is violated.**

### The pattern

> **State:** allow an object to alter its behavior when its internal state changes. The object will appear to change its class.

Each state becomes a **class implementing a `State` interface**. The machine delegates actions to its current state object.

### The canonical code

```java
public interface State {
    void insertQuarter();
    void ejectQuarter();
    void turnCrank();
    void dispense();
}

public class GumballMachine {
    State soldOutState;
    State noQuarterState;
    State hasQuarterState;
    State soldState;
    State winnerState;          // new state added — no other state class changes

    State state;
    int count = 0;

    public GumballMachine(int numberGumballs) {
        soldOutState    = new SoldOutState(this);
        noQuarterState  = new NoQuarterState(this);
        hasQuarterState = new HasQuarterState(this);
        soldState       = new SoldState(this);
        winnerState     = new WinnerState(this);
        this.count = numberGumballs;
        state = numberGumballs > 0 ? noQuarterState : soldOutState;
    }

    public void insertQuarter() { state.insertQuarter(); }
    public void ejectQuarter()  { state.ejectQuarter(); }
    public void turnCrank()     { state.turnCrank(); state.dispense(); }
    void setState(State state)  { this.state = state; }

    public State getNoQuarterState()  { return noQuarterState; }
    public State getHasQuarterState() { return hasQuarterState; }
    public State getSoldState()       { return soldState; }
    public State getSoldOutState()    { return soldOutState; }
    public State getWinnerState()     { return winnerState; }
    public int getCount()             { return count; }
    public void releaseBall() {
        if (count > 0) count--;
    }
}

public class NoQuarterState implements State {
    GumballMachine gumballMachine;
    public NoQuarterState(GumballMachine gm) { this.gumballMachine = gm; }

    public void insertQuarter() {
        System.out.println("You inserted a quarter");
        gumballMachine.setState(gumballMachine.getHasQuarterState());
    }
    public void ejectQuarter() { System.out.println("You haven't inserted a quarter"); }
    public void turnCrank()    { System.out.println("You turned, but there's no quarter"); }
    public void dispense()     { System.out.println("You need to pay first"); }
}

public class HasQuarterState implements State {
    GumballMachine gumballMachine;
    Random randomWinner = new Random(System.currentTimeMillis());

    public HasQuarterState(GumballMachine gm) { this.gumballMachine = gm; }

    public void insertQuarter() { System.out.println("You can't insert another quarter"); }
    public void ejectQuarter() {
        System.out.println("Quarter returned");
        gumballMachine.setState(gumballMachine.getNoQuarterState());
    }
    public void turnCrank() {
        System.out.println("You turned...");
        int winner = randomWinner.nextInt(10);
        if (winner == 0 && gumballMachine.getCount() > 1) {
            gumballMachine.setState(gumballMachine.getWinnerState());
        } else {
            gumballMachine.setState(gumballMachine.getSoldState());
        }
    }
    public void dispense() { System.out.println("No gumball dispensed"); }
}
```

### State vs. Strategy

> **Strategy:** the *client* picks the algorithm.
> **State:** the *object itself* transitions through states.

Structurally identical (same UML); semantically different.

### Key points

* Each state encapsulates **state-specific behavior**.
* Adding a new state ≠ editing existing states — Open/Closed holds.
* The machine becomes a *context*; state classes do the real work.
* States can transition the context to another state (`setState`).

### Pitfalls

* State explosion if the FSM has hundreds of states — at that point, use a real state-machine library (Stateless in .NET, XState in JS).
* States holding references to the context create *circular references* — fine in managed runtimes, watch in native code.

> **C# parallel:** the [Stateless](https://github.com/dotnet-state-machine/stateless) library; workflow engines (Workflow Core, Elsa).

---

## 14. Chapter 11 — Proxy Pattern

> *"Controlling object access."*

### The story — Monitoring the Gumball Machine, then loading CD images

Two examples in one chapter:

**Remote Proxy:** monitor the gumball machine **from another JVM** using Java RMI. A `GumballMachineProxy` looks like a local `GumballMachine` but actually forwards method calls over the network.

**Virtual Proxy:** display CD cover images, where each cover takes *seconds* to download. The proxy shows a "loading" message while the image downloads in the background; once loaded, the proxy delegates `paintIcon` to the real image.

### The pattern

> **Proxy:** provide a surrogate or placeholder for another object to control access to it.

### Two flavors covered

#### Remote Proxy (Java RMI)

```java
// The remote interface
public interface GumballMachineRemote extends Remote {
    int getCount()      throws RemoteException;
    String getLocation() throws RemoteException;
    State getState()    throws RemoteException;
}

// Implementation registered with RMI registry
public class GumballMachine extends UnicastRemoteObject implements GumballMachineRemote {
    // ... usual state machine code ...
}

// Client: looks up the proxy in the registry
GumballMachineRemote machine = (GumballMachineRemote)
    Naming.lookup("rmi://Boulder.mightygumball.com/gumballmachine");
int count = machine.getCount();   // network call hidden behind a method call
```

#### Virtual Proxy (lazy loading)

```java
public class ImageProxy implements Icon {
    ImageIcon imageIcon;
    URL imageURL;
    Thread retrievalThread;
    boolean retrieving = false;

    public ImageProxy(URL url) { imageURL = url; }

    public int getIconWidth()  { return imageIcon != null ? imageIcon.getIconWidth()  : 800; }
    public int getIconHeight() { return imageIcon != null ? imageIcon.getIconHeight() : 600; }

    public synchronized void paintIcon(final Component c, Graphics g, int x, int y) {
        if (imageIcon != null) {
            imageIcon.paintIcon(c, g, x, y);
        } else {
            g.drawString("Loading CD cover, please wait...", x + 300, y + 190);
            if (!retrieving) {
                retrieving = true;
                retrievalThread = new Thread(() -> {
                    try {
                        imageIcon = new ImageIcon(imageURL, "CD Cover");
                        c.repaint();
                    } catch (Exception e) { e.printStackTrace(); }
                });
                retrievalThread.start();
            }
        }
    }
}
```

#### Other proxy variants (mentioned)

| Variant            | What it controls                                                |
| ------------------ | --------------------------------------------------------------- |
| **Remote Proxy**   | A representative of a remote object.                            |
| **Virtual Proxy** | Creates an expensive object on demand.                          |
| **Protection Proxy** | Access based on caller credentials.                          |
| **Caching Proxy** | Caches results of expensive operations.                         |
| **Firewall Proxy** | Network protection.                                             |
| **Smart Reference** | Adds bookkeeping (e.g., reference count).                      |
| **Synchronization Proxy** | Safe access from multiple threads.                        |
| **Copy-On-Write Proxy** | Defers copies until the object is modified.                 |

The book also discusses **Java's Dynamic Proxy** (`java.lang.reflect.Proxy`), which lets you generate proxies at runtime.

### Proxy vs. Decorator

Mechanically identical — both wrap an object with the same interface. **Intent** differs:

* **Decorator** *adds behavior* the client wants.
* **Proxy** *controls access* (often without the client knowing).

### Key points

* Proxy is a chameleon — many specific subtypes, one structural shape.
* Modern alternatives: code generators (gRPC client stubs, Refit, Castle DynamicProxy) automate the proxy code.
* The proxy can be **smarter than its target** (caching, lazy loading, security).

### Pitfalls

* Hidden network call latency surprises callers — be explicit when method calls cross machines.
* Caching proxies need invalidation strategies.

> **C# parallel:** EF Core lazy-loading proxies (`UseLazyLoadingProxies()`); gRPC / Refit clients; Castle DynamicProxy (Moq, NSubstitute, AutoFac interceptors).

---

## 15. Chapter 12 — Compound Patterns (Model-View-Controller)

> *"Patterns of patterns."*

The book demonstrates that **patterns combine**. The motivating example is a **Duck Simulator** that mixes Strategy, Adapter, Decorator, Observer, Composite, Iterator, and Factory in one program. The chapter then dissects the most famous compound pattern of all: **MVC**.

### MVC = Strategy + Composite + Observer

```
   ┌──────────────┐     observes/updates    ┌──────────────┐
   │     View     │◄────────────────────────│    Model     │
   │ (Composite)  │       (Observer)        │              │
   └──────┬───────┘                          └──────▲───────┘
          │ requests state                         │
          │ delegates input                        │ state changes
          ▼                                        │
   ┌──────────────┐                                │
   │  Controller  │────────────────────────────────┘
   │  (Strategy)  │      changes state
   └──────────────┘
```

* **Model:** holds data + business logic. Implements **Subject** so it can notify Views when state changes (**Observer**).
* **View:** renders the Model. Composed of nested UI components (**Composite**). Subscribes to model changes (**Observer**).
* **Controller:** receives user input, decides what to do, mutates the Model. Different controller = different application behavior (**Strategy** — swap the controller, keep the view).

### Example — DJ Beat Machine (from the book)

```java
public class BeatModel implements BeatModelInterface, MetaEventListener {
    Sequencer sequencer;
    List<BeatObserver> beatObservers = new ArrayList<>();
    List<BPMObserver>  bpmObservers  = new ArrayList<>();
    int bpm = 90;

    public void initialize() { /* set up sequencer */ }
    public void on()  { sequencer.start(); notifyBPMObservers(); }
    public void off() { stopBeat(); sequencer.stop(); }
    public void setBPM(int bpm) { this.bpm = bpm; notifyBPMObservers(); }

    public void registerObserver(BeatObserver o) { beatObservers.add(o); }
    public void notifyBeatObservers() { for (BeatObserver o : beatObservers) o.updateBeat(); }
    // ...
}

public class DJView implements ActionListener, BeatObserver, BPMObserver {
    BeatModelInterface model;
    ControllerInterface controller;
    JLabel bpmOutputLabel;
    BeatBar beatBar;
    JButton increaseBPMButton, decreaseBPMButton;

    public DJView(ControllerInterface controller, BeatModelInterface model) {
        this.controller = controller;
        this.model      = model;
        model.registerObserver((BeatObserver) this);
        model.registerObserver((BPMObserver) this);
    }
    public void updateBPM()  { /* update the label */ }
    public void updateBeat() { beatBar.setValue(100); }   // pulse the UI

    // ActionListener — translate UI events to controller calls
    public void actionPerformed(ActionEvent e) {
        if (e.getSource() == increaseBPMButton) controller.increaseBPM();
        else if (e.getSource() == decreaseBPMButton) controller.decreaseBPM();
    }
}

public class BeatController implements ControllerInterface {
    BeatModelInterface model;
    DJView view;
    public BeatController(BeatModelInterface model) {
        this.model = model;
        view = new DJView(this, model);
        model.initialize();
    }
    public void start() { model.on(); }
    public void stop()  { model.off(); }
    public void increaseBPM() { model.setBPM(model.getBPM() + 1); }
    public void decreaseBPM() { model.setBPM(model.getBPM() - 1); }
}
```

### Key points

* **MVC is not one pattern — it's three working together.**
* The View and Model communicate via Observer, decoupling them.
* The Controller is a Strategy — swap it to repurpose the same view+model for a different application.
* The View itself is a Composite (panels containing panels).

### Pitfalls

* "Fat controller" is a smell — controllers should orchestrate, not contain business rules.
* In the web world, the *controller* often becomes a router or thin endpoint; the heavy lifting moves to use-case handlers (e.g., MediatR).
* The 2nd edition discusses the variants — MVP, MVVM, MV* — and emphasizes that **the principles** matter more than the letter combinations.

> **C# parallel:** ASP.NET Core MVC; Blazor's component model (Composite + Observer); WPF/MAUI (MVVM = a variant focused on data binding).

---

## 16. Chapter 13 — Better Living with Patterns

> *"Patterns in the real world."*

This chapter is not new patterns — it is **how to live with the ones you've learned**.

### The official pattern definition

> *"A pattern is a solution to a problem in a context."*
> — Christopher Alexander (architecture, then adopted by software)

A pattern has four essential parts:
* **Name** — gives developers a shared vocabulary.
* **Problem** — when to use this pattern.
* **Solution** — the structure.
* **Consequences** — trade-offs and forces.

### The pattern catalog (canonical classification)

| Creational                                          | Structural                                                                      | Behavioral                                                                                                                          |
| --------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Singleton, Factory Method, Abstract Factory, Builder, Prototype | Adapter, Facade, Decorator, Composite, Bridge, Flyweight, Proxy | Strategy, Observer, Command, Template Method, Iterator, State, Chain of Responsibility, Mediator, Memento, Interpreter, Visitor |

### Anti-patterns

The book briefly defines:

> *"An anti-pattern is a solution to a problem that has the wrong consequences."*

Examples:
* **Golden Hammer** — using one pattern for everything.
* **Stovepipe System** — re-inventing every time.
* **Lava Flow** — abandoned code that congeals into "load-bearing legacy".

### How to develop pattern thinking

1. Learn the patterns by their **canonical example** (this book's whole job).
2. Recognize the patterns *being applied around you* in frameworks (JDK, .NET BCL, ASP.NET, Spring).
3. Pattern-name your designs in code reviews — shared vocabulary speeds discussions.
4. Resist **patternitis** — patterns are not goals; *solving problems* is the goal.
5. Refactor *to* a pattern when pain appears, not *toward* it speculatively.

---

## 17. Appendix — Leftover Patterns

The book covers, briefly, the GoF patterns it didn't make full chapters for. Each gets a one-page treatment.

### Bridge

> Decouple an abstraction from its implementation so that the two can vary independently.

Example: **Remote Control + TV brand**. The remote (abstraction) and the TV (implementation) evolve separately.

### Builder

> Encapsulate the construction of a product and allow it to be constructed in steps.

Example: **Vacation planner** — different vacations built step by step.

### Chain of Responsibility

> Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request.

Example: **Email spam filter chain** — each handler decides to handle or pass.

### Flyweight

> Use sharing to support large numbers of fine-grained objects efficiently.

Example: **Trees in a yard simulation** — one `Tree` flyweight per type, many positions.

### Interpreter

> Build an interpreter for a language.

Example: **A duck simulator scripting language** — `quack;quack;move` parsed and evaluated.

### Mediator

> Centralize complex communications and control between related objects.

Example: **Smart home mediator** — devices talk *through* a hub, never directly.

### Memento

> Capture and externalize an object's internal state without violating encapsulation, so the object can be restored to this state later.

Example: **Save game state** — snapshot the game; restore on load.

### Prototype

> Create new objects by copying an existing object.

Example: **Monster spawner** in a game — clone a Monster prototype rather than rebuild.

### Visitor

> Add new operations to a structure of objects without modifying the classes.

Example: **Menu nutritional analyzer** — walk the Composite menu structure with a Visitor that totals calories.

---

## 18. Big-Picture Take-Aways

The book's deepest lessons are **not** the individual patterns — they are:

1. **Patterns codify principles.** If you internalize the 9 OO design principles, the patterns become discoverable, not memorized.
2. **Encapsulate what varies.** This single idea drives Strategy, Decorator, Factory, State, and Bridge.
3. **Favor composition over inheritance.** Inheritance is brittle; composition is swappable.
4. **Program to an interface, not an implementation.** This is the foundation of testability, mockability, and flexibility.
5. **The Hollywood Principle** — frameworks call your code, not the other way around. This is **inversion of control** at the design level and the foundation of modern DI containers.
6. **Patterns are vocabulary, not goals.** Their value is shared language and shared trade-offs, *not* their existence in your code.
7. **Resist over-engineering.** Apply a pattern when a real second variation appears. Premature patterns are worse than duplicated code.

> **Bottom line:** the book is a 600+ page training program in *thinking like an OO designer*. Reading it once gives you the patterns; rereading it once a year keeps the *principles* warm.

---

## 19. Reading Order Recommendations

* **First time:** read the chapters in order. The principles accumulate; skipping breaks the build-up.
* **Reference mode:** after the first read, jump to any chapter when its pattern is relevant — every chapter is self-contained enough to skim.
* **Pair with this folder's docs:**
  * Pattern definitions and .NET samples: [`../dot-net/docs/design-pattern/`](../dot-net/docs/design-pattern/).
  * The 9 principles, expanded: [`../principles/`](../principles/).
  * The OO pillars the book assumes: [`../oop/`](../oop/).

A common path for engineers new to design patterns:
1. Read **OOP** ([`../oop/`](../oop/)).
2. Read **Principles** ([`../principles/`](../principles/)).
3. Read **Head First Design Patterns** (this book).
4. Skim the GoF original ([Gamma et al., *Design Patterns*, 1994](https://en.wikipedia.org/wiki/Design_Patterns)) for the canonical, drier formulation.
5. Apply to a **real** project and see which patterns earn their keep.

---

## 20. Compared to the Original GoF Book

|                            | Head First Design Patterns                                   | Gang of Four (1994)                                          |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Tone**                   | Conversational, illustrated, humorous                         | Academic, dense, formal                                      |
| **Examples**               | One memorable running example per chapter                     | Multiple short examples, more abstract                       |
| **Language**               | Java (Java 8+ in 2nd ed., with lambdas)                      | C++ and Smalltalk                                            |
| **Patterns covered (in depth)** | 14 + MVC + brief on rest                              | All 23 in equal depth                                        |
| **Principles**             | 9 OO principles introduced gradually                          | Two principles (programming to interface; favor composition) |
| **Best for**               | First-time learners; visual readers; teams sharing vocabulary | Reference; deep theoretical understanding                    |
| **Worst for**              | Quick reference (it's playful, not terse)                    | Beginners; people learning from examples                     |

The two books are **complementary**. The GoF book is the canonical reference; HFDP is the canonical *teacher*.

---

## 21. References

* Eric Freeman & Elisabeth Robson — *Head First Design Patterns* (2nd ed., O'Reilly, 2020).
* Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (Addison-Wesley, 1994). The GoF book.
* Joshua Bloch — *Effective Java* (3rd ed.) — Item 3 (Singleton via enum), Item 18 (Composition over Inheritance), and many others reinforce HFDP's lessons.
* Christopher Alexander — *A Pattern Language* (1977). The book that introduced the pattern-language concept to software via Alexander's earlier architectural work.
* Head First Design Patterns 2nd edition source code: [github.com/bethrobson/Head-First-Design-Patterns](https://github.com/bethrobson/Head-First-Design-Patterns).
* O'Reilly product page: [oreilly.com/library/view/head-first-design/9781492077992](https://www.oreilly.com/library/view/head-first-design/9781492077992/).
