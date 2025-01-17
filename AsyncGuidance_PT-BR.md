# Table of contents
 - [Programação assícrona](#programação-assíncrona)
   - [Assincronia é viral](#assincronia-é-viral)
   - [Async void](#async-void)
   - [Prefira Task.FromResult ao invés de Task.Run para dados pré-computados ou facilmente computados](#prefer-taskfromresult-over-taskrun-for-pre-computed-or-trivially-computed-data)
   - [Evite usar Task.Run para tarefas que executem por muito tempo e bloqueiem a thread](#avoid-using-taskrun-for-long-running-work-that-blocks-the-thread)
   - [Evite usar Task.Result e Task.Wait](#avoid-using-taskresult-and-taskwait)
   - [Prefira await ao invés de ContinueWith](#prefer-await-over-continuewith)
   - [Sempre crie TaskCompletionSource\<T\> com TaskCreationOptions.RunContinuationsAsynchronously](#always-create-taskcompletionsourcet-with-taskcreationoptionsruncontinuationsasynchronously)
   - [Sempre descarte os CancellationTokenSource(s) used for timeouts](#always-dispose-cancellationtokensources-used-for-timeouts)
   - [Sempre passe CancellationToken(s) para APIs que aceitem CancellationToken](#always-flow-cancellationtokens-to-apis-that-take-a-cancellationtoken)
   - [Cancelando operações não-canceláveis](#cancelling-uncancellable-operations)
   - [Sempre use FlushAsync em StreamWriter(s) ou Stream(s) antes de chamar Dispose](#always-call-flushasync-on-streamwriters-or-streams-before-calling-dispose)
   - [Prefira async/await ao invés de retornar diretamente uma Task](#prefer-asyncawait-over-directly-returning-task)
   - [AsyncLocal\<T\>](#asynclocalt)
   - [ConfigureAwait](#configureawait)
 - [Cenários](#scenarios)
   - [Timer callbacks](#timer-callbacks)
   - [Implicit async void delegates](#implicit-async-void-delegates)
   - [ConcurrentDictionary.GetOrAdd](#concurrentdictionarygetoradd)
   - [Constructors](#constructors)
   - [WindowsIdentity.RunImpersonated](#windowsidentityrunimpersonated)
 
# Programação assíncrona

Programação assíncrona existe a muitos anos na plataforma .NET, porém históricamente é muito difícil de se fazer bem feito. Desde a chegada do async/await no C# 5, programação assíncrona se tornou convencional. Frameworks modernos (como ASP.NET Core) são completamente assíncronos e é quase impossível não usar async/await ao escrever web services.
Por causa disso, há muita incerteza com relação às melhores práticas para async e como utilizar o recurso apropriadamente. Esse texto tentará trazer alguns fundamentos com exemplos do que fazer e do que não fazer ao escrever código assíncrono.


## Assincronia é viral 

Uma vez que você utiliza chamadas assíncronas, todas as suas chamadas **DEVEM** ser assíncronas. Caso contrário, o esforço para usar async resulta em nada.
Em muitos casos, utilizar async parcialmente pode ser pior do que realizar todas as chamadas de forma síncrona. Por isso, é melhor se comprometer com a mudança e fazer com que tudo seja assíncrono de uma vez só.


❌ **NÃO FAÇA** Esse exemplo utiliza `Task.Result` e por causa disso bloqueia a thread aguardando o retorno do resultado. Esse é um exemplo de [sync over async](#avoid-using-taskresult-and-taskwait).

```C#
public int DoSomethingAsync()
{
    var result = CallDependencyAsync().Result;
    return result + 1;
}
```

:white_check_mark: **FAÇA** Esse exemplo utiliza a palavra chave async para obter o resultado do método `CallDependencyAsync`.

```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

## Async void

O uso de `async void` em uma aplicação ASP.NET Core é **SEMPRE** ruim. Evite ou nunca faça. Normalmente, é usada quando os desenvolvedores estão tentando implementar o padrão "fire and forget" de algo que é ativado com uma chamada em um controller.
Métodos async void vão quebrar a aplicação caso uma exceção seja lançada.

Veremos alguns padrões que levam desenvolvedores a cometer esse erro, mas aqui vai um exemplo simples:

❌ **NÃO FAÇA** Métodos async void não são rastreados e portanto, exceções não tratadas podem resultar no crash da aplicação.

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        BackgroundOperationAsync();
        return Accepted();
    }
    
    public async void BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        DoSomething(result);
    }
}
```

:white_check_mark: **FAÇA** Métodos que retornam `Task` são melhores pois exceções não tratadas vão ativar [`TaskScheduler.UnobservedTaskException`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler.unobservedtaskexception?view=netframework-4.7.2).

```C#
public class MyController : Controller
{
    [HttpPost("/start")]
    public IActionResult Post()
    {
        Task.Run(BackgroundOperationAsync);
        return Accepted();
    }
    
    public async Task BackgroundOperationAsync()
    {
        var result = await CallDependencyAsync();
        DoSomething(result);
    }
}
```

## Prefira `Task.FromResult` ao invés de `Task.Run` para valores pré-calculados ou com cálculos simples

Para resultados pré-computados não há necessidade de utilizar `Task.Run`. Esse método irá enfileirar um item na thread pool que irá completar imediatamente com o valor pré-computado. Ao invés disso, use `Task.FromResult` para criar uma Task em volta dos dados já computados.

❌ **NÃO FAÇA** Esse exemplo desperdiça uma thread pra retornar um dado computado trivialmente.

```C#
public class MyLibrary
{
   public Task<int> AddAsync(int a, int b)
   {
       return Task.Run(() => a + b);
   }
}
```

:white_check_mark: **FAÇA** Esse exemplo utiliza `Task.FromResult` para retornar dados facilmente computados e não utiliza nenhuma thread extra no processo.

```C#
public class MyLibrary
{
   public Task<int> AddAsync(int a, int b)
   {
       return Task.FromResult(a + b);
   }
}
```

:bulb:**NOTA: Usar `Task.FromResult` vai resultar na alocação de uma `Task`. Para evitar isso, pode-se usar `ValueTask<T>`.**

:white_check_mark: **BOM** Esse exemplo utiliza `ValueTask<int>` para retornar um valor que foi computado de forma trivial. Por causa disso, não utiliza nenhuma thread extra e também não aloca um objeto no heap gerenciado.

```C#
public class MyLibrary
{
   public ValueTask<int> AddAsync(int a, int b)
   {
       return new ValueTask<int>(a + b);
   }
}
```

## Evite usar Task.Run para tarefas que rodem por muito tempo e bloqueiem a thread

Tarefas de longa duração, nesse contexto, se refere a tarefas que rodam em background durante o ciclo de vida da aplicação (como processar itens de filas, ou "dormir" e "acordar" para processar algum dado). `Task.Run` vai enfileirar um item na pool de threads. Aqui assumimos que um processo será finalizado rapidamente (ou rápido o suficiente para permitir a reutilização da thread em um tempo razoável).
Utilizar uma thread da thread-pool nesse contexto de tarefas de longa duração é ruim pois vai impedir que essa thread seja usada para realizar outras tarefas (callbacks de timers, continuação de tasks etc). Ao invés disso, inicie uma nova thread manualmente para realizar essas tarefas de longa duração.

:bulb: **NOTA: A thread pool aumenta se você bloquear threads, mas isso é uma má prática.**

:bulb: **NOTA:`Task.Factory.StartNew` tem uma opção `TaskCreationOptions.LongRunning` que por baixo dos panos cria uma nova thread e retorna a Task que representa a execução. Utilizar essa feature apropriadamente requer diversos parâmetros que não são óbvios para que o comportamento seja correto em todas as plataformas.**

:bulb: **NOTA: Não use `TaskCreationOptions.LongRunning` com código assíncrono pois isso irá criar uma nova thread que será destruída após o primeiro `await`.**


❌ **NÃO FAÇA** Esse exemplo "rouba" uma thread da thread-pool para sempre, para executar trabalho enfileirado em uma `BlockingCollection<T>`.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new BlockingCollection<Message>();
    
    public void StartProcessing()
    {
        Task.Run(ProcessQueue);
    }
    
    public void Enqueue(Message message)
    {
        _messageQueue.Add(message);
    }
    
    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
             ProcessItem(item);
        }
    }
    
    private void ProcessItem(Message message) { }
}
```

:white_check_mark: **FAÇA** Esse exemplo usa uma thread dedicada para processar a fila de mensagens ao invés de usar uma thread da thread-pool.

```C#
public class QueueProcessor
{
    private readonly BlockingCollection<Message> _messageQueue = new BlockingCollection<Message>();
    
    public void StartProcessing()
    {
        var thread = new Thread(ProcessQueue) 
        {
            // Isso é importante pois permite o processo ser finalizado enquanto essa thread ainda está rodando.
            IsBackground = true
        };
        thread.Start();
    }
    
    public void Enqueue(Message message)
    {
        _messageQueue.Add(message);
    }
    
    private void ProcessQueue()
    {
        foreach (var item in _messageQueue.GetConsumingEnumerable())
        {
             ProcessItem(item);
        }
    }
    
    private void ProcessItem(Message message) { }
}
```

## Evite usar `Task.Result` e `Task.Wait`

Existem poucas maneiras de usar `Task.Result` e `Task.Wait` corretamente, então, geralmente o melhor é não usar.

### :warning: Síncrono ao invés de `async`

Usar `Task.Result` ou `Task.Wait` pra bloquear e aguardar o retorno de uma operação assíncriona é *MUITO* pior do que chamar uma API assíncrona de verdade. Esse fenômeno é chamado de "Sync over async". Isso é o que acontece:

- Uma operação assíncrona é iniciada.
- A thread que está chamado a operação é bloqueada aguardando que a operação termine.
- Quando a operação termina, o código que estava aguardando a execução é "desbloqueado" e isso acontece em OUTRA thread.

Resultado: precisamos de 2 threads ao invés de 1 pra realizar uma operação síncrona. Isso normalmente gera um fenômeno chamado [thread-pool starvation](https://blogs.msdn.microsoft.com/vancem/2018/10/16/diagnosing-net-core-threadpool-starvation-with-perfview-why-my-service-is-not-saturating-all-cores-or-seems-to-stall/) e isso pode resultar em interrupções no serviço.

### :warning: Deadlocks

A classe `SynchronizationContext` é uma abstração que permite aos `ApplicationModels` a possibilidade de controlar onde as 
continuações assíncronas rodam. [Leia mais sobre ApplicationModels](https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/application-model?view=aspnetcore-7.0).
ASP.NET, WPF e Winforms tem implementações únicas que vão resultar em um deadlock se Task.Wait ou Task.Result forem utilizados na thread principal.
Esse comportamento resultou em alguns code snippets que mostram o jeito "certo" de aguardar uma task bloqueante. Na verdade, não existe uma boa forma de aguardar uma task bloqueante.

:bulb:**NOTA: ASP.NET Core não tem `SynchronizationContext` e não é propenso a deadlocks.**

❌ **NÃO FAÇA** Abaixo todos os exemplos estão, de uma forma ou de outra, tentando evitar o deadlock mas ainda sofrem de problemas de "sync over async".

```C#
public string DoOperationBlocking()
{
    // Ruim - Bloqueando a thread.
    // DoAsyncOperation vai ser agendado no agendador padrão de tarefa e retirar o risco de deadlock.
    // No caso de uma exceção, o método irá lançar uma AggregateException em volta da exception original.
    return Task.Run(() => DoAsyncOperation()).Result;
}

public string DoOperationBlocking2()
{
    // Ruim - Bloqueando a thread.
    // DoAsyncOperation vai ser agendado no agendador padrão de tarefa e retirar o risco de deadlock.
    // No caso de uma exceção, esse método irá lançar a exceção sem envolve-la por uma AggregateException.
    return Task.Run(() => DoAsyncOperation()).GetAwaiter().GetResult();
}

public string DoOperationBlocking3()
{
    // Ruim - Bloqueando a thread, e bloqueando a thread do threadpool.
    // No caso de uma exceção, o método irá lançar uma AggregateException em volta de outra AggragateException que contém a exception original.
    return Task.Run(() => DoAsyncOperation().Result).Result;
}

public string DoOperationBlocking4()
{
    // Ruim - Bloqueando a thread, e bloqueando a thread do threadpool.
    return Task.Run(() => DoAsyncOperation().GetAwaiter().GetResult()).GetAwaiter().GetResult();
}

public string DoOperationBlocking5()
{
    // Ruim - Bloqueando a thread.
    // Ruim - Nesse caso, não existe nada que impeça SynchonizationContext de sofrer um deadlock.
    // No caso de uma exceção, o método irá lançar uma AggregateException em volta da exception original.
    return DoAsyncOperation().Result;
}

public string DoOperationBlocking6()
{
    // Ruim - Bloqueando a thread.
    // Ruim - Nesse caso, não existe nada que impeça SynchonizationContext de sofrer um deadlock.
    return DoAsyncOperation().GetAwaiter().GetResult();
}

public string DoOperationBlocking7()
{
    // Ruim - Bloqueando a thread.
    // Ruim - Nesse caso, não existe nada que impeça SynchonizationContext de sofrer um deadlock.
    var task = DoAsyncOperation();
    task.Wait();
    return task.GetAwaiter().GetResult();
}
```

## Prefira `await` ao invés de `ContinueWith`

`Task` existe antes mesmo da introdução das keywords async/await e por si só provê formas de continuar execuções sem depender da linguagem. Apesar de que esses métodos ainda são válidos, normalmente nós recomendamos que você prefira usar `async`/`await` ao invés `ContinueWith`. `ContinueWith` não captura o `SyncronizationContext` e por isso é semânticamente diferente de async/await.

❌ **NÃO FAÇA** Esse exemplo usa `ContinueWith` ao invés de `async`

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync().ContinueWith(task =>
    {
        return task.Result + 1;
    });
}
```

:white_check_mark: **FAÇA** Esse exemplo usa `await` para obter o resultado de `CallDependencyAsync`.

```C#
public async Task<int> DoSomethingAsync()
{
    var result = await CallDependencyAsync();
    return result + 1;
}
```

## Always create `TaskCompletionSource<T>` with `TaskCreationOptions.RunContinuationsAsynchronously`

`TaskCompletionSource<T>` is an important building block for libraries trying to adapt things that are not inherently awaitable to be awaitable via a `Task`. It is also commonly used to build higher-level operations (such as batching and other combinators) on top of existing asynchronous APIs. By default, `Task` continuations will run *inline* on the same thread that calls Try/Set(Result/Exception/Canceled). As a library author, this means having to understand that calling code can resume directly on your thread. This is extremely dangerous and can result in deadlocks, thread-pool starvation, corruption of state (if code runs unexpectedly) and more. 

Always use `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`. This will dispatch the continuation onto the thread pool instead of executing it inline.

❌ **BAD** This example does not use `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`.

```C#
public Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>();
    
    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Code awaiting on this task will resume on this thread!
        tcs.SetResult(result);
    };
    
    return tcs.Task;
}
```

:white_check_mark: **GOOD** This example uses `TaskCreationOptions.RunContinuationsAsynchronously` when creating the `TaskCompletionSource<T>`.

```C#
public Task<int> DoSomethingAsync()
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
    
    var operation = new LegacyAsyncOperation();
    operation.Completed += result =>
    {
        // Code awaiting on this task will resume on a different thread-pool thread
        tcs.SetResult(result);
    };
    
    return tcs.Task;
}
```

:bulb:**NOTE: There are 2 enums that look alike. [`TaskCreationOptions.RunContinuationsAsynchronously`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcreationoptions?view=netcore-2.0#System_Threading_Tasks_TaskCreationOptions_RunContinuationsAsynchronously) and [`TaskContinuationOptions.RunContinuationsAsynchronously`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskcontinuationoptions?view=netcore-2.0). Be careful not to confuse their usage.** 

## Always dispose `CancellationTokenSource`(s) used for timeouts

`CancellationTokenSource` objects that are used for timeouts (are created with timers or uses the `CancelAfter` method), can put pressure on the timer queue if not disposed.

❌ **BAD** This example does not dispose the `CancellationTokenSource` and as a result the timer stays in the queue for 10 seconds after each request is made.

```C#
public async Task<Stream> HttpClientAsyncWithCancellationBad()
{
    var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));

    using (var client = _httpClientFactory.CreateClient())
    {
        var response = await client.GetAsync("http://backend/api/1", cts.Token);
        return await response.Content.ReadAsStreamAsync();
    }
}
```

:white_check_mark: **GOOD** This example disposes the `CancellationTokenSource` and properly removes the timer from the queue.

```C#
public async Task<Stream> HttpClientAsyncWithCancellationGood()
{
    using (var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)))
    {
        using (var client = _httpClientFactory.CreateClient())
        {
            var response = await client.GetAsync("http://backend/api/1", cts.Token);
            return await response.Content.ReadAsStreamAsync();
        }
    }
}
```

## Always flow `CancellationToken`(s) to APIs that take a `CancellationToken`

Cancellation is cooperative in .NET. Everything in the call-chain has to be explicitly passed the `CancellationToken` in order for it to work well. This means you need to explicitly pass the token into other APIs that take a token if you want cancellation to be most effective.

❌ **BAD** This example neglects to pass the `CancellationToken` to `Stream.ReadAsync` making the operation effectively not cancellable.

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
   byte[] buffer = new byte[1024];
   // We forgot to pass flow cancellationToken to ReadAsync
   int read = await _stream.ReadAsync(buffer, 0, buffer.Length);
   return Encoding.UTF8.GetString(buffer, 0, read);
}
```

:white_check_mark: **GOOD** This example passes the `CancellationToken` into `Stream.ReadAsync`.

```C#
public async Task<string> DoAsyncThing(CancellationToken cancellationToken = default)
{
   byte[] buffer = new byte[1024];
   // This properly flows cancellationToken to ReadAsync
   int read = await _stream.ReadAsync(buffer, 0, buffer.Length, cancellationToken);
   return Encoding.UTF8.GetString(buffer, 0, read);
}
```

## Cancelling uncancellable operations

One of the coding patterns that appears when doing asynchronous programming is cancelling an uncancellable operation. This usually means creating another task that completes when a timeout or `CancellationToken` fires, and then using `Task.WhenAny` to detect a complete or cancelled operation.

### Using CancellationTokens

❌ **BAD** This example uses `Task.Delay(-1, token)` to create a `Task` that completes when the `CancellationToken` fires, but if it doesn't fire, there's no way to dispose the `CancellationTokenRegistration`. This can lead to a memory leak.

```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    // There's no way to dispose the registration
    var delayTask = Task.Delay(-1, cancellationToken);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        // Operation cancelled
        throw new OperationCanceledException();
    }

    return await task;
}
```

:white_check_mark: **GOOD** This example disposes the `CancellationTokenRegistration` when one of the `Task(s)` complete.

```C#
public static async Task<T> WithCancellation<T>(this Task<T> task, CancellationToken cancellationToken)
{
    var tcs = new TaskCompletionSource<object>(TaskCreationOptions.RunContinuationsAsynchronously);

    // This disposes the registration as soon as one of the tasks trigger
    using (cancellationToken.Register(state =>
    {
        ((TaskCompletionSource<object>)state).TrySetResult(null);
    },
    tcs))
    {
        var resultTask = await Task.WhenAny(task, tcs.Task);
        if (resultTask == tcs.Task)
        {
            // Operation cancelled
            throw new OperationCanceledException(cancellationToken);
        }

        return await task;
    }
}
```

:white_check_mark: **GOOD** Prefer [`Task.WaitAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.waitasync?view=net-6.0) on .NET >= 6;

### Using a timeout

❌ **BAD** This example does not cancel the timer even if the operation successfully completes. This means you could end up with lots of timers, which can flood the timer queue. 

```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    var delayTask = Task.Delay(timeout);

    var resultTask = await Task.WhenAny(task, delayTask);
    if (resultTask == delayTask)
    {
        // Operation cancelled
        throw new OperationCanceledException();
    }

    return await task;
}
```

:white_check_mark: **GOOD** This example cancels the timer if the operation successfully completes.

```C#
public static async Task<T> TimeoutAfter<T>(this Task<T> task, TimeSpan timeout)
{
    using (var cts = new CancellationTokenSource())
    {
        var delayTask = Task.Delay(timeout, cts.Token);

        var resultTask = await Task.WhenAny(task, delayTask);
        if (resultTask == delayTask)
        {
            // Operation cancelled
            throw new OperationCanceledException();
        }
        else
        {
            // Cancel the timer task so that it does not fire
            cts.Cancel();
        }

        return await task;
    }
}
```

:white_check_mark: **GOOD** Prefer [`Task.WaitAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.task.waitasync?view=net-6.0) on .NET >= 6;

## Always call `FlushAsync` on `StreamWriter`(s) or `Stream`(s) before calling `Dispose`

When writing to a `Stream` or `StreamWriter`, even if the asynchronous overloads are used for writing, the underlying data might be buffered. When data is buffered, disposing the `Stream` or `StreamWriter` via the `Dispose` method will synchronously write/flush, which results in blocking the thread and could lead to thread-pool starvation. Either use the asynchronous `DisposeAsync` method (for example via `await using`) or call `FlushAsync` before calling `Dispose`.

:bulb:**NOTE: This is only problematic if the underlying subsystem does IO.**

❌ **BAD** This example ends up blocking the request by writing synchronously to the HTTP-response body.

```C#
app.Run(async context =>
{
    // The implicit Dispose call will synchronously write to the response body
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
    }
});
```

:white_check_mark: **GOOD** This example asynchronously flushes any buffered data while disposing the `StreamWriter`.

```C#
app.Run(async context =>
{
    // The implicit AsyncDispose call will flush asynchronously
    await using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");
    }
});
```

:white_check_mark: **GOOD** This example asynchronously flushes any buffered data before disposing the `StreamWriter`.

```C#
app.Run(async context =>
{
    using (var streamWriter = new StreamWriter(context.Response.Body))
    {
        await streamWriter.WriteAsync("Hello World");

        // Force an asynchronous flush
        await streamWriter.FlushAsync();
    }
});
```

## Prefer `async`/`await` over directly returning `Task`

There are benefits to using the `async`/`await` keyword instead of directly returning the `Task`:
- Asynchronous and synchronous exceptions are normalized to always be asynchronous.
- The code is easier to modify (consider adding a `using`, for example).
- Diagnostics of asynchronous methods are easier (debugging hangs etc).
- Exceptions thrown will be automatically wrapped in the returned `Task` instead of surprising the caller with an actual exception.
- Async locals will not leak out of async methods. If you set an async local in a non-async method, it will "leak" out of that call.

❌ **BAD** This example directly returns the `Task` to the caller.

```C#
public Task<int> DoSomethingAsync()
{
    return CallDependencyAsync();
}
```

:white_check_mark: **GOOD** This examples uses async/await instead of directly returning the Task.

```C#
public async Task<int> DoSomethingAsync()
{
    return await CallDependencyAsync();
}
```

:bulb:**NOTE: There are performance considerations when using an async state machine over directly returning the `Task`. It's always faster to directly return the `Task` since it does less work but you end up changing the behavior and potentially losing some of the benefits of the async state machine.**

## AsyncLocal\<T\>

Async locals are a way to store/retrieve ambient state throughout an application. This can be a *very* useful alternative to flowing explicit state everywhere, especially through call sites that you do not have much control over. While it is powerful, it is also dangerous if used incorrectly. Async locals are attached to the [execution context](https://docs.microsoft.com/en-us/dotnet/api/system.threading.executioncontext) which flows *everywhere implicitly*. Disabling execution context flow requires use of advanced APIs (typically prefixed with the Unsafe name). As such, there's very little control over what code will attempt to access these values. 

### Creating an AsyncLocal\<T\>

If you can avoid async locals, do so by explicitly passing state around or using techniques like inversion of control.

If you cannot avoid it, it's best to make sure that anything put into an async local is:

1. Not disposable
2. Immutable/read only/thread safe

Lets look at 2 examples:

1. ❌ **BAD** A disposable object stored in an async local

```C#
using (var thing = new DisposableThing())
{
    // Make the disposable object available ambiently
    DisposableThing.Current = thing;

    Dispatch();

    // We're about to dispose the object so make sure nobody else captures this instance
    DisposableThing.Current = null;
}

void Dispatch()
{
    // Task.Run will capture the current execution context (which means async locals are captured in the callback)
    _ = Task.Run(async () =>
    {
        // Delay for a second then log
        await Task.Delay(1000);

        Log();
    });
}

void Log()
{
    try
    {
        // Get the current value and make sure it's not null before reading the value
        var thing = DisposableThing.Current;
        if (thing is not null)
        {
            Console.WriteLine($"Logging ambient value {thing.Value}");
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex);
    }
}

Console.ReadLine();

class DisposableThing : IDisposable
{
    private static readonly AsyncLocal<DisposableThing?> _current = new();

    private bool _disposed;

    public static DisposableThing? Current
    {
        get => _current.Value;
        set
        {
            _current.Value = value;
        }
    }

    public int Value
    {
        get
        {
            if (_disposed) throw new ObjectDisposedException(GetType().FullName);
            return 1;
        }
    }

    public void Dispose()
    {
        _disposed = true;
    }
}
```

This above example will always result in an `ObjectDisposedException` being thrown. Even though the `Log` method defensively checks for null before logging the value, it has a reference to the disposed `DisposableThing`. Setting the `AsyncLocal<DisposableThing>` to null does not affect the code inside of `Log`, this is because the execution context is copy on write. This means that all future reads `DisposableThing.Current` will be null, but it won't affect any of the previous reads.

When we set `DisposableThing.Current = null;` we are making a new execution context, not mutating the one that was captured by `Task.Run`. To get a better understanding of this run the following code:

```C#
DisposableThing.Current = new DisposableThing();

Console.WriteLine("After setting thing " + ExecutionContext.Capture().GetHashCode());

DisposableThing.Current = null;

Console.WriteLine("After setting Current to null " + ExecutionContext.Capture().GetHashCode());
```

The hash code of the execution context is different each time we set a new value.

⚠️ It might be tempting to update the logic in `DisposableThing.Current` to mutate the original execution context instead of setting the async local directly ([StrongBox\<T\>](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.strongbox-1) is a reference type that stores the underlying T in a mutable field):

```C#
class DisposableThing : IDisposable
{
    private static readonly AsyncLocal<StrongBox<DisposableThing?>> _current = new();

    private bool _disposed;

    public static DisposableThing? Current
    {
        get => _current.Value?.Value;
        set
        {
            var box = _current.Value;
            if (box is not null)
            {
                // Mutate the value in any execution context that was copied
                box.Value = null;
            }

            if (value is not null)
            {
                _current.Value = new StrongBox<DisposableThing?>(value);
            }
        }
    }

    public int Value
    {
        get
        {
            if (_disposed) throw new ObjectDisposedException(GetType().FullName);
            return 1;
        }
    }

    public void Dispose()
    {
        _disposed = true;
    }
}
```

This will have the desired effect and will set the value to null in any execution context that references this async local value.

```C#
DisposableThing.Current = new DisposableThing();

Console.WriteLine("After setting thing " + ExecutionContext.Capture().GetHashCode());

DisposableThing.Current = null;

Console.WriteLine("After setting Current to null " + ExecutionContext.Capture().GetHashCode());
```

⚠️ While this looks attractive, the reference to `DisposableThing.Current` might have still been captured before the value was set to null:

```C#
void Dispatch()
{
    // Task.Run will capture the current execution context (which means async locals are captured in the callback)
    _ = Task.Run(async () =>
    {
        // Get the current reference
        var current = DisposableThing.Current;

        // Delay for a second then log
        await Task.Delay(1000);

        Log(current);
    });
}

void Log(DisposableThing thing)
{
    try
    {
        Console.WriteLine($"Logging ambient value {thing.Value}");
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex);
    }
}
```

There's a race condition between the capture of the `DisposableThing`, the disposal of `DisposableThing` and setting `DisposableThing.Current` it to null. In the end, the code is unreliable and may fail at random. Don't store disposable objects in async locals.

2. ❌ **BAD** A non-thread safe object stored in an async local

```C#
AmbientValues.Current = new Dictionary<int, string>();

Parallel.For(0, 10, i =>
{
    AmbientValues.Current[i] = "processing";
    LogCurrentValues();
    AmbientValues.Current[i] = "done";
});


void LogCurrentValues()
{
    foreach (var pair in AmbientValues.Current)
    {
        Console.WriteLine(pair);
    }
}

class AmbientValues
{
    private static readonly AsyncLocal<Dictionary<int, string>> _current = new();

    public static Dictionary<int, string> Current
    {
        get => _current.Value!;
        set => _current.Value = value;
    }
}
```

The above example stores a normal `Dictionary<int, string>` in an async local and does some parallel processing on it. While this may be obvious from the above example, async locals allow arbitrary code on arbitrary threads to access the execution context and thus any async locals associated with said context. As a result, it is important to assume that data can be accessed concurrently and should be made thread safe as a result.

```C#
class AmbientValues
{
    private static readonly AsyncLocal<ConcurrentDictionary<int, string>> _current = new();

    public static ConcurrentDictionary<int, string> Current
    {
        get => _current.Value!;
        set => _current.Value = value;
    }
}
```

:white_check_mark: **GOOD** The above uses a `ConcurrentDictionary<int, string>` which is thread safe.

### Don't leak your AsyncLocal\<T\>

Async locals flow across awaits automatically and can be captured by any API that explicitly calls `ExecutionContext.Capture`. The latter can lead to memory leaks in certain situations.

#### Common APIs that capture the ExecutionContext

APIs that run user callbacks usually capture the current execution context in order to preserve async locals between callback registration and execution. Here are examples of some APIs that do this:

- `Timer`
- `CancellationToken.Register`
- `new FileSystemWatcher`
- `SocketAsyncEventArgs`
- `Task.Run`
- `ThreadPool.QueueUserWorkItem`

❌ **BAD** Here's an example of an execution context leak that causes memory pressure because of a lifetime mismatch between the API capturing the execution context, and the lifetime of the data stored in the async local.

```C#
using System.Collections.Concurrent;

// Singleton cache
var cache = new NumberCache(TimeSpan.FromHours(1));

var executionContext = ExecutionContext.Capture();

// Simulate 10000 concurrent requests
Parallel.For(0, 10000, i =>
{
    // Restore the initial ExecutionContext per "request"
    ExecutionContext.Restore(executionContext!);

    ChunkyObject.Current = new ChunkyObject();

    cache.Add(i);
});

Console.WriteLine("Before GC: " + BytesAsString(GC.GetGCMemoryInfo().HeapSizeBytes));
Console.ReadLine();

GC.Collect();
GC.WaitForPendingFinalizers();

Console.WriteLine("After GC: " + BytesAsString(GC.GetGCMemoryInfo().HeapSizeBytes));
Console.ReadLine();

static string BytesAsString(long bytes)
{
    string[] suffix = { "B", "KB", "MB", "GB", "TB" };
    int i;
    double doubleBytes = 0;

    for (i = 0; bytes / 1024 > 0; i++, bytes /= 1024)
    {
        doubleBytes = bytes / 1024.0;
    }

    return string.Format("{0:0.00} {1}", doubleBytes, suffix[i]);
}

public class NumberCache
{
    private readonly ConcurrentDictionary<int, CancellationTokenSource> _cache = new ConcurrentDictionary<int, CancellationTokenSource>();
    private TimeSpan _timeSpan;

    public NumberCache(TimeSpan timeSpan)
    {
        _timeSpan = timeSpan;
    }

    public void Add(int key)
    {
        var cts = _cache.GetOrAdd(key, _ => new CancellationTokenSource());
        // Delete entry on expiration
        cts.Token.Register((_, _) => _cache.TryRemove(key, out _), null);

        // Start count down
        cts.CancelAfter(_timeSpan);
    }
}

class ChunkyObject
{
    private static readonly AsyncLocal<ChunkyObject?> _current = new();

    // Stores lots of data (but it should be gen0)
    private readonly string _data = new string('A', 1024 * 32);

    public static ChunkyObject? Current
    {
        get => _current.Value;
        set => _current.Value = value;
    }

    public string Data => _data;
}
```

The above example has a singleton `NumberCache` that stores numbers for an hour. We have a `ChunkyObject` which stores a 32K string in a field, and has an async local so that any code running may access the current `ChunkyObject`. This object should be collected when the `GC` runs, but instead, we're implicitly capturing the `ChunkyObject` in the `NumberCache` via `CancellationToken.Register`. 

**Instead of just caching the number and a `CancellationTokenSource`, we're implicitly capturing and storing all async locals attached to the current execution context for an hour!**

Try running the sample locally. Running this on my machine reports numbers like this:

```
Before GC: 654.65 MB
After GC: 659.68 MB
```

Here's a look at the heap with those objects. You can see we have stored 10,000 ChunkyObjects, strings rooted by those chunky objects. The object graph looks like
CancellationTokenSource -> ExecutionContext -> AsyncLocalValueMap -> ChunkObject -> string.

<img width="758" alt="image" src="https://user-images.githubusercontent.com/95136/188351756-967f3d37-b302-49d3-ba04-595433c6949c.png">


With one small tweak to this code, we can avoid the implicit execution context capture.

:white_check_mark: **GOOD** Use `CancellationToken.UnsafeRegister` to avoid capturing the execution context and any async locals as part of the `NumberCache`:

```C#
public class NumberCache
{
    private readonly ConcurrentDictionary<int, CancellationTokenSource> _cache = new ConcurrentDictionary<int, CancellationTokenSource>();
    private TimeSpan _timeSpan;

    public NumberCache(TimeSpan timeSpan)
    {
        _timeSpan = timeSpan;
    }

    public void Add(int key)
    {
        var cts = _cache.GetOrAdd(key, _ => new CancellationTokenSource());
        // Delete entry on expiration
        cts.Token.UnsafeRegister((_, _) => _cache.TryRemove(key, out _), null);

        // Start count down
        cts.CancelAfter(_timeSpan);
    }
}
```

The GC numbers after this change:

```
Before GC: 10.32 MB
After GC: 5.10 MB
```

The heap looks like we'd expect. There's no execution context capture, so the `ChunkyObject` isn't stored.

<img width="752" alt="image" src="https://user-images.githubusercontent.com/95136/188352462-d7d627c6-e4e0-4487-b783-30880cc4916f.png">


:bulb: **NOTE: You have NO control over how APIs decide to store the execution context, but with this understanding, you should be able to minimize memory leaks by clearing the memory using the technique described in [Creating an AsyncLocal\<T\>](#creating-an-asynclocalt) section.**

```C#
using System.Collections.Concurrent;

// Singleton cache
var cache = new NumberCache(TimeSpan.FromHours(1));

var executionContext = ExecutionContext.Capture();

// Simulate 10000 concurrent requests
Parallel.For(0, 10000, i =>
{
    // Restore the initial ExecutionContext per "request"
    ExecutionContext.Restore(executionContext!);

    ChunkyObject.Current = new ChunkyObject();

    cache.Add(i);

    // Null out the chunky object so the GC can release the memory
    ChunkyObject.Current = default;
});

class ChunkyObject
{
    private static readonly AsyncLocal<StrongBox<ChunkyObject?>> _current = new();

    // Stores lots of data (but it should be gen0)
    private readonly string _data = new string('A', 1024 * 32);

    public static ChunkyObject? Current
    {
        get => _current.Value?.Value;
        set
        {
            var box = _current.Value;
            if (box is not null)
            {
                // Mutate the value in any execution context that was copied
                box.Value = null;
            }

            if (value is not null)
            {
                _current.Value = new StrongBox<ChunkyObject?>(value);
            }
        }
    }

    public string Data => _data;
}
```

This technique reduces the heap memory **significantly**:

```
Before GC: 7.91 MB
After GC: 5.66 MB
```

The execution context is storing `StrongBox<ChunkyObject>` with a null reference to the `ChunkyObject`. This is technically still a "leak" but we've reduced the impact significantly. Here's a look at the memory profile showing objects with 10,000 allocations (the number of requests we created). You can see the GC has collected `ChunkObject` instances but there are still 10,000 references to `StrongBox<ChunkyObject>`.

<img width="760" alt="image" src="https://user-images.githubusercontent.com/95136/188351308-b174f843-0435-46db-8f31-b4d78c740947.png">

### Avoid setting AsyncLocal\<T\> values outside of async methods

Async methods have a special behavior for async locals that make sure values do not propagate outside of the async method.

❌ **BAD** Avoid setting async local values outside of async methods:

```C#
var local = new AsyncLocal<int>();
MethodA();
Console.WriteLine(local.Value);

void MethodA()
{
    local.Value = 1;
    MethodB();
    Console.WriteLine(local.Value);
}

void MethodB()
{
    local.Value = 2;
    Console.WriteLine(local.Value);
}
```

The above prints 2, 2, 2. The execution context mutations are being propagated outside of the method. This can lead to extremely confusing behavior and hard to track down bugs.

:white_check_mark: **GOOD** Set async locals in async methods:

```C#
var local = new AsyncLocal<int>();
await MethodA();
Console.WriteLine(local.Value);

async Task MethodA()
{
    local.Value = 1;
    await MethodB();
    Console.WriteLine(local.Value);
}

async Task MethodB()
{
    local.Value = 2;
    Console.WriteLine(local.Value);
}
```

The above will print 2, 1, 0. This is because async method restore the original execution context on exit.

## ConfigureAwait

TBD

# Scenarios

The above tries to distill general guidance, but doesn't do justice to the kinds of real-world situations that cause code like this to be written in the first place (bad code). This section tries to take concrete examples from real applications and turn them into something simple to help you relate these problems to existing codebases.

## `Timer` callbacks

❌ **BAD** The `Timer` callback is `void`-returning and we have asynchronous work to execute. This example uses `async void` to accomplish it and as a result can crash the process if an exception occurs.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public async void Heartbeat(object state)
    {
        await _client.GetAsync("http://mybackend/api/ping");
    }
}
```

❌ **BAD** This attempts to block in the `Timer` callback. This may result in thread-pool starvation and is an example of [sync over async](#warning-sync-over-async)

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object state)
    {
        _client.GetAsync("http://mybackend/api/ping").GetAwaiter().GetResult();
    }
}
```

:white_check_mark: **GOOD** This example uses an `async Task`-based method and discards the `Task` in the `Timer` callback. If this method fails, it will not crash the process. Instead, it will fire the [`TaskScheduler.UnobservedTaskException`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.taskscheduler.unobservedtaskexception) event.

```C#
public class Pinger
{
    private readonly Timer _timer;
    private readonly HttpClient _client;
    
    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new Timer(Heartbeat, null, 1000, 1000);
    }

    public void Heartbeat(object state)
    {
        // Discard the result
        _ = DoAsyncPing();
    }

    private async Task DoAsyncPing()
    {
        await _client.GetAsync("http://mybackend/api/ping");
    }
}
```

:white_check_mark: **GOOD** This example uses the new [`PeriodicTimer`](https://learn.microsoft.com/en-us/dotnet/api/system.threading.periodictimer) introduced in .NET 6:

```C#
public class Pinger : IDisposable
{
    private readonly PeriodicTimer _timer;
    private readonly HttpClient _client;

    public Pinger(HttpClient client)
    {
        _client = client;
        _timer = new PeriodicTimer(TimeSpan.FromSeconds(1));
        _ = Task.Run(DoAsyncPings);
    }

    public void Dispose()
    {
        _timer.Dispose();
    }

    private async Task DoAsyncPings()
    {
        while (await _timer.WaitForNextTickAsync())
        {
            // TODO: Handle exceptions
            await _client.GetAsync("http://mybackend/api/ping");
        }
    }
}
```

## Implicit `async void` delegates

Imagine a `BackgroundQueue` with a `FireAndForget` that takes a callback. This method will execute the callback at some time in the future.

❌ **BAD** This will force callers to either block in the callback or use an `async void` delegate.

```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
}
```

❌ **BAD** This calling code is creating an `async void` method implicitly. The compiler fully supports this today.

```C#
public class Program
{
    public void Main(string[] args)
    {
        var httpClient = new HttpClient();
        BackgroundQueue.FireAndForget(async () =>
        {
            await httpClient.GetAsync("http://pinger/api/1");
        });
        
        Console.ReadLine();
    }
}
```

:white_check_mark: **GOOD** This BackgroundQueue implementation offers both sync and `async` callback overloads.

```C#
public class BackgroundQueue
{
    public static void FireAndForget(Action action) { }
    public static void FireAndForget(Func<Task> action) { }
}
```

## `ConcurrentDictionary.GetOrAdd`

It's pretty common to cache the result of an asynchronous operation and `ConcurrentDictionary` is a good data structure for doing that. `GetOrAdd` is a convenience API for trying to get an item if it's already there or adding it if it isn't. The callback is synchronous so it's tempting to write code that uses `Task.Result` to produce the value of an asynchronous process but that can lead to thread-pool starvation.

❌ **BAD** This may result in thread-pool starvation since we're blocking the request thread if the person data is not cached.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, Person> _cache = new ConcurrentDictionary<int, Person>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public IActionResult Get(int id)
   {
       var person = _cache.GetOrAdd(id, (key) => _db.People.FindAsync(key).Result);
       return Ok(person);
   }
}
```

:white_check_mark: **GOOD** This implementation won't result in thread-pool starvation since we're storing a task instead of the result itself.

:warning: `ConcurrentDictionary.GetOrAdd`, when accessed concurrently, may run the value-constructing delegate multiple times. This can result in needlessly kicking off the same potentially expensive computation multiple times.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, Task<Person>> _cache = new ConcurrentDictionary<int, Task<Person>>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public async Task<IActionResult> Get(int id)
   {
       var person = await _cache.GetOrAdd(id, (key) => _db.People.FindAsync(key));
       return Ok(person);
   }
}
```

:white_check_mark: **GOOD** This implementation prevents the delegate from being executed multiple times, by using the `async` lazy pattern: even if construction of the AsyncLazy instance happens multiple times ("cheap" operation), the delegate will be called only once.

```C#
public class PersonController : Controller
{
   private AppDbContext _db;
   
   // This cache needs expiration
   private static ConcurrentDictionary<int, AsyncLazy<Person>> _cache = new ConcurrentDictionary<int, AsyncLazy<Person>>();
   
   public PersonController(AppDbContext db)
   {
      _db = db;
   }
   
   public async Task<IActionResult> Get(int id)
   {
       var person = await _cache.GetOrAdd(id, (key) => new AsyncLazy<Person>(() => _db.People.FindAsync(key))).Value;
       return Ok(person);
   }
   
   private class AsyncLazy<T> : Lazy<Task<T>>
   {
      public AsyncLazy(Func<Task<T>> valueFactory) : base(valueFactory)
      {
      }
   }
}
```

## Constructors

Constructors are synchronous. If you need to initialize some logic that may be asynchronous, there are a couple of patterns for dealing with this.

Here's an example of using a client API that needs to connect asynchronously before use.

```C#
public interface IRemoteConnectionFactory
{
   Task<IRemoteConnection> ConnectAsync();
}

public interface IRemoteConnection
{
    Task PublishAsync(string channel, string message);
    Task DisposeAsync();
}
```


❌ **BAD** This example uses `Task.Result` to get the connection in the constructor. This could lead to thread-pool starvation and deadlocks.

```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;
    
    public Service(IRemoteConnectionFactory connectionFactory)
    {
        _connection = connectionFactory.ConnectAsync().Result;
    }
}
```

:white_check_mark: **GOOD** This implementation uses a static factory pattern in order to allow asynchronous construction:

```C#
public class Service : IService
{
    private readonly IRemoteConnection _connection;

    private Service(IRemoteConnection connection)
    {
        _connection = connection;
    }

    public static async Task<Service> CreateAsync(IRemoteConnectionFactory connectionFactory)
    {
        return new Service(await connectionFactory.ConnectAsync());
    }
}
```

## WindowsIdentity.RunImpersonated

This API runs the specified action as the impersonated Windows identity. An [asynchronous version of the callback](https://docs.microsoft.com/en-us/dotnet/api/system.security.principal.windowsidentity.runimpersonatedasync) was introduced in .NET 5.0.

❌ **BAD** This example tries to execute the query asynchronously, and then wait for it outside of the call to `RunImpersonated`. This will throw because the query might be executing outside of the impersonation context.

```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    Task<IEnumerable<Product>> products = null;
    WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        context =>
        {
            products = _db.QueryAsync("SELECT Name from Products");
        });
    return await products;
}
```

❌ **BAD** This example uses `Task.Result` to get the connection in the constructor. This could lead to thread-pool starvation and deadlocks.

```C#
public IEnumerable<Product> GetDataImpersonated(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle,
        context => _db.QueryAsync("SELECT Name from Products").Result);
}
```

:white_check_mark: **GOOD** This example awaits the result of `RunImpersonated` (the delegate is `Func<Task<IEnumerable<Product>>>` in this case). It is the recommended practice in frameworks earlier than .NET 5.0.

```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return await WindowsIdentity.RunImpersonated(
        safeAccessTokenHandle, 
        context => _db.QueryAsync("SELECT Name from Products"));
}
```

:white_check_mark: **GOOD** This example uses the asynchronous `RunImpersonatedAsync` function and awaits its result. It is available in .NET 5.0 or newer.

```C#
public async Task<IEnumerable<Product>> GetDataImpersonatedAsync(SafeAccessTokenHandle safeAccessTokenHandle)
{
    return await WindowsIdentity.RunImpersonatedAsync(
        safeAccessTokenHandle, 
        context => _db.QueryAsync("SELECT Name from Products"));
}
```
