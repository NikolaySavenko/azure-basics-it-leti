# Durable Functions

[Durable Functions.pptx](Durable%20Functions%204144d6825bdb4ff183b1fc4eb5bc0354/Durable_Functions.pptx)

## В предыдущих сериях

Azure Serverless Functions позволяют нам легко и быстро писать Serverless приложения на основе архитектуры событий. Event-Driven Design

Что за Serverless - это когда мы не имеем постоянно работающего экземпляра приложения, а имеем временный экземпляр, который подготавливается в момент возникновения нужного нам события, а затем, после выполнения кода, уничтожается. Это очень дешево как провайдеру, так и нам.

**Параллельность на уровне множества машин.**

Наше приложение легко и быстро автоматически масштабируется на множество машин.

**Триггеры и привязки.**

Чтобы функция запустилась по нужному нам событию мы указываем нужный нам триггер.

Чтобы функция работала с внешними данными(например БД) мы указываем привязку.

## Устойчивые функции

Устойчивые функции — это расширение [Функций Azure](https://docs.microsoft.com/ru-ru/azure/azure-functions/functions-overview)
.Устойчивые функции предоставляют возможность оркестрации выполнения функции с отслеживанием состояния. Устойчивая функция приложения — это решение, сформированное из различных функций Azure. Функции могут играть различные роли в устойчивой оркестрации функций.

### ****Client function →**** Orchestrator Function → Activity Functions

### Activity Functions (функции действий)

Для задания используем специальный **ActivityTrigger**

Функция, которая выполняет полезную работу и не требуют асинхронности.

Обычно в такие функции передают уже обработанные данные через параметры. 

Так же в таких функциях производят синхронную(это важно) обработку информации, и возвращают результат обратно тому, кто ее вызвал.

Функции сдедует делать максимально быстрыми.

Что делать: 

- запись готовых данных в БД
- отправлять готовые сообщения в очередь

### Orchestrator Functions (****Функции оркестратора****)

Функция, которая рулит и запускает Activity функции.

В такой функции уже можно использовать стандартные операторы для асинхронности.

Функции оркестратора могут быть длительными.

Обычно код таких функций содержит:

- Запуск Activity функций
- Проверка и ожидания выполненности Activity
- Вызов HTTP API если нужно (например веб скраппинг)
- Возврат результатов обратно клиентской функции
- вызов других функций оркестратора если нужно

Но есть ряд ограничений 

[https://docs.microsoft.com/ru-ru/azure/azure-functions/durable/durable-functions-code-constraints](https://docs.microsoft.com/ru-ru/azure/azure-functions/durable/durable-functions-code-constraints)

### ****Client functions (****Клиентская функции)

Функция, которая запускает функцию оркестраторы.

Обычно такие функции используют HTTP триггер или ему подобные.

Т.е. именно эту функцию запускает пользователь приложения.

### Паттерны

Список того, как может выглядеть архитектура функций. Стоит просто примерно помнить какие есть паттерны и вовремя вспомнить какой похож на то, что вы хотите сделать в приложении

****Цепочка функций****

![Untitled](Durable%20Functions%204144d6825bdb4ff183b1fc4eb5bc0354/Untitled.png)

В шаблоне цепочки функций последовательность функций выполняется в определенном порядке. В этом шаблоне выходные данные одной функции применяются к входным данным другой.

```csharp
[FunctionName("Chaining")]
public static async Task<object> Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    try
    {
        var x = await context.CallActivityAsync<object>("F1", null);
        var y = await context.CallActivityAsync<object>("F2", x);
        var z = await context.CallActivityAsync<object>("F3", y);
        return  await context.CallActivityAsync<object>("F4", z);
    }
    catch (Exception)
    {
        // Обработка ошибок.
    }
}
```

****Развертывание и объединение****

![Untitled](Durable%20Functions%204144d6825bdb4ff183b1fc4eb5bc0354/Untitled%201.png)

В шаблоне развертывания и объединения вы выполняете параллельно несколько функций, а затем ожидаете их завершения. Часто некоторые агрегирования завершаются по результатам, возвращаемым функциями.

```csharp
[FunctionName("FanOutFanIn")]
public static async Task Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var parallelTasks = new List<Task<int>>();

    // Запускаем несколько Activity функций
    object[] workBatch = await context.CallActivityAsync<object[]>("F1", null);
    for (int i = 0; i < workBatch.Length; i++)
    {
        Task<int> task = context.CallActivityAsync<int>("F2", workBatch[i]);
        parallelTasks.Add(task);
    }

    await Task.WhenAll(parallelTasks);

    // Aggregate all N outputs and send the result to F3.
    int sum = parallelTasks.Sum(t => t.Result);
    await context.CallActivityAsync("F3", sum);
}
```

****Монитор****

![Untitled](Durable%20Functions%204144d6825bdb4ff183b1fc4eb5bc0354/Untitled%202.png)

Шаблон мониторинга представляет собой гибкий, повторяющийся процесс в рабочем процессе. Например, повторение опроса, пока не будут выполнены определенные условия. Вы можете использовать регулярный [триггер таймера](https://docs.microsoft.com/ru-ru/azure/azure-functions/functions-bindings-timer) для активации основного сценария, например задание периодической очистки.

```csharp
[FunctionName("MonitorJobStatus")]
public static async Task Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    int jobId = context.GetInput<int>();
    int pollingInterval = GetPollingInterval();
    DateTime expiryTime = GetExpiryTime();

    while (context.CurrentUtcDateTime < expiryTime)
    {
        var jobStatus = await context.CallActivityAsync<string>("GetJobStatus", jobId);
        if (jobStatus == "Completed")
        {
            // Выполняем Activity после завершения
            await context.CallActivityAsync("SendAlert", machineId);
            break;
        }

        // Иначе ставим время следующей проверки
        var nextCheck = context.CurrentUtcDateTime.AddSeconds(pollingInterval);
        await context.CreateTimer(nextCheck, CancellationToken.None);
    }
}
```

****Участие пользователя****

![Untitled](Durable%20Functions%204144d6825bdb4ff183b1fc4eb5bc0354/Untitled%203.png)

Процесс утверждения является примером бизнес-процесса, который предполагает взаимодействие с пользователями. Утверждение от менеджера может потребоваться для отчета о расходах, превышающих определенную денежную сумму. Если менеджер не подтверждает отчет о расходах в течение 72 часов (возможно, он отправился в отпуск), начинается процесс эскалации, чтобы получить одобрение от кого-то другого (возможно, вышестоящего менеджера).

```csharp
[FunctionName("ApprovalWorkflow")]
public static async Task Run(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    await context.CallActivityAsync("RequestApproval", null);
    using (var timeoutCts = new CancellationTokenSource())
    {
        DateTime dueTime = context.CurrentUtcDateTime.AddHours(72);
        Task durableTimeout = context.CreateTimer(dueTime, timeoutCts.Token);

        Task<bool> approvalEvent = context.WaitForExternalEvent<bool>("ApprovalEvent");
        if (approvalEvent == await Task.WhenAny(approvalEvent, durableTimeout))
        {
            timeoutCts.Cancel();
            await context.CallActivityAsync("ProcessApproval", approvalEvent.Result);
        }
        else
        {
            await context.CallActivityAsync("Escalate", null);
        }
    }
}
```

Событие можно вызвать например из HTTP функции

```csharp
[FunctionName("RaiseEventToOrchestration")]
public static async Task Run(
    [HttpTrigger] string instanceId,
    [DurableClient] IDurableOrchestrationClient client)
{
    bool isApproved = true;
    await client.RaiseEventAsync(instanceId, "ApprovalEvent", isApproved);
}
```