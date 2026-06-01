# Head First Design Patterns — Tóm tắt

> **Eric Freeman & Elisabeth Robson** — *Head First Design Patterns: Building Extensible & Maintainable Object-Oriented Software*, 2nd edition (O'Reilly, 2020). Edition 1 năm 2004. Dựa trên Java, nhưng lesson translate sạch sang C#, Kotlin, TypeScript, và Swift.

> **Tại sao sách này quan trọng:** đây là giới thiệu thân thiện nhất, được trích dẫn nhiều nhất về pattern **Gang of Four**. Phong cách "Head First" — đối thoại, trực quan, lặp lại có chủ ý — biến định nghĩa pattern trừu tượng thành câu chuyện bạn nhớ nhiều năm sau. Mỗi chương được build quanh một running example cụ thể.

> **Tóm tắt này cover:** các design principle sách dạy xuyên suốt, mỗi chương có pattern (intent, running example, code chính, take-away, cạm bẫy), chương compound pattern (MVC), và appendix leftover pattern.

> 🇻🇳 Phiên bản tiếng Việt. English: [`head-first-design-patterns.md`](./head-first-design-patterns.md)

---

## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)

- **Cái gì** — Tóm tắt tour thân thiện của Freeman & Robson qua 14 pattern GoF mà Head First cover sâu (Strategy, Observer, Decorator, Factory, Singleton, Command, Adapter, Facade, Template Method, Iterator, Composite, State, Proxy, Compound/MVC) cộng 9 nguyên lý OO design sách tích luỹ chương-by-chương.
- **Tại sao** — Cho bạn pattern *cộng* running story làm chúng đáng nhớ; bridge gap giữa định nghĩa pattern trừu tượng (GoF) và nhận diện pattern trong code thật (codebase của bạn).
- **Khi nào** — Lần đầu học design pattern; trước phỏng vấn system-design; khi muốn ví dụ canonical để anchor argument code-review ("đây là Decorator, không phải Adapter — đây là lý do").
- **Ở đâu** — Pair với `dot-net/docs/design-pattern/` (implementation .NET cả 23), `principles/` (9 nguyên lý OO sâu), và `oop/` (trụ cột sách giả định).

---

## Mục lục

1. [Về cuốn sách](#1-về-cuốn-sách)
2. [Approach "Head First"](#2-approach-head-first)
3. [9 Nguyên lý OO Design cốt lõi](#3-9-nguyên-lý-oo-design-cốt-lõi)
4. [Chương 1 — Strategy Pattern](#4-chương-1--strategy-pattern)
5. [Chương 2 — Observer Pattern](#5-chương-2--observer-pattern)
6. [Chương 3 — Decorator Pattern](#6-chương-3--decorator-pattern)
7. [Chương 4 — Factory Method & Abstract Factory](#7-chương-4--factory-method--abstract-factory)
8. [Chương 5 — Singleton Pattern](#8-chương-5--singleton-pattern)
9. [Chương 6 — Command Pattern](#9-chương-6--command-pattern)
10. [Chương 7 — Adapter & Facade Patterns](#10-chương-7--adapter--facade-patterns)
11. [Chương 8 — Template Method Pattern](#11-chương-8--template-method-pattern)
12. [Chương 9 — Iterator & Composite Patterns](#12-chương-9--iterator--composite-patterns)
13. [Chương 10 — State Pattern](#13-chương-10--state-pattern)
14. [Chương 11 — Proxy Pattern](#14-chương-11--proxy-pattern)
15. [Chương 12 — Compound Patterns (Model-View-Controller)](#15-chương-12--compound-patterns-model-view-controller)
16. [Chương 13 — Sống chung tốt với Pattern](#16-chương-13--sống-chung-tốt-với-pattern)
17. [Appendix — Leftover Patterns](#17-appendix--leftover-patterns)
18. [Take-Away bức tranh lớn](#18-take-away-bức-tranh-lớn)
19. [Khuyến nghị thứ tự đọc](#19-khuyến-nghị-thứ-tự-đọc)
20. [So sánh với GoF gốc](#20-so-sánh-với-gof-gốc)
21. [Tham khảo](#21-tham-khảo)

---

## 1. Về cuốn sách

* **Tiêu đề:** *Head First Design Patterns: Building Extensible & Maintainable Object-Oriented Software*
* **Tác giả:** Eric Freeman & Elisabeth Robson (1st edition đồng tác giả với Bert Bates & Kathy Sierra).
* **NXB:** O'Reilly Media.
* **Edition:** 1st (2004), 2nd (2020 — refresh cho Java 8+ với lambda, cộng note mới về `Observable` deprecation của Java).
* **Độ dài:** ~672 trang, nhưng format không thông thường — nặng diagram, dialogue, và exercise.
* **Audience:** developer thoải mái với Java/OO cơ bản muốn *internalize* pattern GoF thay vì chỉ nhận diện.
* **License của ví dụ:** source code open và nổi tiếng được imitate khắp ngành.

Sách cover **14 pattern sâu** (Strategy, Observer, Decorator, Factory Method, Abstract Factory, Singleton, Command, Adapter, Facade, Template Method, Iterator, Composite, State, Proxy) cộng **MVC compound pattern** và tour ngắn các pattern GoF còn lại trong appendix.

---

## 2. Approach "Head First"

Sách là một thí nghiệm dạy học có chủ đích. Format dựa vào nguyên lý khoa học nhận thức để làm idea trừu tượng stick:

* **Tone đối thoại** — prose nói *với* bạn, không nói *về* bạn.
* **Visual & humor kỳ quặc** — não nhớ anomaly.
* **Story chạy xuyên chương** — SimUDuck, Starbuzz Coffee, Gumball Machine, Home Theater… mỗi chương có một ví dụ đơn lẻ, đáng nhớ.
* **Bullet Point (box tóm tắt)** — cuối mỗi chương, list ngắn distill lesson.
* **Sharpen Your Pencil exercise** — fill-in-the-blank UML, "đoán pattern", code-completion.
* **Code Up Close** — code có annotation chỉ rõ dòng nào implement phần nào của pattern.
* **There Are No Dumb Questions** sidebar — anticipate nghi ngờ của reader.
* **Pattern formality ở cuối** — chỉ *sau khi* ví dụ được build mới reveal intent, structure, consequence canonical của GoF.

**Thứ tự đọc quan trọng.** Mỗi chương introduce nguyên lý mà chương sau build lên. Đến Chương 6 (Command), bạn có vocabulary work của *encapsulate cái biến đổi*, *favor composition*, *program to an interface*, và *loose coupling*.

---

## 3. 9 Nguyên lý OO Design cốt lõi

Sách tích luỹ checklist các **Nguyên lý OO Design** khi đi. Đến cuối sách bạn collected:

| #   | Nguyên lý                                                                            | Giới thiệu trong              |
| --- | ------------------------------------------------------------------------------------ | ------------------------ |
| 1   | **Xác định các aspect của application biến đổi và tách chúng khỏi cái giữ nguyên.** | Chương 1 — Strategy      |
| 2   | **Program to an interface, không phải implementation.**                              | Chương 1 — Strategy      |
| 3   | **Ưu tiên composition over inheritance.**                                            | Chương 1 — Strategy      |
| 4   | **Strive cho thiết kế loose coupling giữa các object tương tác.**                    | Chương 2 — Observer      |
| 5   | **Class nên mở cho extension, nhưng đóng cho modification.** (Open-Closed)          | Chương 3 — Decorator     |
| 6   | **Phụ thuộc abstraction. Không phụ thuộc class cụ thể.** (Dependency Inversion)     | Chương 4 — Factory       |
| 7   | **Principle of Least Knowledge — chỉ nói chuyện với bạn ngay sát.** (Law of Demeter) | Chương 7 — Facade       |
| 8   | **Đừng call chúng tôi, chúng tôi sẽ call bạn.** (Hollywood Principle)                | Chương 8 — Template Method |
| 9   | **Một class nên có chỉ một lý do để thay đổi.** (Single Responsibility Principle)    | Chương 9 — Iterator      |

9 nguyên lý này là **xương sống** của sách. Gần như mọi pattern được present như "thực thi của nguyên lý X, Y, Z".

> **Mental model:** pattern là *nguyên lý được áp dụng*. Nếu bạn internalize 9 nguyên lý trên, bạn thường có thể tự reinvent pattern tương ứng theo nhu cầu.

---

## 4. Chương 1 — Strategy Pattern

> *"Welcome to Design Patterns."*

### Câu chuyện — SimUDuck

Bạn thừa kế một duck simulator. Ban đầu, mọi `Duck` thừa kế từ base class và có `quack()` và `swim()`. Để "thêm flying duck", một dev thêm `fly()` vào base class. Đột nhiên **rubber duck bay** và **decoy duck bay**. Inheritance đẩy behavior không mong muốn vào subclass.

Bài học: **xác định cái biến đổi, encapsulate nó, và inject qua composition.**

### Pattern

> **Strategy:** define một gia đình thuật toán, encapsulate mỗi cái, và làm chúng có thể đổi. Strategy cho phép thuật toán vary độc lập khỏi client dùng nó.

### Code canonical

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

    // Composition: behavior có thể swap runtime
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

// Usage — swap behavior runtime
Duck model = new ModelDuck();
model.performFly();                     // "I can't fly"
model.setFlyBehavior(new FlyRocketPowered());
model.performFly();                     // "Rocket-powered flying"
```

### Key point

* Cái biến đổi → behavior. Encapsulate đằng sau interface.
* Cái giữ → cấu trúc của `Duck` chính.
* **HAS-A** (composition) linh hoạt hơn **IS-A** (inheritance).
* Behavior là object hạng nhất bạn có thể pass quanh.

### Cạm bẫy

* Strategy ≠ static utility. Toàn bộ điểm là **swap được runtime**.
* Đừng pre-extract strategy cho variation bạn không có. Chờ behavior cụ thể thứ hai.

---

## 5. Chương 2 — Observer Pattern

> *"Giữ object của bạn được thông báo."*

### Câu chuyện — Weather Station

Một device collect nhiệt độ, độ ẩm, và áp suất. Nhiều "display" (điều kiện hiện tại, statistics, forecast) cần update bất cứ khi nào reading thay đổi. Không pattern, class device hardcode mọi display. Thêm display mới = sửa core.

### Pattern

> **Observer:** define dependency one-to-many giữa các object để khi một object thay đổi state, mọi dependent được notify và update tự động.

### Code canonical

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

### Key point

* **Loose coupling** — subject chỉ biết observer implement `Observer`.
* Subject và observer có thể add/remove runtime.
* Hai flavor: **push** (subject gửi data) và **pull** (observer query subject). Sách dùng push và thảo luận trade-off.
* Edition 2 thêm note rằng `java.util.Observable` của Java **deprecated từ Java 9** — viết subject interface riêng (như trên).

### Cạm bẫy

* **Thứ tự notification không đảm bảo.** Đừng dựa vào nó.
* **Memory leak**: nếu observer không bao giờ unregister, subject giữ chúng alive.
* Trong code multi-thread, observer list cần synchronization hoặc copy-on-write list.

> **C# parallel:** keyword `event`, `IObservable<T>`/`IObserver<T>` (Rx), MediatR `INotification`. Trong .NET hiện đại bạn hiếm khi roll subject interface của riêng mình.

---

## 6. Chương 3 — Decorator Pattern

> *"Trang trí object."*

### Câu chuyện — Starbuzz Coffee

Quán cà phê bán beverage — DarkRoast, Espresso, HouseBlend, Decaf — cộng condiment (Mocha, Soy, Whip, Steamed Milk). Thiết kế ngây thơ là class explosion: `DarkRoastWithMochaAndWhip`, `EspressoWithMochaAndSoy`, …

Mỗi condiment đổi giá; mỗi combo unique. Subclass mọi combination không thể.

### Pattern

> **Decorator:** gắn thêm trách nhiệm vào object động. Decorator cung cấp alternative linh hoạt cho subclassing để extend functionality.

### Nguyên lý giới thiệu — Open/Closed

> **Class nên mở cho extension, nhưng đóng cho modification.**

Bạn extend functionality bằng *wrap* object, không bao giờ sửa code gốc.

### Code canonical

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

// Usage — compose runtime
Beverage beverage = new Espresso();          // Espresso $1.99
beverage = new Mocha(beverage);              // Espresso, Mocha $2.19
beverage = new Mocha(beverage);              // Espresso, Mocha, Mocha $2.39
beverage = new Whip(beverage);               // Espresso, Mocha, Mocha, Whip $2.49

System.out.println(beverage.getDescription() + " $" + beverage.cost());
```

### Key point

* Decorator có **cùng supertype** với object nó decorate.
* Chúng **wrap** object — nhiều decorator stack sâu tuỳ ý.
* Behavior thêm **động** lúc construction.
* Library `java.io` của Java (`BufferedReader` wrap `FileReader` wrap `Reader`) là **ví dụ thực tế canonical**.

### Cạm bẫy

* **Decorator thêm nhiều class nhỏ** — nhận ra API I/O là cái giá bạn trả cho flexibility compositional.
* Type của object được wrap không được preserve — dựa vào `instanceof` sau wrap là vỡ.
* Tránh decorator thêm method *mới* vào interface; chúng không visible qua base type.

> **C# parallel:** ASP.NET Core middleware *chính là* Decorator pattern quanh `HttpContext`; `DelegatingHandler` trong `HttpClient`; helper `Decorate<T>()` của Scrutor.

---

## 7. Chương 4 — Factory Method & Abstract Factory

> *"Bake với goodness OO."*

### Câu chuyện — Pizza Store

Một pizza store có `orderPizza(String type)`. Mỗi block `if (type.equals("cheese")) new CheesePizza()` grow khi bạn thêm topping. Tệ hơn, bạn franchise store: **New York** và **Chicago** style cần nguyên liệu khác nhưng cùng workflow.

### Hai pattern cover

#### Factory Method

> Define interface để tạo object, nhưng cho subclass quyết định class nào instantiate.

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
    // Factory method — defer cho subclass
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

> Cung cấp interface để tạo **gia đình** object liên quan hoặc phụ thuộc nhau mà không chỉ định class cụ thể.

Khi pizza vùng vary không chỉ ở *Pizza class nào* mà ở **dough, sauce, cheese, veggies nào** dùng:

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

### Nguyên lý giới thiệu — Dependency Inversion

> **Phụ thuộc abstraction. Không phụ thuộc class cụ thể.**

Sách list 3 guideline (không phải luật):

* Không biến nào nên giữ reference tới class cụ thể.
* Không class nào nên derive từ class cụ thể.
* Không method nào nên override method đã implement của base class.

Đây là *guideline*. Strive theo càng nhiều càng thực tế.

### Key point

* **Factory Method** dùng **subclassing** để defer choice của type cụ thể.
* **Abstract Factory** dùng **composition** để deliver *gia đình* type liên quan.
* Thường pair: Factory Method **bên trong** Abstract Factory.
* Cả hai **encapsulate object creation** — nguồn phổ biến nhất của code vi phạm OCP.

### Cạm bẫy

* **Simple Factory** (static factory method switch theo type) **không phải** GoF Factory Method — sách gọi đó là "programming idiom", không phải pattern thật.
* Đừng thêm factory cho mọi `new` call. Thêm khi có *lựa chọn thật* giữa type cụ thể hoặc khi construction *phức tạp*.

---

## 8. Chương 5 — Singleton Pattern

> *"Object one-of-a-kind."*

### Câu chuyện — Chocolate Boiler

Nhà máy chocolate có một `ChocolateBoiler` đắt. Nếu 2 thread mỗi cái tạo một và start cùng lúc, nhà máy flood. Chỉ **một instance** được phép tồn tại.

### Pattern

> **Singleton:** đảm bảo class chỉ có một instance, và cung cấp single global access point tới nó.

### Tiến hoá implementation

**Classic (KHÔNG thread-safe):**

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

**Synchronized (an toàn, chậm):**

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

**Double-checked locking (Java 5+, cần `volatile`):**

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

**Enum singleton (khuyến nghị trong 2nd ed., theo Effective Java):**

```java
public enum Singleton {
    UNIQUE_INSTANCE;

    public void doSomething() { /* ... */ }
}
// Usage:
Singleton.UNIQUE_INSTANCE.doSomething();
```

### Key point

* Singleton chống hai kẻ thù: **multiple instance** và **race concurrency**.
* Form enum là form đúng *đơn giản nhất* trong Java; nó handle serialization và reflection attack miễn phí.
* Edition 2 explicit cảnh báo: **Singleton thường bị lạm dụng.** Nó ẩn global state, làm test khó, và tạo coupling chặt.

### Cạm bẫy

* **Singleton làm test khổ sở** — không thể substitute fake.
* **Singleton ẩn dependency.** Class hỏi `MyService.getInstance()` lie về needs của nó.
* **Classloader / process boundary** có thể tạo *nhiều* "singleton" trong scenario JEE / multi-process.

> **Alternative hiện đại:** dùng **singleton lifetime** của DI container. Bạn nhận một instance per application, full testability, và dependency rõ ràng. **Singleton pattern as written** hiếm khi là answer đúng trong code .NET / Spring / NestJS 2025+.

---

## 9. Chương 6 — Command Pattern

> *"Encapsulate invocation."*

### Câu chuyện — Universal Remote Control

Remote 7 nút: mỗi nút có thể program để control device khác (light, fan, garage door, stereo). Remote không được biết API của mỗi device. **Undo** cũng required.

### Pattern

> **Command:** encapsulate request thành object, vì thế cho phép parameterize client với request khác nhau, queue hoặc log request, và support operation undoable.

### Code canonical

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

public class NoCommand implements Command {        // Null Object — tránh null check
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

### Key point

* Command **encapsulate** receiver + action + parameter.
* Invoker (remote control) không biết command làm gì.
* **Macro command** là composition của command — pattern đệ quy.
* **Null Object** (`NoCommand`) tránh `null` check khắp nơi.
* Command enable **queue**, **log**, và **undo** bằng treat action như data.

### Cạm bẫy

* Cho call single-action trivial, command object là overkill. Dùng lambda thay vào.
* `undo()` khó hơn `execute()` — nhiều operation không thực sự reversible.
* Trong code hiện đại, **lambda** là command không có ceremony.

> **C# parallel:** `IRequest<T>` + `IRequestHandler` của MediatR; command CQRS; delegate `Action`/`Func`.

---

## 10. Chương 7 — Adapter & Facade Patterns

> *"Adaptive."*

Hai pattern share chương vì cả hai đều **wrap** thứ gì đó — nhưng *intent* khác.

### Adapter

> **Adapter:** convert interface của class thành interface client mong đợi. Adapter cho phép class work cùng nhau mà không thể vì interface không tương thích.

#### Running example — Duck và Turkey

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
        for (int i = 0; i < 5; i++) turkey.fly();  // turkey bay ít hơn duck
    }
}

// Usage
Turkey turkey = new WildTurkey();
Duck duckAdapter = new TurkeyAdapter(turkey);
duckAdapter.quack();  // gobble
duckAdapter.fly();    // bay 5 lần
```

#### Hai flavor của Adapter

* **Object Adapter** (composition — phổ biến nhất, ví dụ trên).
* **Class Adapter** (multiple inheritance — không khả thi trong Java/C#, included cho completeness).

### Facade

> **Facade:** cung cấp interface thống nhất tới một bộ interface trong subsystem. Facade define interface high-level làm subsystem dễ dùng hơn.

#### Running example — Home Theater

Để xem phim, phải bật projector, dim đèn, hạ màn, fire amplifier, set surround sound, start streaming player, và pop popcorn — 7 device, theo thứ tự.

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

### Nguyên lý giới thiệu — Principle of Least Knowledge

> **Chỉ nói chuyện với bạn ngay sát.**

Method chỉ nên invoke method thuộc về:
* bản thân object
* object pass vào như parameter
* object method tạo hoặc instantiate
* component của object (field)

Không chain như `getA().getB().doSomething()` ("train wreck").

### Key point

* **Adapter** *thay* interface để match cái client mong đợi.
* **Facade** *đơn giản hoá* interface tới subsystem; nó không transform — nó consolidate.
* Cả hai wrap, nhưng **intent** khác (adapt vs. simplify).
* Facade giảm coupling (Law of Demeter) — client nói chuyện chỉ với facade.

### Cạm bẫy

* Facade nên **dễ bypass**. Nếu client *phải* dùng facade, bạn có controller, không facade.
* Adapter try làm quá nhiều logic thành **God Adapter**. Giữ chúng mỏng.

---

## 11. Chương 8 — Template Method Pattern

> *"Encapsulate thuật toán."*

### Câu chuyện — Caffeine Beverage

Starbuzz prepare Coffee và Tea dùng recipe gần giống:

* **Coffee:** boil water → brew coffee grind → pour in cup → add sugar và milk.
* **Tea:** boil water → steep tea bag → pour in cup → add lemon.

**Cấu trúc** giống nhau: boil → brew/steep → pour → condiment. Chỉ step 2 và 4 khác.

### Pattern

> **Template Method:** define skeleton của thuật toán trong một operation, defer một số step cho subclass. Template Method cho subclass redefine step cụ thể của thuật toán mà không thay đổi cấu trúc thuật toán.

### Code canonical

```java
public abstract class CaffeineBeverage {
    // Template method — final để subclass không phá thuật toán
    public final void prepareRecipe() {
        boilWater();
        brew();
        pourInCup();
        addCondiments();
    }

    abstract void brew();           // step subclass
    abstract void addCondiments();  // step subclass

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

### Hook tuỳ chọn

**Hook** là method declared trong abstract class với default (thường empty), subclass *có thể* override để influence thuật toán:

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

    // Hook với default
    boolean customerWantsCondiments() { return true; }
}
```

### Nguyên lý giới thiệu — Hollywood Principle

> **Đừng call chúng tôi, chúng tôi sẽ call bạn.**

Base class control thuật toán; subclass *plug in* step mà không bao giờ call lên chain. Đây là **inversion of control** ở mức class-design — cùng idea power framework như Spring, ASP.NET Core, và Angular.

### Key point

* Template Method = inheritance cho *cấu trúc cố định với step biến đổi*.
* Subclass fill `abstract` step và *tuỳ chọn* override hook.
* Mark template method `final` để ngăn subclass phá skeleton.
* Hollywood Principle generalize sang **framework**: framework call code của bạn.

### Cạm bẫy

* Inheritance couple subclass với base class. Nếu base thay đổi thuật toán, mọi subclass bị ảnh hưởng.
* Cho flexibility runtime, **Strategy** (composition) thường thắng Template Method (inheritance).
* Đừng đặt quá nhiều trong template — chừa chỗ cho intelligence subclass.

> **C# parallel:** `BackgroundService.ExecuteAsync` là template method; hook lifecycle `WebApplicationBuilder`; filter controller ASP.NET Core.

---

## 12. Chương 9 — Iterator & Composite Patterns

> *"Collection well-managed."*

### Câu chuyện — Merge Menu

Một diner-coffeehouse mới merge ba menu: PancakeHouse (dùng `ArrayList`), DinerMenu (dùng array), CafeMenu (dùng `Hashtable`). Mỗi chef từ chối thay đổi. Waitress cần iterate qua **mọi** menu thống nhất.

### Iterator

> **Iterator:** cung cấp cách truy cập element của aggregate object tuần tự không expose representation bên dưới.

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

### Nguyên lý giới thiệu — Single Responsibility

> **Class nên có chỉ một lý do để thay đổi.**

`PancakeHouseMenu` nên quản lý menu item của nó, không cũng biết cách iterate. **Cohesion cao** = một trách nhiệm well-defined per class.

### Composite

> **Composite:** compose object thành cấu trúc cây để biểu diễn hierarchy part-whole. Composite cho client treat object đơn lẻ và composition đồng nhất.

Cafe muốn **submenu** cho dessert bên trong diner menu. Iterate menu chứa submenu chứa menu item cần đệ quy. Composite cho Menu *chứa* cả MenuItem và Menu khác.

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
        for (MenuComponent c : menuComponents) c.print();   // đệ quy trong suốt
    }
}
```

### Key point

* Iterator decouple storage collection khỏi traversal.
* Composite biểu diễn **hierarchy part-whole** — leaf và composite share interface.
* "Vi phạm" SRP của Composite kinh điển: interface đơn cho cả leaf và composite *có chủ đích* trộn trách nhiệm — sách defend như trade-off **transparency vs. safety**.

### Cạm bẫy

* Rủi ro lớn nhất của Iterator: behavior **fail-fast** khi collection bị modify giữa iteration (`ConcurrentModificationException`).
* Composite làm leaf *trông như* có operation child. Throw `UnsupportedOperationException` cho leaf là một cách; check type là cách khác. Sách chấp nhận trade-off.

> **C# parallel:** `IEnumerable<T>` / `IEnumerator<T>` — mỗi `foreach` là Iterator pattern; cây syntax Roslyn là Composite; cây render Blazor.

---

## 13. Chương 10 — State Pattern

> *"State của things."*

### Câu chuyện — Gumball Machine

Máy gumball có 4 state: **NoQuarter**, **HasQuarter**, **Sold**, **SoldOut**. Mỗi state cho phép hoặc từ chối 4 action: `insertQuarter`, `ejectQuarter`, `turnCrank`, `dispense`.

Thiết kế ngây thơ dùng field state khổng lồ + `switch` khắp nơi. Thêm **WinnerState** (10% chance 2 gumball) cần sửa mọi method action trong machine. **Open/Closed bị vi phạm.**

### Pattern

> **State:** cho object alter behavior khi state nội bộ thay đổi. Object sẽ trông như thay đổi class.

Mỗi state thành **class implement interface `State`**. Machine delegate action tới object state hiện tại.

### Code canonical

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
    State winnerState;          // state mới thêm — không class state khác thay đổi

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

> **Strategy:** *client* chọn thuật toán.
> **State:** *bản thân object* transition qua state.

Giống nhau về cấu trúc (cùng UML); khác về semantic.

### Key point

* Mỗi state encapsulate **behavior state-specific**.
* Thêm state mới ≠ sửa state hiện có — Open/Closed giữ.
* Machine thành *context*; class state làm việc thật.
* State có thể transition context sang state khác (`setState`).

### Cạm bẫy

* State explosion nếu FSM có hàng trăm state — tại đó, dùng library state-machine thật (Stateless trong .NET, XState trong JS).
* State giữ reference tới context tạo *circular reference* — OK trong managed runtime, watch trong native.

> **C# parallel:** library [Stateless](https://github.com/dotnet-state-machine/stateless); workflow engine (Workflow Core, Elsa).

---

## 14. Chương 11 — Proxy Pattern

> *"Control truy cập object."*

### Câu chuyện — Monitor Gumball Machine, sau đó load image CD

Hai ví dụ trong một chương:

**Remote Proxy:** monitor gumball machine **từ JVM khác** dùng Java RMI. `GumballMachineProxy` trông như `GumballMachine` local nhưng thực ra forward method call qua mạng.

**Virtual Proxy:** display image CD cover, mỗi cover tốn *vài giây* để download. Proxy show message "loading" trong khi image download background; khi loaded, proxy delegate `paintIcon` cho image thật.

### Pattern

> **Proxy:** cung cấp surrogate hoặc placeholder cho object khác để control truy cập tới nó.

### Hai flavor cover

#### Remote Proxy (Java RMI)

```java
// Interface remote
public interface GumballMachineRemote extends Remote {
    int getCount()      throws RemoteException;
    String getLocation() throws RemoteException;
    State getState()    throws RemoteException;
}

// Implementation register với RMI registry
public class GumballMachine extends UnicastRemoteObject implements GumballMachineRemote {
    // ... code state machine thông thường ...
}

// Client: lookup proxy trong registry
GumballMachineRemote machine = (GumballMachineRemote)
    Naming.lookup("rmi://Boulder.mightygumball.com/gumballmachine");
int count = machine.getCount();   // network call ẩn đằng sau method call
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

#### Các variant proxy khác (mentioned)

| Variant            | Cái nó control                                                  |
| ------------------ | --------------------------------------------------------------- |
| **Remote Proxy**   | Đại diện của object remote.                                     |
| **Virtual Proxy** | Tạo object đắt theo nhu cầu.                                    |
| **Protection Proxy** | Truy cập dựa trên credential caller.                          |
| **Caching Proxy** | Cache kết quả của operation đắt.                                |
| **Firewall Proxy** | Bảo vệ network.                                                 |
| **Smart Reference** | Thêm bookkeeping (vd., reference count).                       |
| **Synchronization Proxy** | Truy cập an toàn từ nhiều thread.                         |
| **Copy-On-Write Proxy** | Defer copy tới khi object bị modify.                       |

Sách cũng thảo luận **Java's Dynamic Proxy** (`java.lang.reflect.Proxy`), cho phép generate proxy runtime.

### Proxy vs. Decorator

Cơ học giống nhau — cả hai wrap object với cùng interface. **Intent** khác:

* **Decorator** *thêm behavior* mà client muốn.
* **Proxy** *control access* (thường client không biết).

### Key point

* Proxy là tắc kè — nhiều subtype cụ thể, một shape cấu trúc.
* Alternative hiện đại: code generator (gRPC client stub, Refit, Castle DynamicProxy) tự động code proxy.
* Proxy có thể **thông minh hơn target** (caching, lazy loading, security).

### Cạm bẫy

* Latency network call ẩn surprise caller — explicit khi method call cross machine.
* Caching proxy cần strategy invalidation.

> **C# parallel:** EF Core lazy-loading proxy (`UseLazyLoadingProxies()`); client gRPC / Refit; Castle DynamicProxy (Moq, NSubstitute, interceptor AutoFac).

---

## 15. Chương 12 — Compound Patterns (Model-View-Controller)

> *"Pattern của pattern."*

Sách demo rằng **pattern kết hợp**. Ví dụ motivating là **Duck Simulator** mix Strategy, Adapter, Decorator, Observer, Composite, Iterator, và Factory trong một program. Chương sau dissect compound pattern nổi tiếng nhất: **MVC**.

### MVC = Strategy + Composite + Observer

```
   ┌──────────────┐     observe/update      ┌──────────────┐
   │     View     │◄────────────────────────│    Model     │
   │ (Composite)  │       (Observer)        │              │
   └──────┬───────┘                          └──────▲───────┘
          │ request state                          │
          │ delegate input                         │ state change
          ▼                                        │
   ┌──────────────┐                                │
   │  Controller  │────────────────────────────────┘
   │  (Strategy)  │      change state
   └──────────────┘
```

* **Model:** giữ data + business logic. Implement **Subject** để notify View khi state thay đổi (**Observer**).
* **View:** render Model. Compose từ UI component nested (**Composite**). Subscribe model change (**Observer**).
* **Controller:** nhận user input, quyết định làm gì, mutate Model. Controller khác = behavior application khác (**Strategy** — swap controller, giữ view).

### Ví dụ — DJ Beat Machine (từ sách)

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
    public void updateBPM()  { /* update label */ }
    public void updateBeat() { beatBar.setValue(100); }   // pulse UI

    // ActionListener — dịch event UI sang call controller
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

### Key point

* **MVC không phải một pattern — là ba làm việc cùng nhau.**
* View và Model giao tiếp qua Observer, decouple chúng.
* Controller là Strategy — swap để repurpose cùng view+model cho application khác.
* View tự nó là Composite (panel chứa panel).

### Cạm bẫy

* "Fat controller" là smell — controller nên orchestrate, không chứa business rule.
* Trong web world, *controller* thường thành router hoặc endpoint mỏng; heavy lifting move sang use-case handler (vd., MediatR).
* Edition 2 thảo luận variant — MVP, MVVM, MV* — và emphasize **nguyên lý** quan trọng hơn combination letter.

> **C# parallel:** ASP.NET Core MVC; model component Blazor (Composite + Observer); WPF/MAUI (MVVM = variant tập trung vào data binding).

---

## 16. Chương 13 — Sống chung tốt với Pattern

> *"Pattern trong thế giới thực."*

Chương này không phải pattern mới — là **cách sống chung với cái bạn đã học**.

### Định nghĩa pattern chính thức

> *"Pattern là giải pháp cho vấn đề trong context."*
> — Christopher Alexander (kiến trúc, sau adopt bởi software)

Pattern có 4 phần thiết yếu:
* **Name** — cho developer vocabulary chung.
* **Problem** — khi nào dùng pattern này.
* **Solution** — cấu trúc.
* **Consequence** — trade-off và force.

### Catalog pattern (phân loại canonical)

| Creational                                          | Structural                                                                      | Behavioral                                                                                                                          |
| --------------------------------------------------- | ------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Singleton, Factory Method, Abstract Factory, Builder, Prototype | Adapter, Facade, Decorator, Composite, Bridge, Flyweight, Proxy | Strategy, Observer, Command, Template Method, Iterator, State, Chain of Responsibility, Mediator, Memento, Interpreter, Visitor |

### Anti-pattern

Sách briefly define:

> *"Anti-pattern là giải pháp cho vấn đề có consequence sai."*

Ví dụ:
* **Golden Hammer** — dùng một pattern cho mọi thứ.
* **Stovepipe System** — re-invent mỗi lần.
* **Lava Flow** — code bị bỏ rơi đông lại thành "legacy chịu lực".

### Cách phát triển tư duy pattern

1. Học pattern theo **ví dụ canonical** của chúng (job của sách này).
2. Nhận diện pattern *đang được áp dụng quanh bạn* trong framework (JDK, .NET BCL, ASP.NET, Spring).
3. Pattern-name thiết kế của bạn trong code review — vocabulary chung tăng tốc thảo luận.
4. Kháng cự **patternitis** — pattern không phải goal; *giải vấn đề* là goal.
5. Refactor *về* pattern khi pain xuất hiện, không *về* pattern speculative.

---

## 17. Appendix — Leftover Patterns

Sách cover, briefly, pattern GoF không có chương đầy đủ. Mỗi cái có một trang.

### Bridge

> Decouple abstraction khỏi implementation để hai cái vary độc lập.

Ví dụ: **Remote Control + TV brand**. Remote (abstraction) và TV (implementation) tiến hoá riêng.

### Builder

> Encapsulate construction của product và cho phép construct từng bước.

Ví dụ: **Vacation planner** — kỳ nghỉ khác nhau build từng bước.

### Chain of Responsibility

> Tránh couple sender của request với receiver bằng cho hơn một object cơ hội handle request.

Ví dụ: **Email spam filter chain** — mỗi handler quyết định handle hoặc pass.

### Flyweight

> Dùng sharing để support số lượng lớn object fine-grained hiệu quả.

Ví dụ: **Cây trong simulation sân** — một flyweight `Tree` per type, nhiều vị trí.

### Interpreter

> Build interpreter cho một ngôn ngữ.

Ví dụ: **Ngôn ngữ scripting duck simulator** — `quack;quack;move` parse và evaluate.

### Mediator

> Tập trung communication và control phức tạp giữa các object liên quan.

Ví dụ: **Smart home mediator** — device nói chuyện *qua* hub, không bao giờ trực tiếp.

### Memento

> Capture và externalize internal state của object không vi phạm encapsulation, để object có thể restore sau.

Ví dụ: **Save game state** — snapshot game; restore lúc load.

### Prototype

> Tạo object mới bằng copy object hiện có.

Ví dụ: **Monster spawner** trong game — clone prototype Monster thay vì rebuild.

### Visitor

> Thêm operation mới vào cấu trúc object không sửa class.

Ví dụ: **Menu nutritional analyzer** — walk cấu trúc Composite menu với Visitor total calorie.

---

## 18. Take-Away bức tranh lớn

Lesson sâu nhất của sách **không** phải pattern cá nhân — là:

1. **Pattern codify nguyên lý.** Nếu internalize 9 OO design principle, pattern thành có thể discover, không phải memorize.
2. **Encapsulate cái biến đổi.** Idea đơn lẻ này drive Strategy, Decorator, Factory, State, và Bridge.
3. **Ưu tiên composition over inheritance.** Inheritance brittle; composition swappable.
4. **Program to an interface, không phải implementation.** Đây là nền của testability, mockability, và flexibility.
5. **Hollywood Principle** — framework call code của bạn, không ngược lại. Đây là **inversion of control** ở mức design và nền của container DI hiện đại.
6. **Pattern là vocabulary, không phải goal.** Giá trị là ngôn ngữ chung và trade-off chung, *không* phải sự tồn tại trong code của bạn.
7. **Kháng cự over-engineering.** Áp dụng pattern khi variation thứ hai thật xuất hiện. Pattern premature tệ hơn code trùng lặp.

> **Bottom line:** sách là chương trình training 600+ trang trong *tư duy như OO designer*. Đọc một lần cho bạn pattern; đọc lại mỗi năm giữ *nguyên lý* ấm.

---

## 19. Khuyến nghị thứ tự đọc

* **Lần đầu:** đọc chương theo thứ tự. Nguyên lý tích luỹ; skip phá build-up.
* **Reference mode:** sau lần đọc đầu, nhảy tới chương bất kỳ khi pattern của nó liên quan — mỗi chương self-contained đủ để skim.
* **Pair với doc trong folder này:**
  * Định nghĩa pattern và sample .NET: [`../dot-net/docs/design-pattern/`](../dot-net/docs/design-pattern/).
  * 9 nguyên lý, expand: [`../principles/`](../principles/).
  * Trụ cột OO sách giả định: [`../oop/`](../oop/).

Path phổ biến cho engineer mới với design pattern:
1. Đọc **OOP** ([`../oop/`](../oop/)).
2. Đọc **Principles** ([`../principles/`](../principles/)).
3. Đọc **Head First Design Patterns** (sách này).
4. Skim GoF gốc ([Gamma et al., *Design Patterns*, 1994](https://en.wikipedia.org/wiki/Design_Patterns)) cho formulation canonical, khô hơn.
5. Áp dụng cho project **thật** và xem pattern nào kiếm ăn được.

---

## 20. So sánh với GoF gốc

|                            | Head First Design Patterns                                   | Gang of Four (1994)                                          |
| -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Tone**                   | Đối thoại, illustrated, humorous                              | Academic, dense, formal                                      |
| **Ví dụ**                  | Một running example đáng nhớ per chương                       | Nhiều ví dụ ngắn, trừu tượng hơn                              |
| **Ngôn ngữ**               | Java (Java 8+ trong 2nd ed., với lambda)                     | C++ và Smalltalk                                             |
| **Pattern cover (sâu)**    | 14 + MVC + brief các pattern còn lại                          | Cả 23 sâu như nhau                                            |
| **Nguyên lý**              | 9 OO principle introduce dần                                  | Hai nguyên lý (program to interface; favor composition)       |
| **Tốt nhất cho**           | Người học lần đầu; reader visual; team share vocabulary       | Reference; hiểu lý thuyết sâu                                 |
| **Tệ nhất cho**            | Reference nhanh (playful, không terse)                       | Người mới bắt đầu; người học từ ví dụ                          |

Hai sách **bổ sung**. Sách GoF là reference canonical; HFDP là **teacher** canonical.

---

## 21. Tham khảo

* Eric Freeman & Elisabeth Robson — *Head First Design Patterns* (2nd ed., O'Reilly, 2020).
* Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides — *Design Patterns: Elements of Reusable Object-Oriented Software* (Addison-Wesley, 1994). Sách GoF.
* Joshua Bloch — *Effective Java* (3rd ed.) — Item 3 (Singleton qua enum), Item 18 (Composition over Inheritance), và nhiều cái khác reinforce lesson HFDP.
* Christopher Alexander — *A Pattern Language* (1977). Sách giới thiệu khái niệm pattern-language vào software qua công trình kiến trúc earlier của Alexander.
* Source code Head First Design Patterns 2nd edition: [github.com/bethrobson/Head-First-Design-Patterns](https://github.com/bethrobson/Head-First-Design-Patterns).
* O'Reilly product page: [oreilly.com/library/view/head-first-design/9781492077992](https://www.oreilly.com/library/view/head-first-design/9781492077992/).
