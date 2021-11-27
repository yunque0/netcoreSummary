@[TOC](博文目录）

## 前言

[原文链接](https://stackoverflow.com/questions/38138100/addtransient-addscoped-and-addsingleton-services-differences)

\* 注：知识搬运，供学习交流使用，侵联删！	

最近在学习.net core,看到了.netcore中的依赖注入有三种生命周期，分别是Transient,Scoped和Singleton，其中Singleton这个生命周期比较容易理解，单例模式，创建后直到依赖注入容器被销毁才会被回收掉，但是Transient和scoped文档中就说的比较不容易理解了。

Transient lifetime services are created each time they are requested.

Scoped lifetime services are created once per request.

按文档中的描述，Transient和Scoped都是每一次request的时候创建一个新的对象，那他们的区别是什么呢，直到我找到上面这篇文档，这两个生命周期中的request的含义其实是不一样的，因为原文是英文的，我用中文又记录了一遍以供以后复习（基本是翻译的）。




___

## 正文

结论:

​	Transient对象总是不同的，每一个controller或者每一个以Transient对象的作为依赖的其他对象都会得到一个全新的transient对象。

​	Scoped对象在同一个web请求中是一样的，但是在不同的请求中是不同的。

​	Singleton对象对于每一个以Singleton对象的作为依赖的其他对象以及在每一次请求中都是相同的。

​	我们可以用一个简单的拥有唯一标识符OperationId的接口来进行实验。通过观察不同生命周期的对象在不同条件下的OperationId，可以更好地理解对象的生命周期。

​	首先，创建一个拥有有OperationId的简单接口，然后继承这个接口来创建之后会以不同生命周期创建的的接口

```c#
using System;

namespace DependencyInjectionSample.Interfaces
{
    public interface IOperation
    {
        Guid OperationId { get; }
    }

    public interface IOperationTransient : IOperation
    {
    }

    public interface IOperationScoped : IOperation
    {
    }

    public interface IOperationSingleton : IOperation
    {
    }

    public interface IOperationSingletonInstance : IOperation
    {
    }
}
```

然后实现一个继承了上面所有不同生命周期接口的类，这样之后就可以通过注入不同的接口和同一个类来实现不同生命周期对象的注入。

```c#
using System;
using DependencyInjectionSample.Interfaces;
namespace DependencyInjectionSample.Classes
{
    public class Operation : IOperationTransient, IOperationScoped, IOperationSingleton, IOperationSingletonInstance
    {
        Guid _guid;
        public Operation() : this(Guid.NewGuid())
        {

        }

        public Operation(Guid guid)
        {
            _guid = guid;
        }

        public Guid OperationId => _guid;
    }
}
```

然后在ConfigureService中把这些对象都注入进入

```c#
services.AddTransient<IOperationTransient, Operation>();
services.AddScoped<IOperationScoped, Operation>();
services.AddSingleton<IOperationSingleton, Operation>();
services.AddSingleton<IOperationSingletonInstance>(new Operation(Guid.Empty));
services.AddTransient<OperationService, OperationService>();
```

其中IOperationSingleton注入的是产生随机OperationId的对象，而IOperationSingletonInstance注入的是一个固定OperationId为Guid.Empty的对象。

最后，注入了一个实现了前面四种Operation接口的Operation service，以便后面观察在同一个request中Transient和Scoped对象的表现，实现如下:

```c#
using DependencyInjectionSample.Interfaces;

namespace DependencyInjectionSample.Services
{
    public class OperationService
    {
        public IOperationTransient TransientOperation { get; }
        public IOperationScoped ScopedOperation { get; }
        public IOperationSingleton SingletonOperation { get; }
        public IOperationSingletonInstance SingletonInstanceOperation { get; }

        public OperationService(IOperationTransient transientOperation,
            IOperationScoped scopedOperation,
            IOperationSingleton singletonOperation,
            IOperationSingletonInstance instanceOperation)
        {
            TransientOperation = transientOperation;
            ScopedOperation = scopedOperation;
            SingletonOperation = singletonOperation;
            SingletonInstanceOperation = instanceOperation;
        }
    }
}
```

最后在controller中将这些对象的opertionId打印出来。

```c#
using DependencyInjectionSample.Interfaces;
using DependencyInjectionSample.Services;
using Microsoft.AspNetCore.Mvc;

namespace DependencyInjectionSample.Controllers
{
    public class OperationsController : Controller
    {
        private readonly OperationService _operationService;
        private readonly IOperationTransient _transientOperation;
        private readonly IOperationScoped _scopedOperation;
        private readonly IOperationSingleton _singletonOperation;
        private readonly IOperationSingletonInstance _singletonInstanceOperation;

        public OperationsController(OperationService operationService,
            IOperationTransient transientOperation,
            IOperationScoped scopedOperation,
            IOperationSingleton singletonOperation,
            IOperationSingletonInstance singletonInstanceOperation)
        {
            _operationService = operationService;
            _transientOperation = transientOperation;
            _scopedOperation = scopedOperation;
            _singletonOperation = singletonOperation;
            _singletonInstanceOperation = singletonInstanceOperation;
        }

        public IActionResult Index()
        {
            // ViewBag contains controller-requested services
            ViewBag.Transient = _transientOperation;
            ViewBag.Scoped = _scopedOperation;
            ViewBag.Singleton = _singletonOperation;
            ViewBag.SingletonInstance = _singletonInstanceOperation;

            // Operation service has its own requested services
            ViewBag.Service = _operationService;
            return View();
        }
    }
}
```

在Request了两次之后，结果如下

![image-20211128000213079](C:\Users\rongchen\AppData\Roaming\Typora\typora-user-images\image-20211128000213079.png)

![image-20211128000234039](C:\Users\rongchen\AppData\Roaming\Typora\typora-user-images\image-20211128000234039.png)

通过观察OperationId，可以看出，Singleton的对象在不同的request里都是一样的，Scoped对象在同一个request里是一样的，而Transient对象即使在同一request里面，每次出现都是不一样的。


___

## 参考资料

* [微软文档](https://docs.microsoft.com/dotnet/core/extensions/dependency-injection-usage)
* [翻译原文](https://stackoverflow.com/questions/38138100/addtransient-addscoped-and-addsingleton-services-differences)

示例代码可以在仓库中的ToLearnnet文件夹中找到。