[合集 - 消息队列RabbitMQ教程(10)](https://github.com)

[1.【RabbitMQ】环境搭建，版本选择和安装，添加用户授权2023-12-31](https://github.com/jixingsuiyuan/p/15905326.html)[2.【RabbitMQ】消息队列理论部分，另一种环境搭建Docker运行RabbitMQ09-13](https://github.com/jixingsuiyuan/p/19089393)[3.【RabbitMQ】核心模型简介，以及消息的生产与消费09-20](https://github.com/jixingsuiyuan/p/19103083)[4.【RabbitMQ】工作队列（Work Queues）与消息确认（Ack）09-21](https://github.com/jixingsuiyuan/p/19104405)[5.【RabbitMQ】发布/订阅（Publish/Subscribe）与交换机（Exchange）09-22](https://github.com/jixingsuiyuan/p/19106408):[樱花宇宙官网](https://yzygzn.com)[6.【RabbitMQ】路由（Routing）与直连交换机（Direct Exchange）09-24](https://github.com/jixingsuiyuan/p/19108392)[7.【RabbitMQ】主题（Topics）与主题交换机（Topic Exchange）09-26](https://github.com/jixingsuiyuan/p/19114022)[8.【RabbitMQ】实现完整的消息可靠性保障体系09-28](https://github.com/jixingsuiyuan/p/19117336)[9.【RabbitMQ】与ASP.NET Core集成10-28](https://github.com/jixingsuiyuan/p/19167628)

10.【RabbitMQ】RPC模式（请求/回复）10-31

收起

## 本章目标

* 理解RabbitMQ RPC模式的工作原理和适用场景。
* 掌握回调队列（Callback Queue）和关联ID（Correlation Id）的使用。
* 实现基于RabbitMQ的异步RPC调用。
* 学习RPC模式下的错误处理和超时机制。
* 构建完整的微服务间同步通信解决方案。

---

## 一、理论部分

### 1. RPC模式简介

RPC（Remote Procedure Call）模式允许客户端应用程序调用远程服务器上的方法，就像调用本地方法一样。在RabbitMQ中，RPC是通过消息队列实现的异步RPC。

与传统HTTP RPC的区别：

* HTTP RPC：同步，直接连接，需要服务端在线
* 消息队列RPC：异步，通过消息代理，支持解耦和负载均衡

### 2. RabbitMQ RPC核心组件

1. 请求队列（Request Queue）：客户端发送请求的队列
2. 回复队列（Reply Queue）：服务器返回响应的队列
3. 关联ID（Correlation Id）：匹配请求和响应的唯一标识
4. 消息属性：使用`IBasicProperties.ReplyTo`和`IBasicProperties.CorrelationId`

### 3. RPC工作流程

```
Client端：
1. 生成唯一CorrelationId
2. 创建临时回复队列
3. 发送请求到请求队列，设置ReplyTo和CorrelationId
4. 监听回复队列，等待匹配的CorrelationId

Server端：
1. 监听请求队列
2. 处理请求
3. 将响应发送到请求中的ReplyTo队列
4. 设置相同的CorrelationId

Client端：
5. 收到响应，根据CorrelationId匹配请求
6. 处理响应
```

### 4. 适用场景

* 需要同步响应的异步操作
* 微服务间的同步通信
* 计算密集型任务的分布式处理
* 需要负载均衡的同步调用

---

## 二、实操部分：构建分布式计算服务

我们将创建一个分布式斐波那契数列计算服务，演示完整的RPC模式实现。

### 第1步：创建项目结构

```
# 创建解决方案
dotnet new sln -n RpcSystem

# 创建项目
dotnet new webapi -n RpcClient.API
dotnet new classlib -n RpcClient.Core
dotnet new classlib -n RpcServer.Service
dotnet new classlib -n RpcShared

# 添加到解决方案
dotnet sln add RpcClient.API/RpcClient.API.csproj
dotnet sln add RpcClient.Core/RpcClient.Core.csproj
dotnet sln add RpcServer.Service/RpcServer.Service.csproj
dotnet sln add RpcShared/RpcShared.csproj

# 添加项目引用
dotnet add RpcClient.API reference RpcClient.Core
dotnet add RpcClient.API reference RpcShared
dotnet add RpcClient.Core reference RpcShared
dotnet add RpcServer.Service reference RpcShared

# 添加NuGet包
cd RpcClient.API
dotnet add package RabbitMQ.Client

cd ../RpcClient.Core
dotnet add package RabbitMQ.Client

cd ../RpcServer.Service
dotnet add package RabbitMQ.Client
```

### 第2步：定义共享模型（RpcShared）

Models/RpcRequest.cs

![]()![]()

```
using System.Text.Json.Serialization;

namespace RpcShared.Models
{
    public class RpcRequest
    {
        [JsonPropertyName("requestId")]
        public string RequestId { get; set; } = Guid.NewGuid().ToString();

        [JsonPropertyName("method")]
        public string Method { get; set; } = string.Empty;

        [JsonPropertyName("parameters")]
        public Dictionary<string, object> Parameters { get; set; } = new();

        [JsonPropertyName("timestamp")]
        public DateTime Timestamp { get; set; } = DateTime.UtcNow;

        public RpcRequest WithParameter(string key, object value)
        {
            Parameters[key] = value;
            return this;
        }

        public T? GetParameter(string key)
        {
            if (Parameters.TryGetValue(key, out var value))
            {
                try
                {
                    return (T)Convert.ChangeType(value, typeof(T));
                }
                catch
                {
                    return default;
                }
            }
            return default;
        }
    }
}
```

View Code

Models/RpcResponse.cs

![]()![]()

```
using System.Text.Json.Serialization;

namespace RpcShared.Models
{
    public class RpcResponse
    {
        [JsonPropertyName("requestId")]
        public string RequestId { get; set; } = string.Empty;

        [JsonPropertyName("success")]
        public bool Success { get; set; }

        [JsonPropertyName("data")]
        public object? Data { get; set; }

        [JsonPropertyName("error")]
        public string? Error { get; set; }

        [JsonPropertyName("timestamp")]
        public DateTime Timestamp { get; set; } = DateTime.UtcNow;

        [JsonPropertyName("processingTimeMs")]
        public long ProcessingTimeMs { get; set; }

        public static RpcResponse SuccessResponse(string requestId, object data, long processingTimeMs = 0)
        {
            return new RpcResponse
            {
                RequestId = requestId,
                Success = true,
                Data = data,
                ProcessingTimeMs = processingTimeMs
            };
        }

        public static RpcResponse ErrorResponse(string requestId, string error, long processingTimeMs = 0)
        {
            return new RpcResponse
            {
                RequestId = requestId,
                Success = false,
                Error = error,
                ProcessingTimeMs = processingTimeMs
            };
        }

        public T? GetData()
        {
            if (Data is JsonElement jsonElement)
            {
                return jsonElement.Deserialize();
            }
            return Data is T typedData ? typedData : default;
        }
    }
}
```

View Code

Messages/FibonacciRequest.cs

![]()![]()

```
namespace RpcShared.Messages
{
    public class FibonacciRequest
    {
        public int Number { get; set; }
        public bool UseOptimizedAlgorithm { get; set; } = true;
    }

    public class FibonacciResponse
    {
        public long Result { get; set; }
        public long CalculationTimeMs { get; set; }
        public int InputNumber { get; set; }
    }
}
```

View Code

### 第3步：RPC客户端核心库（RpcClient.Core）

Services/IRpcClient.cs

![]()![]()

```
using RpcShared.Models;

namespace RpcClient.Core.Services
{
    public interface IRpcClient : IDisposable
    {
        Task CallAsync(RpcRequest request, TimeSpan timeout);
        Task CallAsync(RpcRequest request, TimeSpan timeout) where TResponse : class;
    }
}
```

View Code

Services/RpcClient.cs

```
using System.Collections.Concurrent;
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Logging;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using RpcShared.Models;

namespace RpcClient.Core.Services
{
    public class RpcClient : IRpcClient
    {
        private readonly IConnection _connection;
        private readonly IModel _channel;
        private readonly ILogger _logger;
        private readonly string _replyQueueName;
        private readonly ConcurrentDictionary<string, TaskCompletionSource> _pendingRequests;
        private readonly AsyncEventingBasicConsumer _consumer;
        private bool _disposed = false;

        public RpcClient(
            IConnectionFactory connectionFactory,
            ILogger logger)
        {
            _logger = logger;
            _pendingRequests = new ConcurrentDictionary<string, TaskCompletionSource>();

            // 建立连接和通道
            _connection = connectionFactory.CreateConnection();
            _channel = _connection.CreateModel();

            // 声明临时回复队列（排他性，连接关闭时自动删除）
            _replyQueueName = _channel.QueueDeclare(
                queue: "",
                durable: false,
                exclusive: true,
                autoDelete: true,
                arguments: null).QueueName;

            // 创建消费者监听回复队列
            _consumer = new AsyncEventingBasicConsumer(_channel);
            _consumer.Received += OnResponseReceived;

            // 开始消费回复队列
            _channel.BasicConsume(
                queue: _replyQueueName,
                autoAck: false,
                consumer: _consumer);

            _logger.LogInformation("RPC Client initialized with reply queue: {ReplyQueue}", _replyQueueName);
        }

        public async Task CallAsync(RpcRequest request, TimeSpan timeout)
        {
            if (_disposed)
                throw new ObjectDisposedException(nameof(RpcClient));

            var tcs = new TaskCompletionSource();
            var cancellationTokenSource = new CancellationTokenSource(timeout);

            // 注册超时取消
            cancellationTokenSource.Token.Register(() =>
            {
                if (_pendingRequests.TryRemove(request.RequestId, out var removedTcs))
                {
                    removedTcs.TrySetException(new TimeoutException($"RPC call timed out after {timeout.TotalSeconds} seconds"));
                    _logger.LogWarning("RPC request {RequestId} timed out", request.RequestId);
                }
            });

            // 将请求添加到待处理字典
            if (!_pendingRequests.TryAdd(request.RequestId, tcs))
            {
                throw new InvalidOperationException($"Request with ID {request.RequestId} is already pending");
            }

            try
            {
                // 序列化请求
                var requestJson = JsonSerializer.Serialize(request);
                var requestBody = Encoding.UTF8.GetBytes(requestJson);

                // 设置消息属性
                var properties = _channel.CreateBasicProperties();
                properties.ReplyTo = _replyQueueName;
                properties.CorrelationId = request.RequestId;
                properties.Persistent = true;

                _logger.LogDebug("Sending RPC request {RequestId} to queue: rpc_queue", request.RequestId);

                // 发布请求到RPC队列
                _channel.BasicPublish(
                    exchange: "",
                    routingKey: "rpc_queue",
                    basicProperties: properties,
                    body: requestBody);

                _logger.LogInformation("RPC request {RequestId} sent successfully", request.RequestId);

                // 等待响应
                return await tcs.Task;
            }
            catch (Exception ex)
            {
                // 发生异常时移除待处理请求
                _pendingRequests.TryRemove(request.RequestId, out _);
                _logger.LogError(ex, "Error sending RPC request {RequestId}", request.RequestId);
                throw;
            }
        }

        public async Task CallAsync(RpcRequest request, TimeSpan timeout) where TResponse : class
        {
            var response = await CallAsync(request, timeout);
            
            if (!response.Success)
            {
                throw new InvalidOperationException($"RPC call failed: {response.Error}");
            }

            return response.GetData();
        }

        private async Task OnResponseReceived(object sender, BasicDeliverEventArgs ea)
        {
            var responseBody = ea.Body.ToArray();
            var responseJson = Encoding.UTF8.GetString(responseBody);
            var correlationId = ea.BasicProperties.CorrelationId;

            _logger.LogDebug("Received RPC response for correlation ID: {CorrelationId}", correlationId);

            try
            {
                var response = JsonSerializer.Deserialize(responseJson);
                if (response == null)
                {
                    _logger.LogError("Failed to deserialize RPC response for correlation ID: {CorrelationId}", correlationId);
                    return;
                }

                // 查找匹配的待处理请求
                if (_pendingRequests.TryRemove(correlationId, out var tcs))
                {
                    tcs.TrySetResult(response);
                    _logger.LogDebug("RPC response for {CorrelationId} delivered to waiting task", correlationId);
                }
                else
                {
                    _logger.LogWarning("Received response for unknown correlation ID: {CorrelationId}", correlationId);
                }

                // 手动确认消息
                _channel.BasicAck(ea.DeliveryTag, false);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error processing RPC response for correlation ID: {CorrelationId}", correlationId);
                
                // 处理失败时拒绝消息（不重新入队）
                _channel.BasicNack(ea.DeliveryTag, false, false);
                
                // 如果反序列化失败，仍然通知等待的任务
                if (_pendingRequests.TryRemove(correlationId, out var tcs))
                {
                    tcs.TrySetException(new InvalidOperationException("Failed to process RPC response"));
                }
            }

            await Task.CompletedTask;
        }

        public void Dispose()
        {
            if (!_disposed)
            {
                _disposed = true;

                // 取消所有待处理的请求
                foreach (var (requestId, tcs) in _pendingRequests)
                {
                    tcs.TrySetCanceled();
                }
                _pendingRequests.Clear();

                _channel?.Close();
                _channel?.Dispose();
                _connection?.Close();
                _connection?.Dispose();

                _logger.LogInformation("RPC Client disposed");
            }
        }
    }
}
```

Services/FibonacciRpcClient.cs

```
using RpcClient.Core.Services;
using RpcShared.Messages;
using RpcShared.Models;

namespace RpcClient.Core.Services
{
    public class FibonacciRpcClient
    {
        private readonly IRpcClient _rpcClient;
        private readonly ILogger _logger;

        public FibonacciRpcClient(IRpcClient rpcClient, ILogger logger)
        {
            _rpcClient = rpcClient;
            _logger = logger;
        }

        public async Task<long> CalculateFibonacciAsync(int number, bool useOptimized = true, TimeSpan? timeout = null)
        {
            var request = new RpcRequest
            {
                Method = "fibonacci.calculate",
                Timestamp = DateTime.UtcNow
            }
            .WithParameter("number", number)
            .WithParameter("useOptimized", useOptimized);

            timeout ??= TimeSpan.FromSeconds(30);

            try
            {
                _logger.LogInformation("Calculating Fibonacci({Number}) with timeout {Timeout}s", 
                    number, timeout.Value.TotalSeconds);

                var response = await _rpcClient.CallAsync(request, timeout.Value);
                
                if (response != null)
                {
                    _logger.LogInformation(
                        "Fibonacci({Number}) = {Result} (calculated in {Time}ms)", 
                        number, response.Result, response.CalculationTimeMs);
                    
                    return response.Result;
                }

                throw new InvalidOperationException("Received null response from RPC server");
            }
            catch (TimeoutException ex)
            {
                _logger.LogError(ex, "Fibonacci calculation timed out for number {Number}", number);
                throw;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error calculating Fibonacci for number {Number}", number);
                throw;
            }
        }

        public async Task CalculateFibonacciDetailedAsync(int number, bool useOptimized = true, TimeSpan? timeout = null)
        {
            var request = new RpcRequest
            {
                Method = "fibonacci.calculate",
                Timestamp = DateTime.UtcNow
            }
            .WithParameter("number", number)
            .WithParameter("useOptimized", useOptimized);

            timeout ??= TimeSpan.FromSeconds(30);

            var response = await _rpcClient.CallAsync(request, timeout.Value);
            return response ?? throw new InvalidOperationException("Received null response from RPC server");
        }
    }
}
```

### 第4步：RPC客户端API（RpcClient.API）

Program.cs

![]()![]()

```
using RpcClient.API.Services;
using RpcClient.Core.Services;
using RpcShared.Models;
using RabbitMQ.Client;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Configure RabbitMQ
builder.Services.AddSingleton(sp =>
{
    var configuration = sp.GetRequiredService();
    return new ConnectionFactory
    {
        HostName = configuration["RabbitMQ:HostName"],
        UserName = configuration["RabbitMQ:UserName"],
        Password = configuration["RabbitMQ:Password"],
        Port = int.Parse(configuration["RabbitMQ:Port"] ?? "5672"),
        VirtualHost = configuration["RabbitMQ:VirtualHost"] ?? "/",
        DispatchConsumersAsync = true
    };
});

// Register RPC services
builder.Services.AddSingleton();
builder.Services.AddScoped();
builder.Services.AddScoped();

// Add health checks
builder.Services.AddHealthChecks()
    .AddRabbitMQ(provider => 
    {
        var factory = provider.GetRequiredService();
        return factory.CreateConnection();
    });

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();
app.MapHealthChecks("/health");

app.Run();
```

View Code

Services/IMathRpcService.cs

![]()![]()

```
using RpcShared.Messages;

namespace RpcClient.API.Services
{
    public interface IMathRpcService
    {
        Task<long> CalculateFibonacciAsync(int number);
        Task CalculateFibonacciDetailedAsync(int number);
        Task<bool> HealthCheckAsync();
    }
}
```

View Code

Services/MathRpcService.cs

![]()![]()

```
using RpcClient.Core.Services;
using RpcShared.Messages;

namespace RpcClient.API.Services
{
    public class MathRpcService : IMathRpcService
    {
        private readonly FibonacciRpcClient _fibonacciClient;
        private readonly ILogger _logger;

        public MathRpcService(FibonacciRpcClient fibonacciClient, ILogger logger)
        {
            _fibonacciClient = fibonacciClient;
            _logger = logger;
        }

        public async Task<long> CalculateFibonacciAsync(int number)
        {
            if (number < 0)
                throw new ArgumentException("Number must be non-negative", nameof(number));

            if (number > 50)
                throw new ArgumentException("Number too large for demonstration", nameof(number));

            return await _fibonacciClient.CalculateFibonacciAsync(number);
        }

        public async Task CalculateFibonacciDetailedAsync(int number)
        {
            if (number < 0)
                throw new ArgumentException("Number must be non-negative", nameof(number));

            if (number > 50)
                throw new ArgumentException("Number too large for demonstration", nameof(number));

            return await _fibonacciClient.CalculateFibonacciDetailedAsync(number);
        }

        public async Task<bool> HealthCheckAsync()
        {
            try
            {
                // 简单的健康检查：计算 Fibonacci(1)
                var result = await _fibonacciClient.CalculateFibonacciAsync(1, timeout: TimeSpan.FromSeconds(5));
                return result == 1;
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "RPC health check failed");
                return false;
            }
        }
    }
}
```

View Code

Controllers/MathController.cs

![]()![]()

```
using Microsoft.AspNetCore.Mvc;
using RpcClient.API.Services;
using RpcShared.Messages;

namespace RpcClient.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class MathController : ControllerBase
    {
        private readonly IMathRpcService _mathService;
        private readonly ILogger _logger;

        public MathController(IMathRpcService mathService, ILogger logger)
        {
            _mathService = mathService;
            _logger = logger;
        }

        [HttpGet("fibonacci/{number}")]
        public async Tasklong>> CalculateFibonacci(int number)
        {
            try
            {
                _logger.LogInformation("Calculating Fibonacci({Number}) via RPC", number);
                var result = await _mathService.CalculateFibonacciAsync(number);
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(ex.Message);
            }
            catch (TimeoutException ex)
            {
                _logger.LogWarning(ex, "Fibonacci calculation timed out for number {Number}", number);
                return StatusCode(408, "Calculation timed out");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error calculating Fibonacci for number {Number}", number);
                return StatusCode(500, "Internal server error");
            }
        }

        [HttpGet("fibonacci/{number}/detailed")]
        public async Task> CalculateFibonacciDetailed(int number)
        {
            try
            {
                _logger.LogInformation("Calculating Fibonacci({Number}) with details via RPC", number);
                var result = await _mathService.CalculateFibonacciDetailedAsync(number);
                return Ok(result);
            }
            catch (ArgumentException ex)
            {
                return BadRequest(ex.Message);
            }
            catch (TimeoutException ex)
            {
                _logger.LogWarning(ex, "Fibonacci calculation timed out for number {Number}", number);
                return StatusCode(408, "Calculation timed out");
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error calculating Fibonacci for number {Number}", number);
                return StatusCode(500, "Internal server error");
            }
        }

        [HttpGet("health")]
        public async Task HealthCheck()
        {
            var isHealthy = await _mathService.HealthCheckAsync();
            return isHealthy ? Ok("RPC service is healthy") : StatusCode(503, "RPC service is unavailable");
        }
    }
}
```

View Code

### 第5步：RPC服务器（RpcServer.Service）

Program.cs

![]()![]()

```
using RpcServer.Service.Services;
using RpcShared.Models;
using RabbitMQ.Client;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddHostedService();

// Configure RabbitMQ
builder.Services.AddSingleton(sp =>
{
    var configuration = sp.GetRequiredService();
    return new ConnectionFactory
    {
        HostName = configuration["RabbitMQ:HostName"],
        UserName = configuration["RabbitMQ:UserName"],
        Password = configuration["RabbitMQ:Password"],
        Port = int.Parse(configuration["RabbitMQ:Port"] ?? "5672"),
        VirtualHost = configuration["RabbitMQ:VirtualHost"] ?? "/",
        DispatchConsumersAsync = true
    };
});

builder.Services.AddSingleton();

var host = builder.Build();
host.Run();
```

View Code

Services/FibonacciCalculator.cs

![]()![]()

```
using RpcShared.Messages;

namespace RpcServer.Service.Services
{
    public class FibonacciCalculator
    {
        private readonly ILogger _logger;
        private readonly Dictionary<int, long> _cache = new();

        public FibonacciCalculator(ILogger logger)
        {
            _logger = logger;
        }

        public FibonacciResponse Calculate(int number, bool useOptimized = true)
        {
            var startTime = DateTime.UtcNow;

            try
            {
                _logger.LogInformation("Calculating Fibonacci({Number}) with optimized: {Optimized}", 
                    number, useOptimized);

                long result;
                if (useOptimized)
                {
                    result = CalculateOptimized(number);
                }
                else
                {
                    result = CalculateNaive(number);
                }

                var calculationTime = (DateTime.UtcNow - startTime).TotalMilliseconds;

                _logger.LogInformation(
                    "Fibonacci({Number}) = {Result} (calculated in {Time}ms)", 
                    number, result, calculationTime);

                return new FibonacciResponse
                {
                    Result = result,
                    CalculationTimeMs = (long)calculationTime,
                    InputNumber = number
                };
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error calculating Fibonacci({Number})", number);
                throw;
            }
        }

        private long CalculateOptimized(int n)
        {
            if (n < 0) throw new ArgumentException("Number must be non-negative");
            if (n <= 1) return n;

            // 检查缓存
            if (_cache.TryGetValue(n, out var cachedResult))
            {
                _logger.LogDebug("Cache hit for Fibonacci({Number})", n);
                return cachedResult;
            }

            long a = 0, b = 1;

            for (int i = 2; i <= n; i++)
            {
                var temp = a + b;
                a = b;
                b = temp;

                // 缓存中间结果
                if (i % 10 == 0) // 每10个数缓存一次以减少内存使用
                {
                    _cache[i] = b;
                }
            }

            // 缓存最终结果
            _cache[n] = b;
            return b;
        }

        private long CalculateNaive(int n)
        {
            if (n < 0) throw new ArgumentException("Number must be non-negative");
            if (n <= 1) return n;

            // 模拟计算密集型任务
            Thread.Sleep(100);

            return CalculateNaive(n - 1) + CalculateNaive(n - 2);
        }

        public void ClearCache()
        {
            _cache.Clear();
            _logger.LogInformation("Fibonacci cache cleared");
        }
    }
}
```

View Code

Services/FibonacciRpcServer.cs

![]()![]()

```
using System.Text;
using System.Text.Json;
using Microsoft.Extensions.Options;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;
using RpcShared.Messages;
using RpcShared.Models;

namespace RpcServer.Service.Services
{
    public class FibonacciRpcServer : BackgroundService
    {
        private readonly IConnection _connection;
        private readonly IModel _channel;
        private readonly FibonacciCalculator _calculator;
        private readonly ILogger _logger;
        private const string QueueName = "rpc_queue";

        public FibonacciRpcServer(
            IConnectionFactory connectionFactory,
            FibonacciCalculator calculator,
            ILogger logger)
        {
            _calculator = calculator;
            _logger = logger;

            // 建立连接和通道
            _connection = connectionFactory.CreateConnection();
            _channel = _connection.CreateModel();

            InitializeQueue();
        }

        private void InitializeQueue()
        {
            // 声明RPC请求队列（持久化）
            _channel.QueueDeclare(
                queue: QueueName,
                durable: true,
                exclusive: false,
                autoDelete: false,
                arguments: null);

            // 设置公平分发，每次只处理一个请求
            _channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);

            _logger.LogInformation("Fibonacci RPC Server initialized and listening on queue: {QueueName}", QueueName);
        }

        protected override Task ExecuteAsync(CancellationToken stoppingToken)
        {
            stoppingToken.ThrowIfCancellationRequested();

            var consumer = new AsyncEventingBasicConsumer(_channel);
            consumer.Received += async (model, ea) =>
            {
                RpcResponse response = null;
                string correlationId = ea.BasicProperties.CorrelationId;
                string replyTo = ea.BasicProperties.ReplyTo;

                _logger.LogDebug("Received RPC request with correlation ID: {CorrelationId}", correlationId);

                try
                {
                    // 处理请求
                    response = await ProcessRequestAsync(ea.Body.ToArray());
                    response.RequestId = correlationId;
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error processing RPC request {CorrelationId}", correlationId);
                    response = RpcResponse.ErrorResponse(correlationId, ex.Message);
                }
                finally
                {
                    // 发送响应
                    if (!string.IsNullOrEmpty(replyTo))
                    {
                        SendResponse(replyTo, correlationId, response ?? 
                            RpcResponse.ErrorResponse(correlationId, "Unknown error occurred"));
                    }

                    // 确认消息处理完成
                    _channel.BasicAck(ea.DeliveryTag, false);
                }
            };

            _channel.BasicConsume(
                queue: QueueName,
                autoAck: false, // 手动确认
                consumer: consumer);

            _logger.LogInformation("Fibonacci RPC Server started successfully");

            return Task.CompletedTask;
        }

        private async Task ProcessRequestAsync(byte[] body)
        {
            var startTime = DateTime.UtcNow;

            try
            {
                var requestJson = Encoding.UTF8.GetString(body);
                var request = JsonSerializer.Deserialize(requestJson);

                if (request == null)
                {
                    return RpcResponse.ErrorResponse("unknown", "Invalid request format");
                }

                _logger.LogInformation("Processing RPC request {RequestId}, Method: {Method}", 
                    request.RequestId, request.Method);

                // 根据方法名路由到不同的处理逻辑
                object result = request.Method.ToLowerInvariant() switch
                {
                    "fibonacci.calculate" => ProcessFibonacciRequest(request),
                    "ping" => new { message = "pong", timestamp = DateTime.UtcNow },
                    _ => throw new NotSupportedException($"Method {request.Method} is not supported")
                };

                var processingTime = (DateTime.UtcNow - startTime).TotalMilliseconds;

                return RpcResponse.SuccessResponse(
                    request.RequestId, 
                    result, 
                    (long)processingTime);
            }
            catch (Exception ex)
            {
                var processingTime = (DateTime.UtcNow - startTime).TotalMilliseconds;
                _logger.LogError(ex, "Error processing RPC request");
                
                return RpcResponse.ErrorResponse(
                    "unknown", 
                    ex.Message, 
                    (long)processingTime);
            }
        }

        private FibonacciResponse ProcessFibonacciRequest(RpcRequest request)
        {
            var number = request.GetParameter<int>("number");
            var useOptimized = request.GetParameter<bool>("useOptimized") ?? true;

            if (number < 0)
            {
                throw new ArgumentException("Fibonacci number must be non-negative");
            }

            // 防止过大的计算消耗资源
            if (number > 50)
            {
                throw new ArgumentException("Number too large for calculation");
            }

            return _calculator.Calculate(number, useOptimized);
        }

        private void SendResponse(string replyTo, string correlationId, RpcResponse response)
        {
            try
            {
                var responseJson = JsonSerializer.Serialize(response);
                var responseBody = Encoding.UTF8.GetBytes(responseJson);

                var properties = _channel.CreateBasicProperties();
                properties.CorrelationId = correlationId;
                properties.Persistent = true;

                _channel.BasicPublish(
                    exchange: "",
                    routingKey: replyTo,
                    basicProperties: properties,
                    body: responseBody);

                _logger.LogDebug("Sent RPC response for correlation ID: {CorrelationId}", correlationId);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error sending RPC response for correlation ID: {CorrelationId}", correlationId);
            }
        }

        public override void Dispose()
        {
            _channel?.Close();
            _channel?.Dispose();
            _connection?.Close();
            _connection?.Dispose();
            
            _logger.LogInformation("Fibonacci RPC Server disposed");
            
            base.Dispose();
        }
    }
}
```

View Code

### 第6步：高级特性 - 带重试的RPC客户端

Services/ResilientRpcClient.cs

![]()![]()

```
using Microsoft.Extensions.Logging;
using Polly;
using Polly.Retry;
using RpcShared.Models;

namespace RpcClient.Core.Services
{
    public class ResilientRpcClient : IRpcClient
    {
        private readonly IRpcClient _innerClient;
        private readonly ILogger _logger;
        private readonly AsyncRetryPolicy _retryPolicy;

        public ResilientRpcClient(IRpcClient innerClient, ILogger logger)
        {
            _innerClient = innerClient;
            _logger = logger;

            // 配置重试策略
            _retryPolicy = Policy
                .Handle()
                .Or()
                .WaitAndRetryAsync(
                    retryCount: 3,
                    sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                    onRetry: (exception, delay, retryCount, context) =>
                    {
                        _logger.LogWarning(
                            "RPC call failed. Retry {RetryCount} after {Delay}ms. Error: {Error}",
                            retryCount, delay.TotalMilliseconds, exception.Message);
                    });
        }

        public async Task CallAsync(RpcRequest request, TimeSpan timeout)
        {
            return await _retryPolicy.ExecuteAsync(async () =>
            {
                return await _innerClient.CallAsync(request, timeout);
            });
        }

        public async Task CallAsync(RpcRequest request, TimeSpan timeout) where TResponse : class
        {
            return await _retryPolicy.ExecuteAsync(async () =>
            {
                return await _innerClient.CallAsync(request, timeout);
            });
        }

        public void Dispose()
        {
            _innerClient?.Dispose();
        }
    }
}
```

View Code

### 第7步：运行与测试

1. 启动服务

   ```
   # 终端1：启动RPC服务器
   cd RpcServer.Service
   dotnet run

   # 终端2：启动RPC客户端API
   cd RpcClient.API
   dotnet run
   ```
2. 测试API

   ```
   # 计算斐波那契数列
   curl -X GET "https://localhost:7000/api/math/fibonacci/10"
   curl -X GET "https://localhost:7000/api/math/fibonacci/20/detailed"

   # 健康检查
   curl -X GET "https://localhost:7000/api/math/health"
   ```
3. 测试错误场景

   ```
   # 测试超时（设置很小的超时时间）
   # 测试无效输入
   curl -X GET "https://localhost:7000/api/math/fibonacci/-5"
   curl -X GET "https://localhost:7000/api/math/fibonacci/100"
   ```
4. 观察日志输出

   * 客户端发送请求，生成CorrelationId
   * 服务器接收请求，处理计算
   * 服务器发送响应，使用相同的CorrelationId
   * 客户端接收响应，匹配CorrelationId

### 第8步：性能测试和监控

创建性能测试控制器

```
[ApiController]
[Route("api/[controller]")]
public class BenchmarkController : ControllerBase
{
    private readonly IMathRpcService _mathService;
    private readonly ILogger _logger;

    public BenchmarkController(IMathRpcService mathService, ILogger logger)
    {
        _mathService = mathService;
        _logger = logger;
    }

    [HttpPost("fibonacci/batch")]
    public async Task CalculateFibonacciBatch([FromBody] List<int> numbers)
    {
        var results = new List<object>();
        var totalStopwatch = System.Diagnostics.Stopwatch.StartNew();

        foreach (var number in numbers)
        {
            var stopwatch = System.Diagnostics.Stopwatch.StartNew();
            try
            {
                var result = await _mathService.CalculateFibonacciAsync(number);
                results.Add(new
                {
                    number,
                    result,
                    success = true,
                    durationMs = stopwatch.ElapsedMilliseconds
                });
            }
            catch (Exception ex)
            {
                results.Add(new
                {
                    number,
                    success = false,
                    error = ex.Message,
                    durationMs = stopwatch.ElapsedMilliseconds
                });
            }
        }

        return Ok(new
        {
            totalDurationMs = totalStopwatch.ElapsedMilliseconds,
            requests = numbers.Count,
            results
        });
    }
}
```

---

## 本章总结

在这一章中，我们完整实现了RabbitMQ的RPC模式：

1. RPC核心概念：理解了回调队列、关联ID、请求-响应模式。
2. 客户端实现：创建了能够发送请求并异步等待响应的RPC客户端。
3. 服务器实现：构建了处理请求并返回响应的RPC服务器。
4. 错误处理：实现了超时控制、异常处理和重试机制。
5. 性能优化：使用缓存和优化算法提高计算效率。
6. \*\* resilience\*\*：通过Polly实现了弹性重试策略。

RPC模式为微服务架构提供了强大的同步通信能力，结合消息队列的异步特性，既保持了系统的解耦性，又提供了同步调用的便利性。这种模式特别适合需要等待计算结果的分布式任务。
