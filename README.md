# Ibis.MutexLeaderElection

Implementation of the [Leader Election pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/leader-election) based on a distributed lock by taking a lease on an Azure Storage Blob:

> Coordinate the actions performed by a collection of collaborating instances in a distributed application by electing one instance as the leader that assumes responsibility for managing the others. This can help to ensure that instances don't conflict with each other, cause contention for shared resources, or inadvertently interfere with the work that other instances are performing.

This repository is heavily based on the code provided in the link shown above, but with updated libs and available as a NuGet package.


[Download the NuGet package](https://www.nuget.org/packages/MutexLeaderElection) [![NuGet Status](http://img.shields.io/nuget/v/MutexLeaderElection.svg?style=flat)](https://www.nuget.org/packages/MutexLeaderElection/)

# Usage

A sample application is included that demonstrates the use based on the following scenario:

- A .Net Web Application hosts a background job
- Multiple instances of the application are running (for example, in an Azure Kubernetes Service)
- Only one instance should have a running background job

Normally one would consider moving the background job out of the Web Application into an Azure Function or an other alternative. That would be the recommended approach but there might me plenty of reasons why that is not doable. Therefore this solution is a practical approach.

## Configuration

In the `Startup.cs` file the services should be configured:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    var distributedLockOptions = _configuration
        .GetSection(nameof(DistributedLockOptions))
        .Get<DistributedLockOptions>();

    services.AddApplicationInsightsTelemetry();

    services.AddSingleton<ILeaderElection, LeaderElection>();
    services.AddSingleton<IDistributedLock, DistributedLock>();
    services.AddSingleton(distributedLockOptions);
    services.AddHostedService<DemoBackgroundService>();
}
```  

The background worker can then use the `LeaderElection` instance to try to become the leader and do its work:

```csharp
    public class DemoBackgroundService : IHostedService
    {
        private readonly ILogger<BackgroundService> _logger;
        private readonly ILeaderElection _leaderElection;
        private CancellationTokenSource? _cancellationTokenSource;
        private Task _continuousTask = Task.CompletedTask;

        public DemoBackgroundService(ILogger<BackgroundService> logger, ILeaderElection leaderElection)
        {
            _logger = logger;
            _leaderElection = leaderElection;
        }

        public Task StartAsync(CancellationToken cancellationToken)
        {
            _logger.LogInformation("Starting DemoBackgroundService");

            _cancellationTokenSource = CancellationTokenSource.CreateLinkedTokenSource(cancellationToken);
            _continuousTask = _leaderElection.RunTaskWhenElectedLeaderAsync(WorkerAsync, _cancellationTokenSource.Token);

            return Task.CompletedTask;
        }

        public async Task StopAsync(CancellationToken cancellationToken)
        {
            _logger.LogInformation("Stopping DemoBackgroundService");

            _cancellationTokenSource?.Cancel();

            await _continuousTask;
        }

        public async Task WorkerAsync(CancellationToken cancellationToken)
        {
            while (!cancellationToken.IsCancellationRequested)
            {
                _logger.LogInformation("Working...");
                await Task.Delay(TimeSpan.FromSeconds(10), cancellationToken);
            }
        }
    }
```

## Implementation

Once `RunTaskWhenElectedLeaderAsync` is called an attempt is made to acquire a lease on a blob. If it succeeds it will run the specified tasks and keep renewing the lease at the same time. If it fails to acquire a lease it will wait for a specified time before trying to acquire a lease on a blob again.

When the task is cancelled the lease is released. If the process terminates without releasing the lease explicitly it will automatically be release after the specified lease interval has elapsed.  



