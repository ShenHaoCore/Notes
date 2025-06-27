# 微秒级定时器

## 事件方式

实现方法：

```csharp
/// <summary>
/// 微秒级定时器
/// </summary>
public class MicrosecondTimer : IDisposable
{
    private const long MicrosecondsPerSecond = 1000000; // 每秒微秒数
    private readonly Stopwatch stopwatch = new Stopwatch();
    private readonly object lockObj = new object();
    private long interval;
    private bool enabled;
    private Task? timerTask;
    private CancellationTokenSource cancellationTokenSource;

    public event EventHandler<MicrosecondTimerEventArgs>? Elapsed;

    /// <summary>
    /// 
    /// </summary>
    public MicrosecondTimer()
    {
        interval = Stopwatch.Frequency / 1000; // 默认1毫秒
        enabled = false;
        cancellationTokenSource = new CancellationTokenSource();
    }

    /// <summary>
    /// 
    /// </summary>
    public bool Enabled
    {
        get => enabled;
        set
        {
            if (enabled != value)
            {
                lock (lockObj)
                {
                    enabled = value;
                    if (enabled)
                    {
                        Start();
                    }
                    else
                    {
                       Stop();
                    }
                }
            }
        }
    }

    /// <summary>
    /// 
    /// </summary>
    public long Interval
    {
        get { lock (lockObj) { return interval * MicrosecondsPerSecond / Stopwatch.Frequency; } }
        set { lock (lockObj) { interval = value * Stopwatch.Frequency / MicrosecondsPerSecond; } }
    }

    /// <summary>
    /// 
    /// </summary>
    public void Dispose()
    {
        Stop();
        cancellationTokenSource?.Dispose();
        stopwatch.Stop();
    }

    /// <summary>
    /// 
    /// </summary>
    private void Start()
    {
        cancellationTokenSource = new CancellationTokenSource();
        timerTask = Task.Run(() => TimerLoopAsync(cancellationTokenSource.Token), cancellationTokenSource.Token);
    }

    /// <summary>
    /// 
    /// </summary>
    private void Stop()
    {
        stopwatch.Stop();
        cancellationTokenSource?.Cancel();
        timerTask?.Wait();
    }

    /// <summary>
    /// 
    /// </summary>
    /// <returns></returns>
    private async Task TimerLoopAsync(CancellationToken token)
    {
        stopwatch.Restart();
        long lastTrigger = stopwatch.ElapsedTicks;
        while (!token.IsCancellationRequested)
        {
            long current = stopwatch.ElapsedTicks;
            if (current - lastTrigger >= interval)
            {
                OnElapsed(new MicrosecondTimerEventArgs((current - lastTrigger) * MicrosecondsPerSecond / Stopwatch.Frequency));
                lastTrigger = current;
            }

            long nextTriggerTime = lastTrigger + interval;
            long remainingTicks = nextTriggerTime - stopwatch.ElapsedTicks;
            if (remainingTicks > 0)
            {
                int delayMilliseconds = (int)Math.Max((remainingTicks * MicrosecondsPerSecond / Stopwatch.Frequency / 1000), 1);
                await Task.Delay(delayMilliseconds, token);
            }
            else
            {
                await Task.Yield();
            }
        }
    }

    /// <summary>
    /// 
    /// </summary>
    /// <param name="e"></param>
    protected virtual void OnElapsed(MicrosecondTimerEventArgs e)
    {
        try
        {
            Elapsed?.Invoke(this, e);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"程序错误：{ex.Message}");
        }
    }
}

/// <summary>
/// 
/// </summary>
public class MicrosecondTimerEventArgs : EventArgs
{
    public long ElapsedMicroseconds { get; }

    /// <summary>
    /// 
    /// </summary>
    /// <param name="elapsedMicroseconds"></param>
    public MicrosecondTimerEventArgs(long elapsedMicroseconds)
    {
        ElapsedMicroseconds = elapsedMicroseconds;
    }
}
```

调用方法：

```csharp
using (MicrosecondTimer timer = new MicrosecondTimer())
{
    timer.Interval = 1000; // 设置为1000微秒（1毫秒）
    timer.Elapsed += (sender, e) => { Console.WriteLine($"定时器触发：{e.ElapsedMicroseconds}微秒"); };
    timer.Enabled = true;

    Console.WriteLine("按任意键停止计时器...");
    Console.ReadKey();
}
```
## 委托方式

实现方法：

```csharp
/// <summary>
/// 微秒级定时器
/// </summary>
public class MicrosecondTimer : IDisposable
{
    private const long MicrosecondsPerSecond = 1000000; // 每秒微秒数
    private readonly Stopwatch stopwatch = new Stopwatch();
    private long intervalInTicks;
    private bool isRunning;
    private Task? timerTask;
    private CancellationTokenSource? cancellationTokenSource;
    private Action<long>? tickCallback;

    /// <summary>
    /// 构造函数
    /// </summary>
    public MicrosecondTimer()
    {
        stopwatch.Reset();
    }

    /// <summary>
    /// 释放资源
    /// </summary>
    public void Dispose() => Stop();

    /// <summary>
    /// 启动定时器，默认间隔时间为1毫秒（1000微秒）
    /// </summary>
    /// <param name="callback">回调函数</param>
    public void Start(Action<long> callback) => Start(1000, callback);

    /// <summary>
    /// 启动定时器
    /// </summary>
    /// <param name="intervalMicroseconds">间隔时间（单位：微秒）</param>
    /// <param name="callback">回调函数</param>
    public void Start(long intervalMicroseconds, Action<long> callback)
    {
        if (isRunning) { Stop(); }

        this.tickCallback = callback;
        intervalInTicks = (long)(intervalMicroseconds * (Stopwatch.Frequency / MicrosecondsPerSecond));
        cancellationTokenSource = new CancellationTokenSource();
        stopwatch.Restart();
        timerTask = Task.Run(() => TimerLoopAsync(cancellationTokenSource.Token), cancellationTokenSource.Token);
        isRunning = true;
    }

    /// <summary>
    /// 创建并启动一个新的定时器实例
    /// </summary>
    /// <param name="intervalMicroseconds">间隔时间（单位：微秒）</param>
    /// <param name="callback">回调函数</param>
    /// <returns>已启动的定时器实例</returns>
    public static MicrosecondTimer StartNew(long intervalMicroseconds, Action<long> callback)
    {
        MicrosecondTimer timer = new MicrosecondTimer();
        timer.Start(intervalMicroseconds, callback);
        return timer;
    }

    /// <summary>
    /// 停止定时器
    /// </summary>
    public void Stop()
    {
        if (!isRunning) { return; }

        isRunning = false;
        stopwatch.Stop();
        cancellationTokenSource?.Cancel();
        try
        {
            timerTask?.Wait(); // 等待任务完成，可能抛出异常
        }
        catch (AggregateException ae)
        {
            ae.Handle(ex => ex is OperationCanceledException || ex is TaskCanceledException); // 忽略任务取消引起的异常
        }
        finally
        {
            cancellationTokenSource?.Dispose();
            cancellationTokenSource = null;
            tickCallback = null;
        }
    }

    /// <summary>
    /// 定时器主循环
    /// </summary>
    /// <param name="cancellationToken"></param>
    private async Task TimerLoopAsync(CancellationToken cancellationToken)
    {
        long lastTriggerTime = stopwatch.ElapsedTicks;
        while (!cancellationToken.IsCancellationRequested)
        {
            long currentTime = stopwatch.ElapsedTicks;
            if (currentTime - lastTriggerTime >= intervalInTicks)
            {
                try
                {
                    long elapsedMicroseconds = (currentTime - lastTriggerTime) * MicrosecondsPerSecond / Stopwatch.Frequency;
                    tickCallback?.Invoke(elapsedMicroseconds);
                }
                catch (Exception ex)
                {
                    Console.WriteLine($"回调错误：{ex.Message}");
                }
                lastTriggerTime = currentTime;
            }

            long nextTriggerTime = lastTriggerTime + intervalInTicks;
            long remainingTicks = nextTriggerTime - stopwatch.ElapsedTicks;
            if (remainingTicks > 0)
            {
                int delayMilliseconds = (int)Math.Max((remainingTicks * MicrosecondsPerSecond / Stopwatch.Frequency / 1000), 1);
                await Task.Delay(delayMilliseconds, cancellationToken);
            }
            else
            {
                await Task.Yield();
            }
        }
    }
}
```

调用方法：

```csharp
using (MicrosecondTimer timer = new MicrosecondTimer())
{
    timer.Start(1000, intervalMicroseconds => { Console.WriteLine($"[{clock.GetFormattedTime()}]计时器已用：{intervalMicroseconds}微秒"); });
    Console.WriteLine("按任意键停止计时器...");
    Console.ReadKey();
}
```

```csharp
using (MicrosecondTimer timer = MicrosecondTimer.StartNew(1000, OnTimerTick))
{
    Console.WriteLine("按任意键停止计时器...");
    Console.ReadKey();
}

/// <summary>
/// 定时器触发时的回调函数
/// </summary>
/// <param name="elapsed">自上次触发以来的微秒数（单位：微秒）</param>
static void OnTimerTick(long elapsed)
{
    Console.WriteLine($"定时器触发：{elapsed}微秒");
}
```