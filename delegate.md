# 事件与委托

## 1. 委托（Delegate）

### 1.1 什么是委托？

委托是 **类型安全的函数指针**。它允许你将方法作为参数传递或存储，能够动态调用方法。委托本质上是对一个方法的引用，能让你在运行时决定调用哪个方法。 

### 1.2 委托的定义和使用

#### 定义委托

```csharp
public delegate void MyDelegate(string message);
```

`MyDelegate` 是一个委托类型，它定义了一个方法签名：接受一个 `string` 参数并返回 `void`。

#### 委托的实例化

你可以将委托绑定到任何匹配的函数或方法：

```csharp
MyDelegate del = new MyDelegate(ShowMessage);

void ShowMessage(string message)
{
    Console.WriteLine(message);
}
```

#### 通过委托调用方法

```csharp
del("Hello, Delegate!"); // 调用 ShowMessage 方法
```

### 1.3 委托的多播

委托允许多个方法绑定在同一个委托实例上，称为多播委托（Multicast Delegate）。委托会按照添加的顺序依次调用所有绑定的方法。

```csharp
MyDelegate del = ShowMessage;
del += ShowAnotherMessage;
del("Hello, Multiple Methods!"); 

void ShowAnotherMessage(string message) 
{ 
    Console.WriteLine("Another message: " + message);
}
```

### 1.4 匿名方法和 Lambda 表达式

你也可以通过匿名方法或 Lambda 表达式来定义委托：

```csharp
MyDelegate del = delegate(string message)
{ 
    Console.WriteLine(message);
};

MyDelegate del2 = (message) => Console.WriteLine(message);
```

* * *

## 2. 事件（Event）

### 2.1 什么是事件？

事件是基于委托的一种机制，它用于实现 **发布-订阅** 模式。事件将委托的使用限制为只能由事件的发布者触发，外部订阅者只能响应事件。事件通常用于通知订阅者某些状态或操作的发生。

### 2.2 事件的定义和使用

#### 定义事件

事件通常通过委托来定义：

```csharp
public delegate void NotifyEventHandler(string message); 
public event NotifyEventHandler OnNotify;
```

这里 `NotifyEventHandler` 是委托类型，`OnNotify` 是事件名。

#### 触发事件

事件的触发只能在发布者内部进行：

```csharp
OnNotify?.Invoke("Event triggered!");
```

#### 订阅事件

订阅者通过 `+=` 操作符来订阅事件：

```csharp
publisher.OnNotify += subscriber.HandleEvent;
```

#### 取消订阅

订阅者通过 `-=` 操作符来取消订阅事件：

```csharp
publisher.OnNotify -= subscriber.HandleEvent;
```

### 2.3 事件的示例

```csharp
using System;

public class Publisher
{
    // 定义委托类型
    public delegate void NotifyEventHandler(string message);

    // 定义事件
    public event NotifyEventHandler OnNotify;

    public void TriggerEvent()
    {
        // 触发事件
        OnNotify?.Invoke("Event triggered!");
    }
}

public class Subscriber
{
    public void HandleEvent(string message)
    {
        Console.WriteLine("Received event: " + message);
    }
}

public class EventExample
{
    public void TestEvent()
    {
        Publisher publisher = new Publisher();
        Subscriber subscriber = new Subscriber();

        // 订阅事件
        publisher.OnNotify += subscriber.HandleEvent;

        // 触发事件
        publisher.TriggerEvent();
    }
}
```

* * *

## 3. 委托与事件的关系

1. **委托是事件的基础**：事件是基于委托的，它使用委托来定义事件的签名，实际上事件就是对委托的一种封装。委托负责定义事件的调用方式，事件用于管理委托的注册和触发。

2. **事件限制委托的调用**：委托允许你直接调用方法，而事件限制了委托的调用，**只能由事件的发布者来触发**，订阅者只能响应事件而不能主动触发。这保证了事件的发布者能够控制何时触发事件，增强了封装性和安全性。

3. **事件的多播**：类似于委托，事件也可以有多个订阅者（多播）。当事件被触发时，所有订阅了该事件的方法都会按顺序执行。

4. **订阅和取消订阅**：委托本身不提供订阅和取消订阅的机制，而事件通过 `+=` 和 `-=` 来提供这样的功能，使得代码更加简洁和安全。

* * *

## 4. 委托与事件的实际应用

### 4.1 发布-订阅模式

事件和委托的常见应用场景是 **发布-订阅模式**，其中一个对象发布事件，其他对象订阅并响应这些事件。

### 4.2 用户界面编程

在 GUI 应用（如 WPF 或 WinForms）中，事件与委托常用于处理用户的输入行为，例如按钮点击、鼠标移动等。

### 4.3 异步操作和回调

事件和委托还广泛用于异步编程和回调操作。例如，当一个任务完成时，事件可以用来通知主线程更新界面或执行后续操作。

* * *

## 5. 总结

* **委托**：是一种类型，它表示对一个方法的引用，可以动态调用方法，支持多播。
* **事件**：是基于委托的，它封装了委托，并限制了委托的触发只能在发布者内部进行，避免外部代码直接触发事件。
* **事件与委托的关系**：事件依赖于委托，委托定义事件的签名，而事件通过 `+=` 和 `-=` 让订阅者和发布者之间解耦，简化了事件处理和订阅管理。
  
