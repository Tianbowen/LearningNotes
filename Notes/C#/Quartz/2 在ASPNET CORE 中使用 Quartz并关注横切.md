简单使用 - 自定义JobFactory 和单例IJob

```csharp
public class HelloWorldJob : IJob
{
    private readonly ILogger<HelloWorldJob> _logger;
    public HelloWorldJob(ILogger<HelloWorldJob> logger)
    {
        _logger = logger;
    }

    public Task Execute(IJobExecutionContext context)
    {
        _logger.LogInformation("Hello world!");
        return Task.CompletedTask;
    }
}
```

```csharp
// 实现IJobFactory 可以在需要时从DI容器中检索Job实例
public class SingletonJobFactory : IJobFactory
{
    private readonly IServiceProvider _serviceProvider;
    public SingletonJobFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IJob NewJob(TriggerFiredBundle bundle, IScheduler scheduler)
    {
        return _serviceProvider.GetRequiredService(bundle.JobDetail.JobType) as IJob;
    }

    public void ReturnJob(IJob job) { }
}
```

```csharp
services.AddSingleton<IJobFactory, SingletonJobFactory>();
services.AddSingleton<HelloWorldJob>();
```

之前是 手动创建 Scope Service 例如：

```csharp
public class EmailReminderJob : IJob
{
    private readonly IServiceProvider _provider;
    public EmailReminderJob( IServiceProvider provider)
    {
        _provider = provider;
    }

    public Task Execute(IJobExecutionContext context)
    {
        using(var scope = _provider.CreateScope())
        {
            var dbContext = scope.ServiceProvider.GetService<AppDbContext>();
            var emailSender = scope.ServiceProvider.GetService<IEmailSender>();
            // fetch customers, send email, update DB
        }
 
        return Task.CompletedTask;
    }
}
```

这种方式可以用，但是使用调解器模式(MediatR)来处理横切关注点，例如工作单元和消息调度。

1. 创建 QuartzJobRunner ,创建中间IJob实现 , QuartzJobRunner位于IJobFactory与IJob之间.

```csharp
using Microsoft.Extensions.DependencyInjection;
using Quartz;
using Quartz.Spi;
using System;

public class JobFactory : IJobFactory
{
    private readonly IServiceProvider _serviceProvider;
    public JobFactory(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public IJob NewJob(TriggerFiredBundle bundle, IScheduler scheduler)
    {
        return _serviceProvider.GetRequiredService<QuartzJobRunner>();
    }

    public void ReturnJob(IJob job) { }
}
```

```csharp
services.AddSingleton<QuartzJobRunner>();
```

```csharp
using Microsoft.Extensions.DependencyInjection;
using Quartz;
using System;
using System.Threading.Tasks;

public class QuartzJobRunner : IJob
{
    private readonly IServiceProvider _serviceProvider;
    public QuartzJobRunner(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            var jobType = context.JobDetail.JobType;
            var job = scope.ServiceProvider.GetRequiredService(jobType) as IJob;

            await job.Execute(context);
        }
    }
}
```

这里做有两个优点：

- 我们可以将其注册`EmailReminderJob`为*范围*服务，并直接将任何依赖项注入其构造函数
- 我们可以将其他横切关注点移到`QuartzJobRunner`类中。

2. 作业直接使用Scope Service

```csharp
[DisallowConcurrentExecution]
public class EmailReminderJob : IJob
{
    private readonly AppDbContext _dbContext;
    private readonly IEmailSender _emailSender;
    public EmailReminderJob(AppDbContext dbContext, IEmailSender emailSender)
    {
        _dbContext = dbContext;
        _emailSender = emailSender;
    }

    public Task Execute(IJobExecutionContext context)
    {
        // fetch customers, send email, update DB
        return Task.CompletedTask;
    }
}
```

```csharp
services.AddScoped<EmailReminderJob>();
services.AddSingleton(new JobSchedule(
    jobType: typeof(EmailReminderJob),
    cronExpression: "0 0 12 * * ?")); // every day at noon
```

3. QuartzJobRunner 可以处理横切关注点

```csharp
public class QuartzJobRunner : IJob
{
    private readonly IServiceProvider _serviceProvider;

    public QuartzJobRunner(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public async Task Execute(IJobExecutionContext context)
    {
        using (var scope = _serviceProvider.CreateScope())
        {
            var jobType = context.JobDetail.JobType;
            var job = scope.ServiceProvider.GetRequiredService(jobType) as IJob;

            var dbContext = _serviceProvider.GetRequiredService<AppDbContext>();
            var messageBus = _serviceProvider.GetRequiredService<IBus>();

            await job.Execute(context);

            // job completed, save dbContext changes
            await dbContext.SaveChangesAsync();

            // db transaction succeeded, send messages
            await messageBus.DispatchAsync();
        }
    }
}
```

https://andrewlock.net/using-scoped-services-inside-a-quartz-net-hosted-service-with-asp-net-core/