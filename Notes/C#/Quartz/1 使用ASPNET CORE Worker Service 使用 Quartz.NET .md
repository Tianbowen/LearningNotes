在ASP.NET CORE Worker Service 中使用Quartz.NET 提供更加强大的解决方案



1. 安装Quartz.Net

```bash
dotnet add package Quartz
```

2. 创建IJob

```csharp
[DisallowConcurrentExecution]
public class HelloWorldJob : IJob
{
    
}
```

**[DisallowConcurrentExecution]**  此属性可防止Quartz尝试同时运行相同作业

3. 创建IJobFactory

```csharp
using Microsoft.Extensions.DependencyInjection;
using Quartz;
using Quartz.Spi;
using System;

public class SingletonJobFactory : IJobFactory{
    private readonly IServiceProvider _serviceProvider;
    public SingletonJobFactory(IServiceProvider _serviceProvider){
        _serviceProvider = serviceProvider;
    }
    
    public IJob NewJob(TriggerFiredBundle bundle,IScheduler scheduler){
        return _serviceProvider.GetRequiredService(bundle.JobDetail.JobType) as IJob;
    }
    
    public void ReturnJob(IJob job) { }
}
```

4. 配置作业

```csharp
using System;

public class JobSchedule{
    public JobSchedule(Type jobType, string cronExpression)
    {
        JobType = jobType;
        CronExpression = cronExpression;
    }

    public Type JobType { get; }
    public string CronExpression { get; }
}
```

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Quartz;
using Quartz.Impl;
using Quartz.Spi;

public void ConfigureServices(IServiceCollection services)
{
    // Add Quartz services
    services.AddSingleton<IJobFactory, SingletonJobFactory>();
    services.AddSingleton<ISchedulerFactory, StdSchedulerFactory>();

    // Add our job
    services.AddSingleton<HelloWorldJob>();
    services.AddSingleton(new JobSchedule(
        jobType: typeof(HelloWorldJob),
        cronExpression: "0/5 * * * * ?")); // run every 5 seconds
}
```

5. 创建 QuartzHostedService

```csharp
using System;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using Quartz;
using Quartz.Spi;

public class QuartzHostedService : IHostedService
{
    private readonly ISchedulerFactory _schedulerFactory;
    private readonly IJobFactory _jobFactory;
    private readonly IEnumerable<JobSchedule> _jobSchedules;

    public QuartzHostedService(
        ISchedulerFactory schedulerFactory,
        IJobFactory jobFactory,
        IEnumerable<JobSchedule> jobSchedules)
    {
        _schedulerFactory = schedulerFactory;
        _jobSchedules = jobSchedules;
        _jobFactory = jobFactory;
    }
    public IScheduler Scheduler { get; set; }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        Scheduler = await _schedulerFactory.GetScheduler(cancellationToken);
        Scheduler.JobFactory = _jobFactory;

        foreach (var jobSchedule in _jobSchedules)
        {
            var job = CreateJob(jobSchedule);
            var trigger = CreateTrigger(jobSchedule);

            await Scheduler.ScheduleJob(job, trigger, cancellationToken);
        }

        await Scheduler.Start(cancellationToken);
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await Scheduler?.Shutdown(cancellationToken);
    }

    private static IJobDetail CreateJob(JobSchedule schedule)
    {
        var jobType = schedule.JobType;
        return JobBuilder
            .Create(jobType)
            .WithIdentity(jobType.FullName)
            .WithDescription(jobType.Name)
            .Build();
    }

    private static ITrigger CreateTrigger(JobSchedule schedule)
    {
        return TriggerBuilder
            .Create()
            .WithIdentity($"{schedule.JobType.FullName}.trigger")
            .WithCronSchedule(schedule.CronExpression)
            .WithDescription(schedule.CronExpression)
            .Build();
    }
}
```

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddHostedService<QuartzHostedService>();
}
```

6. 在Job中使用scoped services 

```csharp
public class HelloWorldJob : IJob
{
    // Inject the DI provider
    private readonly IServiceProvider _provider;
    public HelloWorldJob( IServiceProvider provider)
    {
        _provider = provider;
    }

    public Task Execute(IJobExecutionContext context)
    {
        // Create a new scope
        using(var scope = _provider.CreateScope())
        {
            // Resolve the Scoped service
            var service = scope.ServiceProvider.GetService<IScopedService>();
            _logger.LogInformation("Hello world!");
        }
 
        return Task.CompletedTask;
    }
}
```

