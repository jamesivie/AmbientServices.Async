# Overview
AmbientServices.Async is a .NET library that provides tools for migrating even the largest, most challenging and performance-critical projects from non async/TPL code to modern .NET core async/await.

## AA
The static AA (AsyncAwait) class provides a way to make code async-ready little by little rather than the usual "forklift" update normally required due to the zombie virus nature of async.
First, let's get one of my pet peeves out of the way.  Async/Await in C# is a misnomer.  There is nothing asynchronous about it.  
The d asynchronous implies there is a time element, and async/await code does *not* alter the timing or flow of the code.  
It simply runs it on another thread.  Code running on another thread *can* run asynchronously, but async/await code does not do so.
It is completely synchronized so that unless you're not using the await keyword with the "async" function (which is actually quite a rare thing to do), each line of code is run sequentially (ie. *synchronously*) one line at a time.
However, that's the terminology they've used for the system, so from here on out, I will mostly ignore the reality that async is a misnomer.

This library provides a way to run async code in a synchronous context such that everything runs on the thread you've called it from, preventing any cross-thread issues, and allowing you to call async code from places where it's normally not allowed such as static initialization, LINQ, and overloads like ToString.

This code has been successfully used to slowly transition a 100K line production web server with hundreds of thousands of monthly users to async over a period of more than a year with only minor issues due to occasional mistakes in the conversion process.

1. Use async versions of framework and third-party code by calling the async version of the function in the empty delegate in AA.RunTaskSync or AA.RunSync instead of using await (see sample code).
2. Replace all use of thread-affine classes such as Mutex, ReaderWriterLock, Semaphore, ThreadLocal, etc. and constructs not allowed in an async-await context (lock) to their async/await-friendly equivalents (SemaphoreSlim, ReaderWriterLockSlim, SemaphoreSlim, AsyncLocal), using AA.RunTaskSync and AA.RunSync and the async versions of their APIs as appropriate.

This will cause the code to use the async API, but force it so run on the calling thread.
Next, one function at a time, starting in a function that is using AA.RunSync (for functions that return ValueTask) or AA.RunTaskSync (for functions that return Task),

1. Get a list of all callers to the function you are ready to make async-ready and find all callers (In Visual Studio, you can right-click the function and select "View Call Hierarchy").
2. Update the function signature to return Task or ValueTask and take a CancellationToken (if needed).  Use ValueTask unless you need to interact with other systems that don't support ValueTask, or if you need to await the result more than once (ValueTasks can only be awaited once).
3. Change all the calls to AA.RunTaskSync to "await AA.RunTask" and all calle to AA.RunSync to "await AA.Run"
4. Go to each of the callers and switch them to use AA.RunTaskSync or AA.RunSync, as above.
5. Repeat these steps until all instances of AA.RunTaskSync and AA.RunSync are gone.  At some point you'll get to the top of the stack where you'll need to figure out how to get the top-level function to be async.

Note that this process does not include switching to use IAsyncEnumerable<> and IAsyncDisposable.  
These changes can be made during the transition, but I would recommend making these changes after the above steps are complete, as these changes are much more complicated and will alter the flow of the code.

Once a top-level function is converted to async, everything below will automatically switch to run asynchronously, without any change to the code.  
(AA.RunSync sets the synchronization context to use a synchronous task scheduler, so if there are no instances of this up the call stack, that scheduler will not be used).
Once you're sure there are no synchronous callers firectly or indirectly calling a given function and you have no need to run any of the code synchronously, change "await AA.RunTask" and "await AA.Run" to just await like normal final-state async code.
The samples below show how this transition migh progress for a sample class.  Note that while we change the name of the class each time to indicate the progress of the transition, you would likely not do that.

### Piecemeal Conversion to Async/Await
[//]: # (AASample1)
```csharp
sealed class MySoonToBeAsyncClass : IDisposable
{
    private readonly Stream _file;
    /// <summary>
    /// Opens an output file.
    /// </summary>
    /// <param name="filename">The filename of the file to append to.</param>
    public MySoonToBeAsyncClass(string filename)
    {
        _file = new FileStream(filename, FileMode.Append, FileAccess.Write);
    }
    /// <summary>
    /// Writes the data into the file in UTF8 format.
    /// </summary>
    /// <param name="s"></param>
    public void WriteData(string s)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(s);
        _file.Write(buffer, 0, buffer.Length);
    }
    /// <summary>
    /// Flushes data to the file.
    /// </summary>
    public void Flush()
    {
        _file.Flush();
    }
    /// <summary>
    /// Disposes of the instance.
    /// </summary>
    public void Dispose()
    {
        _file.Dispose();
    }
}
```
[//]: # (AASample2)
```csharp
sealed class MyAlmostAsyncClass : IDisposable
{
    private readonly Stream _file;
    /// <summary>
    /// Opens an output file.
    /// </summary>
    /// <param name="filename">The filename of the file to append to.</param>
    public MyAlmostAsyncClass(string filename)
    {
        _file = new FileStream(filename, FileMode.Append, FileAccess.Write);
    }
    /// <summary>
    /// Writes the data into the file in UTF8 format.
    /// </summary>
    /// <param name="s"></param>
    public void WriteData(string s)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(s);
        AA.RunTaskSync(() => _file.WriteAsync(buffer, 0, buffer.Length));
    }
    /// <summary>
    /// Flushes data to the file.
    /// </summary>
    public void Flush()
    {
        AA.RunTaskSync(() => _file.FlushAsync());
    }
    /// <summary>
    /// Disposes of the instance.
    /// </summary>
    public void Dispose()
    {
        _file.Dispose();
    }
}
```
[//]: # (AASample3)
```csharp
sealed class MyAsyncReadyClass : IDisposable
{
    private readonly Stream _file;
    /// <summary>
    /// Opens an output file.
    /// </summary>
    /// <param name="filename">The filename of the file to append to.</param>
    public MyAsyncReadyClass(string filename)
    {
        _file = new FileStream(filename, FileMode.Append, FileAccess.Write);
    }
    /// <summary>
    /// Writes the data into the file in UTF8 format.
    /// </summary>
    /// <param name="s"></param>
    public async ValueTask WriteData(string s, CancellationToken cancel = default)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(s);
        await AA.RunTask(() => _file.WriteAsync(buffer, 0, buffer.Length, cancel));
    }
    /// <summary>
    /// Flushes data to the file.
    /// </summary>
    public async ValueTask Flush()
    {
        await AA.RunTask(() => _file.FlushAsync());
    }
    /// <summary>
    /// Disposes of the instance.
    /// </summary>
    public void Dispose()
    {
        _file.Dispose();
    }
}
```
[//]: # (AASample4)
```csharp
sealed class MyFullyAsyncClass : IDisposable
{
    private readonly Stream _file;
    /// <summary>
    /// Opens an output file.
    /// </summary>
    /// <param name="filename">The filename of the file to append to.</param>
    public MyFullyAsyncClass(string filename)
    {
        _file = new FileStream(filename, FileMode.Append, FileAccess.Write);
    }
    /// <summary>
    /// Writes the data into the file in UTF8 format.
    /// </summary>
    /// <param name="s"></param>
    public async ValueTask WriteData(string s, CancellationToken cancel = default)
    {
        byte[] buffer = Encoding.UTF8.GetBytes(s);
        await _file.WriteAsync(buffer, 0, buffer.Length, cancel);
    }
    /// <summary>
    /// Flushes data to the file.
    /// </summary>
    public async ValueTask Flush()
    {
        await _file.FlushAsync();
    }
    /// <summary>
    /// Disposes of the instance.
    /// </summary>
    public void Dispose()
    {
        _file.Dispose();
    }
}
```


## HighPerformanceFifoTaskScheduler
HighPerformanceFifoTaskScheduler is a high performance async task scheduler that is highly scalable and far more responsive than the standard .NET ThreadPool.

In my attempts to asyncify our codebase, I spent many man-weeks over at least six months attempting to use every imaginable invocation of the ThreadPool to spawn processes that we previously used a custom thread pool to run.
The results were underwhelming.  Even with simple test cases, I was unable to fully utilize the CPU on multi-core systems, and most of my attempts resulted in the ThreadPool going into a loop allocating threads and memory and making the system completely unresponsive, despite the CPU utilization remaining low most of the time.
When I did manage to get it to fully utilize the CPU for minutes at a time, I was never able to get from the starting state into such a state in less than a minute, and it usually took ten minutes or more, carefully watching numerous performance statistics in an attempt to avoid the non-responsive crazy thread creation loop.
When I switch to a real workload, which had a wider mix of tasks being CPU-bound, Memory-bound, and IO-bound, the system broke down again.
In addition to these reliability and performance issues, the system default ThreadPool doesn't process tasks in first-in first-out order, resulting in starvation for many tasks, making processing largely unpredictable.
The HighPerformanceFifoTaskScheduler provided here has none of these problems.  
My first test run pegged the CPU in less than ten seconds and kept it pegged with good system responsiveness indefinitely and low latency.
Here is a sample of how to do this using TaskFactory.StartNew:

### Usage Sample
[//]: # (HPFTS)
```csharp
/// <summary>
/// 
/// </summary>
[TestClass]
public class TestHighPerformanceFifoTaskScheduler
{
    [TestMethod]
    public void InvokeSingleFireAndForgetWork()
    {
        FakeWork w = new(-1, true);
        // fire and forget the work, discarding the returned task (it may not finish running until after the test is marked as successful--sometimes this is what you want, but usually not--we're just testing it here)
        _ = HighPerformanceFifoTaskFactory.Default.StartNew(() => w.DoMixedWorkAsync(CancellationToken.None).AsTask());
    }
    [TestMethod]
    public async Task InvokeSingleWorkItem()
    {
        FakeWork w = new(-2, true);
        // fire and forget the work, discarding the returned task (it may not finish running until after the test is marked as successful--sometimes this is what you want, but usually not--we're just testing it here)
        await HighPerformanceFifoTaskFactory.Default.StartNew(() => w.DoMixedWorkAsync(CancellationToken.None).AsTask());
    }
    [TestMethod]
    public void StartNew()
    {
        List<Task> tasks = new();
        for (int i = 0; i < 1000; ++i)
        {
            FakeWork w = new(i, true);
            // note the use of AsTask here because Task.WaitAll might await the resulting Task more than once (it probably doesn't, but just to be safe...)
            tasks.Add(HighPerformanceFifoTaskFactory.Default.StartNew(() => w.DoMixedWorkAsync(CancellationToken.None).AsTask()));
        }
        Task.WaitAll(tasks.ToArray());
        HighPerformanceFifoTaskScheduler.Default.Reset();
    }
}
public class FakeWork
{
    private readonly bool _fast;
    private readonly long _id;

    public FakeWork(long id, bool fast)
    {
        _fast = fast;
        _id = id;
    }

    public async ValueTask DoMixedWorkAsync(CancellationToken cancel = default)
    {
        ulong hash = GetHash(_id);
        await Task.Yield();
        //string? threadName = Thread.CurrentThread.Name;

        Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
        Stopwatch s = Stopwatch.StartNew();
        for (int outer = 0; outer < (int)(hash % 256) && !cancel.IsCancellationRequested; ++outer)
        {
            Stopwatch cpu = Stopwatch.StartNew();
            // use some CPU
            for (int spin = 0; spin < (int)((hash >> 6) % (_fast ? 16UL : 256UL)); ++spin)
            {
                double d1 = 0.0000000000000001;
                double d2 = 0.0000000000000001;
                for (int inner = 0; inner < (_fast ? 100 : 1000000); ++inner) { d2 = d1 * d2; }
            }
            cpu.Stop();
            Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
            Stopwatch mem = Stopwatch.StartNew();
            // use some memory
            int bytesPerLoop = (int)((hash >> 12) % (_fast ? 10UL : 1024UL));
            int loops = (int)((hash >> 22) % 1024);
            for (int memory = 0; memory < loops; ++memory)
            {
                byte[] bytes = new byte[bytesPerLoop];
            }
            mem.Stop();
            Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
            Stopwatch io = Stopwatch.StartNew();
            // simulate I/O by blocking
            await Task.Delay((int)((hash >> 32) % (_fast ? 5UL : 500UL)), cancel);
            io.Stop();
            Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
        }
        Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
        //Debug.WriteLine($"Ran work {_id} on {threadName}!", "Work");
    }
    public async ValueTask DoDelayOnlyWorkAsync(CancellationToken cancel = default)
    {
        ulong hash = GetHash(_id);
        await Task.Yield();
        //string? threadName = Thread.CurrentThread.Name;

        Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
        Stopwatch s = Stopwatch.StartNew();
        for (int outer = 0; outer < (int)(hash % 256) && !cancel.IsCancellationRequested; ++outer)
        {
            Stopwatch io = Stopwatch.StartNew();
            // simulate I/O by blocking
            await Task.Delay((int)((hash >> 32) % (_fast ? 5UL : 500UL)), cancel);
            io.Stop();
            Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
        }
        Assert.AreEqual(typeof(HighPerformanceFifoSynchronizationContext), SynchronizationContext.Current?.GetType());
        //Debug.WriteLine($"Ran work {_id} on {threadName}!", "Work");
    }
    private static ulong GetHash(long id)
    {
        unchecked
        {
            ulong x = (ulong)id * 1_111_111_111_111_111_111UL;        // note that this is a prime number (but not a mersenne prime)
            x = (((x & 0xaaaaaaaaaaaaaaaa) >> 1) | ((x & 0x5555555555555555) << 1));
            x = (((x & 0xcccccccccccccccc) >> 2) | ((x & 0x3333333333333333) << 2));
            x = (((x & 0xf0f0f0f0f0f0f0f0) >> 4) | ((x & 0x0f0f0f0f0f0f0f0f) << 4));
            x = (((x & 0xff00ff00ff00ff00) >> 8) | ((x & 0x00ff00ff00ff00ff) << 8));
            x = (((x & 0xffff0000ffff0000) >> 16) | ((x & 0x0000ffff0000ffff) << 16));
            return ((x >> 32) | (x << 32));
        }
    }
}
```

### Other notes on performance
Note that there are a number of other ways to invoke tasks asynchronously, but in my experience, TaskFactory.StartNew performs much better than any of the following:
	* new Task()
	* Task.Start
	* ThreadPool.UnsafeQueueUserWorkItem
	* indirect invocation using System.Timer


## Getting Started
In Visual Studio, use Manage Nuget Packages and search nuget.org for AmbientServices to add a package reference for this library.

For .NET Core environments, use:
`dotnet add package https://www.nuget.org/packages/AmbientServices.Async/`


## Miscellaneous
Some provided extension methods may conflict with existing extension methods, so those are put into the separate AmbientServices.Async.Extensions namespace so that they may be included only where needed.

# Library Information

## Author and License
AmbientServices is written and maintained by James Ivie.

AmbientServices is licensed under [MIT](https://opensource.org/licenses/MIT).

## Language and Tools
AmbientServices is written in C#, using .NET Standard 2.0, .NET Core 3.1, .NET 5.0, and .NET 6.0.  Unit tests are written in .NET 6.0.

The code can be built using either Microsoft Visual Studio 2022+, Microsoft Visual Studio Code, or .NET Core command-line utilities.

Binaries are available at https://www.nuget.org/packages/AmbientServices.Async.

## Contributions
Contributions are welcome under the following conditions:
1. enhancements are consistent with the overall scope of the project
2. no new assembly dependencies are introduced
3. code coverage by unit tests cover all new lines and conditions whenever possible
4. documentation (both inline and here) is updated appropriately
5. style for code and documentation contributions remains consistent

## Status
[![.NET](https://github.com/AmbientServices/AmbientServices.Async/actions/workflows/dotnet.yml/badge.svg)](https://github.com/AmbientServices/AmbientServices.Async/actions/workflows/dotnet.yml)