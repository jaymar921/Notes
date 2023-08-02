# Hosted and Worker Services

### What are hosted services?
Hosted service are deriving from the BackgroundService base class, hosted services provide a clean pattern for executing code within the ASP.NET core application that sits outside the request pipeline. Each service is started as an asynchronous background task. Hosted services are useful when you need to run some background activity within a webserver process.

### Use case
- Polling for data from an external service
- Responding to external messages or events
- Performing data-intensive work, outside of the request lifecycle

### Creating a Hosted Service
- Periodically perform background work
Package to Import
```c#
using Microsoft.Extensions.Hosting;
```
For this Example, we will be implementing the **BackgroundServices** abstract class for creation of background services
```c#
public abstract class BackgroundService : IHostedService, IDisposable
{
    protected BackgroundService();
    public virtual void Dispose();
    public virtual Task StartAsync(CancellationToken cancellationToken);
    public virtual Task StopAsync(CancellationToken cancellationToken);
    protected abstract Task ExecuteAsync(CancellationToken cancellationToken);
}
```

We will be creating a Weather Caching Service, basically when a clients requests for a Weather Service, we call an external API to recieve information about the weather forecast. To avoid calling the external API everytime the client calls a request, we will cache the external API result for faster response and to limit the number of external API calls.

```c#
public class WeatherCacheService : BackgroundService
{
    private readonly IWeatherApiClient _weatherApiClient;
    private readonly IDistributedCache<CurrentWeatherResult> _cache;
    private readonly ILogger<WeatherCacheService> _logger;

    private readonly int _minutesToCache;
    private readonly int _refreshIntervalInSeconds;

    /*
        Dependency inject all the required dependencies in the
        class constructor
    */
    public WeatherCacheService(
        IWeatherApiClient weatherApiClient,
        IDistributedCache<CurrentWeatherResult> cache,
        IOptionsMonitor<ExternalServicesConfig> options,
        ILogger<WeatherCacheService> logger
    )
    {
        _weatherApiClient = weatherApiClient;
        _cache = cache;
        _logger = logger;
        _minutesToCache = options.Get(ExternalServicesConfig.WeatherApi).MinsToCache;
        _refreshIntervalInSeconds = _minutesToCache > 1 (_minutesToCache - 1) * 60 : 30;
    }

    /*
        We are overriding the ExecuteAsync method from the BackgroundService abstract class

        Since we are setting a long running task, we will wrap all the logic inside the
        while loop and a cancellation token will be the option to break from the while loop
    */
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while(!stoppingToken.IsCancellationRequested)
        {
            // we call the external api from the weather api client and save the current forecast in an object
            var forecast = await _weatherApiClient.GetWeatherForecastAsync(stoppingToken);

            // if the forecast is not null, we can then execute the logics inside the 'if' block
            if(forecast is object){
                // create the current weather result object
                var currentWeather = new CurrentWeatherResult {
                    Description = forecast.Weather.Description
                };

                // we will be creating a key to be used to store the information in the cache
                var cacheKey = $"current_weather_{DateTime.UtcNow:yyyy_MM_dd}";

                _logger.LogInformation("Updating weather in cache");

                // we store the forecast data in the cache by calling the SetAsync method in IDistributedCache class
                await _cache.SetAsync(cacheKey, currentWeather, _minutesToCache);
            }

            // delays the execution until the calculated refresh interval has passed
            await Task.Delay(TimeSpan.FromSeconds(_refreshIntervalInSeconds), stoppingToken);
        }
    }
}
```
### Registering the Service
After creating the Services, in order for us to use them is to register the service in the Application's configuration. Inside the **Startup** class, register the service.
```c#
/*
    We will make the service conditional,
    only use the service when the configuration allows to use
    the weather service api feature
*/
if(Configuration.IsWeatherForecastEnabled())
{
    services.AddHostedService<WeatherCacheService>();
}
```
Once the application is running and the weather forecast feature is enabled in the configuration. The Service will automatically be executed. 

# Worker Services
A worker service is a .NET project built using a template which supplies a few useful features that turn a regular console application into something more powerful. A worker service runs on top of the concept of a host, which maintains the lifetime of the application. The host also makes available some familiar features, such as dependency injection, logging and configuration.

Worker services will generally be long-running services, performing some regularly occurring workload.
### Common Workloads
- Processing messages/events from a queue, service bus or event stream
- Reacting to file changes in a object/file store
- Aggregating data from a data store
- Enriching data in data ingestion pipelines
- Formatting and cleansing of AI/ML datasets
### Creating a Worker Service
Create a new project with a **Worker Service** project template or simple use the .NET CLI command below
```
dotnet new worker -n "WorkerService.ScoreProcessor"
```
### Hosting in .NET Core
A host in .NET core 
- Manages application lifetime
- Provides components such as logging, configuration and dependency injection
- Turns a console application into a long-running service
- Responsible for starting and stopping hosted services

[Worker services in .NET documentation at Microsoft](https://learn.microsoft.com/en-us/dotnet/core/extensions/workers?pivots=dotnet-7-0)
