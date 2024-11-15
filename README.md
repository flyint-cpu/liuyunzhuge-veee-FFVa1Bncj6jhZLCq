**阅读目录**

* [前言](#_label0)
* [介绍](#_label1)
* [IHostedService 示例](#_label2):[milou加速器](https://jiechuangmoxing.com)
* [BackgroundService 示例](#_label3)
* [IHostedService 和 BackgroundService 区别](#_label4)
* [总结](#_label5)

## 前言


在.NET 8中，IHostedService 和 BackgroundService 两个核心接口的引入，增强了项目开发中处理定时任务的能力。这两个接口不仅简化了定时任务、后台处理作业以及定期维护任务的实现过程，还提升了在ASP.NET Core 或任何基于.NET的宿主应用程序中的集成与管理效率。


IHostedService接口提供了一个基本的框架，允许自定义后台服务的启动和停止逻辑。通过实现该接口，可以灵活地控制服务的生命周期，确保任务在应用程序启动时自动运行，并在应用程序关闭时结束。


而 BackgroundService 类则是对 IHostedService 接口的进一步封装，它专为需要长时间运行的任务而设计。


通过继承 BackgroundService并重写其 ExecuteAsync方法，可以轻松地实现复杂的后台逻辑，如循环执行的任务、基于时间间隔的操作等。这种设计模式让代码的可读性和可维护性变的更好。


利用这些功能，可以快速构建出高效、可靠的定时任务系统，用于执行诸如消息推送、数据更新、定时发布等关键业务操作。这些任务可以在不影响应用程序主流程的情况下独立运行，从而提高了整个系统的性能和稳定性。


## 介绍


.NET 中的后台服务允许在后台独立于主应用程序线程运行任务。这对于需要连续或定期运行而不阻塞主应用程序流的任务至关重要。


**IHostedService**


IHostedService 是一个简单的接口，用于实现后台服务。当需要实现自定义的托管服务时，可以通过实现这个接口来创建。


该接口定义了两个方法：StartAsync(CancellationToken cancellationToken) 和 StopAsync(CancellationToken cancellationToken)，分别用于启动和停止服务。


**BackgroundService**


BackgroundService 是一个抽象类，它继承自 IHostedService 并提供了更简洁的方式来编写后台服务。它通常用于需要长时间运行的任务，如监听器、工作队列处理器等。通过重写 ExecuteAsync(CancellationToken stoppingToken)方法，可以在其中编写任务的逻辑。ExecuteAsync方法会循环在后台执行，直到服务停止。


## IHostedService 示例


### 1、注册服务


Program.cs 中添加配置，.NET 5 及以下在需要在 Startup.cs 注册服务。




```
// .NET 8
using ManageCore.Api;
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddHostedService();
var app = builder.Build();
```


### 2、创建服务接口


创建一个类，该类继承 IHostedService 接口，并实现该接口成员.


在不需要定时执行任务的时候，也可以在这里进行应用启动后的操作，例如创建 RabbitMQ 连接




```
using Microsoft.Extensions.Hosting;

namespace ManageCore.Api
{
    public class DemoHostedService : IHostedService, IDisposable
    {
        private Timer? _timer;

        public Task StartAsync(CancellationToken cancellationToken)
        {
            _timer = new Timer(DoWork, null, TimeSpan.Zero, TimeSpan.FromSeconds(5));

            return Task.CompletedTask;
        }

        private void DoWork(object? state)
        {
            Console.WriteLine($"{DateTime.Now:yyyy-MM-dd HH:mm:ss}");
        }

        public Task StopAsync(CancellationToken cancellationToken)
        {
            Console.WriteLine("StopAsync");

            return Task.CompletedTask;
        }

        public void Dispose()
        {
            _timer?.Dispose();
        }
    }
}
```


上面的Demo代码非常简单，应用在运行后，会去执行 StartAsync 函数，应用关闭执行 StopAsync，由于这里使用的定时器，所以每过5秒都会执行一次 DoWork 函数。


### 3、运行效果


![](https://img2024.cnblogs.com/blog/576536/202408/576536-20240803150933784-1989824344.png)


### 4、IHostedService 说明


注意：定时是不等待任务执行完成，只要时间一到，就会调用 DoWork 函数，所以适合一些简单、特定的场景。


以下为官方文档对 IHostedService 接口 的说明


IHostedService 接口为主机托管的对象定义了两种方法：


* StartAsync(CancellationToken)
* StopAsync(CancellationToken)


**StartAsync(CancellationToken)** 包含用于启动后台任务的逻辑。 在以下操作之前调用 \`StartAsync\`：已配置应用的请求处理管道。已启动服务器且已触发 IApplicationLifetime.ApplicationStarted。


StartAsync应仅限于短期任务，因为托管服务是按顺序运行的，在 StartAsync 运行完成之前不会启动其他服务。


**StopAsync(CancellationToken)** 在主机执行正常关闭时触发。 StopAsync\`包含结束后台任务的逻辑。 实现 IDisposable 和终结器（析构函数）以处置任何非托管资源。


默认情况下，取消令牌会有五秒超时，以指示关闭进程不再正常。 在令牌上请求取消时：


* 应中止应用正在执行的任何剩余后台操作。
* StopAsync 中调用的任何方法都应及时返回。


但是，在请求取消后，将不会放弃任务，调用方会等待所有任务完成。


如果应用意外关闭（例如，应用的进程失败），则可能不会调用 StopAsync。 因此，在 StopAsync 中执行的任何方法或操作都可能不会发生。


若要延长默认值为 5 秒的关闭超时值，请设置：


* ShutdownTimeout（当使用通用主机时）
* 使用 Web 主机时为关闭超时值主机配置设置


托管服务在应用启动时激活一次，在应用关闭时正常关闭。 如果在执行后台任务期间引发错误，即使未调用 StopAsync，也应调用 Dispose。


## BackgroundService 示例


### 1、注册服务


首先，同样需要在配置中注册服务接口。




```
using ManageCore.Api;
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped();
```


### 2、BackgroundService 源码


查看 BackgroundService 的源码，帮助我们理解BackgroundService 实现原理。BackgroundService 是 IHostedService的一个简单实现，内部 IHostedService 的 StartAsync 调用了 ExecuteAsync，本质上就是使用了 IHostedService。




```
public abstract class BackgroundService : IHostedService, IDisposable
{
    private Task _executingTask;
    private readonly CancellationTokenSource _stoppingCts = new CancellationTokenSource();

    /// 
    /// This method is called when the  starts. The implementation should return a task that represents
    /// the lifetime of the long running operation(s) being performed.
    /// /// 
    /// Triggered when  is called.
    /// A  that represents the long running operations.
    protected abstract Task ExecuteAsync(CancellationToken stoppingToken);

    /// 
    /// Triggered when the application host is ready to start the service.
    /// 
    /// Indicates that the start process has been aborted.
    public virtual Task StartAsync(CancellationToken cancellationToken)
    {
        // Store the task we're executing
        _executingTask = ExecuteAsync(_stoppingCts.Token);

        // If the task is completed then return it, this will bubble cancellation and failure to the caller
        if (_executingTask.IsCompleted)
        {
            return _executingTask;
        }

        // Otherwise it's running
        return Task.CompletedTask;
    }

    /// 
    /// Triggered when the application host is performing a graceful shutdown.
    /// 
    /// Indicates that the shutdown process should no longer be graceful.
    public virtual async Task StopAsync(CancellationToken cancellationToken)
    {
        // Stop called without start
        if (_executingTask == null)
        {
            return;
        }

        try
        {
            // Signal cancellation to the executing method
            _stoppingCts.Cancel();
        }
        finally
        {
            // Wait until the task completes or the stop token triggers
            await Task.WhenAny(_executingTask, Task.Delay(Timeout.Infinite, cancellationToken));
        }

    }

    public virtual void Dispose()
    {
        _stoppingCts.Cancel();
    }
}
```


### 3、创建服务接口


创建一个服务接口，定义需要实现的任务，以及对应的实现，如果需要执行异步方法，记得加上 await，不然任务将不会等待执行结果，直接进行下一个任务。




```
namespace ManageCore.Api
{
    public interface IDemoTaskWorkService
    {
        /// 
        /// 测试任务
        /// 
        /// 
        /// 
        Task TaskWorkAsync(CancellationToken stoppingToken);
    }
}
```




```
public class DemoTaskWorkService : IDemoTaskWorkService
{
      /// 
      /// 任务执行
      /// 
      /// 
      /// 
      public async Task TaskWorkAsync(CancellationToken stoppingToken)
      {
          while (!stoppingToken.IsCancellationRequested)
          {
              //执行任务
              Console.WriteLine($"{DateTime.Now}");

              //周期性任务，于上次任务执行完成后，等待5秒，执行下一次任务
              await Task.Delay(500);
          }
      }
 }
```


创建后台服务类，继承基类 BackgroundService，这里需要注意的是，要在 BackgroundService 中使用有作用域的服务，请创建作用域， 默认情况下，不会为托管服务创建作用域，得自己管理服务的生命周期，切记！于构造函数中注入 IServiceProvider即可。




```
namespace ManageCore.Api
{
    public class DemoBackgroundService : BackgroundService
    {
        private readonly IServiceProvider _services;

        public DemoBackgroundService(IServiceProvider services)
        {
            _services = services;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            using var scope = _services.CreateScope();

            var taskWorkService = scope.ServiceProvider.GetRequiredService();

            await taskWorkService.TaskWorkAsync(stoppingToken);
        }
    }
}
```


DemoBackgroundService类也是需要注册的，注册方式与 IHostedService 接口的方式一样




```
builder.Services.AddHostedService();
```


### 4、运行效果


![](https://img2024.cnblogs.com/blog/576536/202408/576536-20240803151406538-1875724380.png)


### 5、BackgroundService 说明


BackgroundService 是用于实现长时间运行的 IHostedService 的基类。


调用 ExecuteAsync(CancellationToken) 来运行后台服务。 实现返回一个 Task，其表示后台服务的整个生存期。


在 ExecuteAsync 变为异步（例如通过调用 await）之前，不会启动任何其他服务。 避免在 ExecuteAsync 中执行长时间的阻塞初始化工作。


StopAsync(CancellationToken) 中的主机块等待完成 ExecuteAsync。


调用 IHostedService.StopAsync 时，将触发取消令牌。 当激发取消令牌以便正常关闭服务时，ExecuteAsync 的实现应立即完成。 否则，服务将在关闭超时后不正常关闭。


StartAsync 应仅限于短期任务，因为托管服务是按顺序运行的，在 StartAsync 运行完成之前不会启动其他服务。


长期任务应放置在 ExecuteAsync 中。


## IHostedService 和 BackgroundService 区别


### 抽象级别


* IHostedService：需要手动实现启动和停止逻辑。
* BackgroundService：通过提供具有要重写的单个方法的基类来简化实现。


### 使用案例


* IHostedService：适用于需要对服务生命周期进行精细控制的更复杂的方案。
* BackgroundService：非常适合受益于减少样板代码的更简单、长时间运行的任务。


## 总结


总之.NET 8 中的 IHostedService 和 BackgroundService 提供了强大的工具集，使定时任务、后台处理以及定期维护等功能的实现变得更加直接、高效和灵活。无论是构建复杂的企业级应用还是简单的服务应用，这两个组件都能提供稳定且高效的解决方案。


如果你觉得这篇文章对你有帮助，不妨点个赞支持一下！你的支持是我继续分享知识的动力。如果有任何疑问或需要进一步的帮助，欢迎随时留言。


也可以加入微信公众号 \[DotNet技术匠] 社区，与其他热爱技术的同行一起交流心得，共同成长！


![](https://img2024.cnblogs.com/blog/576536/202407/576536-20240722093332651-213039456.png)


