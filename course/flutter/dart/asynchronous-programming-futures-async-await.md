### 异步编程：使用 Future 和 async-await  

*   [为何异步代码至关重要](#why-asynchronous-code-matters)
    *   [示例：错误使用异步函数](#example-incorrectly-using-an-asynchronous-function)
*   [什么是 Future？](#what-is-a-future)
    *   [未完成状态](#uncompleted)
    *   [完成状态](#completed)
    *   [示例：初识 Future](#example-introducing-futures)
    *   [示例：以错误完成](#example-completing-with-an-error)
*   [使用 Future：async 和 await](#working-with-futures-async-and-await)
    *   [async 和 await 的执行流程](#execution-flow-with-async-and-await)
    *   [示例：异步函数内的执行](#example-execution-within-async-functions)
    *   [练习：使用 async 和 await 进行练习](#exercise-practice-using-async-and-await)
*   [错误处理](#handling-errors)
    *   [示例：async 和 await 与 try-catch](#example-async-and-await-with-try-catch)
    *   [练习：练习处理错误](#exercise-practice-handling-errors)
*   [综合练习](#exercise-putting-it-all-together)
*   [哪些代码检查规则适用于 Future？](#which-lints-work-for-futures)
*   [下一步](#whats-next)


本教程将教你如何使用 Future 以及 `async` 和 `await` 关键字编写异步代码。通过嵌入的 DartPad 编辑器，你可以运行示例代码并完成练习来测试你的知识。  

为了充分利用本教程，你应该具备以下条件：  

*   了解 [Dart 基础语法](/language)。  
*   有其他语言的异步编程经验。  
*   启用了 [`discarded_futures`](/tools/linter-rules/discarded_futures) 和 [`unawaited_futures`](/tools/linter-rules/unawaited_futures) 代码检查规则。  

本教程涵盖以下内容：  

*   如何以及何时使用 `async` 和 `await` 关键字。  
*   使用 `async` 和 `await` 对执行顺序的影响。  
*   如何在 `async` 函数中使用 `try-catch` 表达式处理异步调用中的错误。  

预计完成本教程的时间：40-60 分钟。  

**提示**  
> 如果你只看到了空白的框框（而没有任何内容），请查阅 [DartPad 常见问题页面](/tools/dartpad/troubleshoot)。  

本教程中的练习包含部分完成的代码片段。你可以使用 DartPad 完成代码并点击 **Run** 按钮来测试你的知识。**不要编辑 `main` 函数或以下的测试代码**。  

如果你需要帮助，可以在每个练习后展开 **Hint** 或 **Solution** 下拉菜单。  

---

### 为何异步代码至关重要  
[#](#why-asynchronous-code-matters)  

异步操作允许你的程序在等待其他操作完成时继续工作。以下是一些常见的异步操作：  

*   通过网络获取数据。  
*   写入数据库。  
*   从文件中读取数据。  

这些异步计算通常将其结果作为 `Future` 提供，或者如果结果有多个部分，则作为 `Stream` 提供。这些计算将异步性引入程序中。为了适应这种初始的异步性，其他普通的 Dart 函数也需要变为异步的。  

为了与这些异步结果交互，你可以使用 `async` 和 `await` 关键字。大多数异步函数只是依赖于（可能在深层）本质上异步计算的异步 Dart 函数。  

#### 示例：错误使用异步函数  
[#](#example-incorrectly-using-an-asynchronous-function)  

以下示例展示了错误使用异步函数（`fetchUserOrder()`）的方式。稍后你将使用 `async` 和 `await` 修复此示例。在运行此示例之前，尝试找出问题——你认为输出会是什么？  

以下是该示例未能打印 `fetchUserOrder()` 最终生成的值的原因：  

*   `fetchUserOrder()` 是一个异步函数，经过延迟后提供一个描述用户订单的字符串：“Large Latte”。  
*   为了获取用户的订单，`createOrderMessage()` 应该调用 `fetchUserOrder()` 并等待其完成。由于 `createOrderMessage()` 没有等待 `fetchUserOrder()` 完成，`createOrderMessage()` 未能获取 `fetchUserOrder()` 最终提供的字符串值。  
*   相反，`createOrderMessage()` 获取了一个待完成工作的表示：一个未完成的 Future。你将在下一节中了解更多关于 Future 的内容。  
*   由于 `createOrderMessage()` 未能获取描述用户订单的值，该示例未能将“Large Latte”打印到控制台，而是打印了“Your order is: Instance of '_Future<String>'”。  

在接下来的部分中，你将学习 Future 以及如何使用 Future（使用 `async` 和 `await`），以便能够编写必要的代码，使 `fetchUserOrder()` 将所需的值（“Large Latte”）打印到控制台。  

**关键术语**  

*   **同步操作**：同步操作会阻止其他操作执行，直到它完成。  
*   **同步函数**：同步函数仅执行同步操作。  
*   **异步操作**：一旦启动，异步操作允许其他操作在其完成之前执行。  
*   **异步函数**：异步函数至少执行一个异步操作，也可以执行同步操作。  

---

### 什么是 Future？  
[#](#what-is-a-future)  

Future（小写“f”）是 [Future](https://api.dart.cn/dart-async/Future-class.html)（大写“F”）类的一个实例。Future 表示异步操作的结果，可以有两种状态：未完成或已完成。  

**提示**  
> _未完成_ 是 Dart 中的一个术语，指的是 Future 在生成值之前的状态。  

#### 未完成  
[#](#uncompleted)  

当你调用一个异步函数时，它会返回一个未完成的 Future。该 Future 正在等待函数的异步操作完成或抛出错误。  

#### 已完成  
[#](#completed)  

如果异步操作成功，Future 将以一个值完成。否则，它将以一个错误完成。  

##### 以值完成  
[#](#completing-with-a-value)  

类型为 `Future<T>` 的 Future 将以类型为 `T` 的值完成。例如，类型为 `Future<String>` 的 Future 会生成一个字符串值。如果 Future 没有生成可用的值，则 Future 的类型为 `Future<void>`。  

##### 以错误完成  
[#](#completing-with-an-error)  

如果函数执行的异步操作因任何原因失败，Future 将以错误完成。  

#### 示例：初识 Future  
[#](#example-introducing-futures)  

在以下示例中，`fetchUserOrder()` 返回一个在打印到控制台后完成的 Future。由于它没有返回可用的值，`fetchUserOrder()` 的类型为 `Future<void>`。在运行此示例之前，尝试预测哪个会先打印：“Large Latte”还是“Fetching user order...”。  

在前面的示例中，尽管 `fetchUserOrder()` 在第 8 行的 `print()` 调用之前执行，但控制台显示第 8 行的输出（“Fetching user order...”）先于 `fetchUserOrder()` 的输出（“Large Latte”）。这是因为 `fetchUserOrder()` 在打印“Large Latte”之前延迟了。  

#### 示例：以错误完成  
[#](#example-completing-with-an-error)  

运行以下示例以查看 Future 如何以错误完成。稍后你将学习如何处理错误。  

在此示例中，`fetchUserOrder()` 以指示用户 ID 无效的错误完成。  

你已经了解了 Future 及其完成方式，但如何使用异步函数的结果呢？在下一节中，你将学习如何使用 `async` 和 `await` 关键字获取结果。  

**快速回顾**  

*   [Future<T>](https://api.dart.cn/dart-async/Future-class.html) 实例生成类型为 `T` 的值。  
*   如果 Future 没有生成可用的值，则 Future 的类型为 `Future<void>`。  
*   Future 可以处于两种状态之一：未完成或已完成。  
*   当你调用返回 Future 的函数时，该函数会排队等待工作完成并返回一个未完成的 Future。  
*   当 Future 的操作完成时，Future 将以一个值或错误完成。  

**关键术语：**  

*   **Future**：Dart 的 [Future](https://api.dart.cn/dart-async/Future-class.html) 类。  
*   **future**：Dart `Future` 类的实例。  

---

### 使用 Future：async 和 await  
[#](#working-with-futures-async-and-await)  

`async` 和 `await` 关键字提供了一种声明式的方式来定义异步函数并使用其结果。使用 `async` 和 `await` 时，请记住以下两条基本准则：  

*   **要定义异步函数，请在函数体前添加 `async`。**  
*   **`await` 关键字仅在 `async` 函数中有效。**  

以下是一个将 `main()` 从同步函数转换为异步函数的示例。  

首先，在函数体前添加 `async` 关键字：  

```dart  
void main() async { ··· }  
```  

如果函数有声明的返回类型，则将其更新为 `Future<T>`，其中 `T` 是函数返回值的类型。如果函数没有显式返回值，则返回类型为 `Future<void>`：  

```dart  
Future<void> main() async { ··· }  
```  

现在你有了一个 `async` 函数，可以使用 `await` 关键字等待 Future 完成：  

```dart  
print(await createOrderMessage());  
```  

如以下两个示例所示，`async` 和 `await` 关键字使异步代码看起来非常像同步代码。唯一的区别在异步示例中突出显示，如果你的窗口足够宽，它位于同步示例的右侧。  

#### 示例：同步函数  
[#](#example-synchronous-functions)  

```dart  
String createOrderMessage() {  
  var order = fetchUserOrder();  
  return 'Your order is: $order';  
}  

Future<String> fetchUserOrder() =>  
  // 假设此函数更复杂且更慢。  
  Future.delayed(const Duration(seconds: 2), () => 'Large Latte');  

void main() {  
  print('Fetching user order...');  
  print(createOrderMessage());  
}  
```  

输出：  
```  
Fetching user order...  
Your order is: Instance of 'Future<String>'  
```  

如以下两个示例所示，它的行为类似于同步代码。  

#### 示例：异步函数  
[#](#example-asynchronous-functions)  

```dart  
Future<String> createOrderMessage() async {  
  var order = await fetchUserOrder();  
  return 'Your order is: $order';  
}  

Future<String> fetchUserOrder() =>  
  // 假设此函数更复杂且更慢。  
  Future.delayed(const Duration(seconds: 2), () => 'Large Latte');  

Future<void> main() async {  
  print('Fetching user order...');  
  print(await createOrderMessage());  
}  
```  

输出：  
```  
Fetching user order...  
Your order is: Large Latte  
```  

异步示例在三个方面有所不同：  

*   `createOrderMessage()` 的返回类型从 `String` 更改为 `Future<String>`。  
*   **`async`** 关键字出现在 `createOrderMessage()` 和 `main()` 的函数体前。  
*   **`await`** 关键字出现在调用异步函数 `fetchUserOrder()` 和 `createOrderMessage()` 之前。  

**关键术语**  

*   **async**：你可以在函数体前使用 `async` 关键字将其标记为异步。  
*   **异步函数**：异步函数是用 `async` 关键字标记的函数。  
*   **await**：你可以使用 `await` 关键字获取异步表达式的完成结果。`await` 关键字仅在 `async` 函数中有效。  

#### 使用 async 和 await 的执行流程  
[#](#execution-flow-with-async-and-await)  

`async` 函数会同步运行，直到第一个 `await` 关键字。这意味着在 `async` 函数体内，第一个 `await` 关键字之前的所有同步代码会立即执行。  

#### 示例：异步函数内的执行  
[#](#example-execution-within-async-functions)  

运行以下示例以查看 `async` 函数体内的执行过程。你认为输出会是什么？  

在运行前面示例中的代码后，尝试反转第 2 行和第 3 行：  

```dart  
var order = await fetchUserOrder();  
print('Awaiting user order...');  
```  

注意输出的时间变化，现在 `print('Awaiting user order')` 出现在 `printOrderMessage()` 中第一个 `await` 关键字之后。  

#### 练习：使用 async 和 await 进行练习  
[#](#exercise-practice-using-async-and-await)  

以下练习是一个失败的单元测试，包含部分完成的代码片段。你的任务是通过编写代码完成练习以使测试通过。你不需要实现 `main()`。  

为了模拟异步操作，调用以下提供的函数：  

| 函数 | 类型签名 | 描述 |  
|------|----------|------|  
| `fetchRole()` | `Future<String> fetchRole()` | 获取用户角色的简短描述。 |  
| `fetchLoginAmount()` | `Future<int> fetchLoginAmount()` | 获取用户登录的次数。 |  

##### 第 1 部分：`reportUserRole()`  
[#](#part-1-reportuserrole)  

向 `reportUserRole()` 函数添加代码，使其执行以下操作：  

*   返回一个以以下字符串完成的 Future：`"User role: <user role>"`  
    *   注意：你必须使用 `fetchRole()` 返回的实际值；复制粘贴示例返回值不会使测试通过。  
    *   示例返回值：`"User role: tester"`  
*   通过调用提供的函数 `fetchRole()` 获取用户角色。  

##### 第 2 部分：`reportLogins()`  
[#](#part-2-reportlogins)  

实现一个异步函数 `reportLogins()`，使其执行以下操作：  

*   返回字符串 `"Total number of logins: <# of logins>"`。  
    *   注意：你必须使用 `fetchLoginAmount()` 返回的实际值；复制粘贴示例返回值不会使测试通过。  
    *   `reportLogins()` 的示例返回值：`"Total number of logins: 57"`  
*   通过调用提供的函数 `fetchLoginAmount()` 获取登录次数。  

**提示**  
你是否记得在 `reportUserRole` 函数中添加 `async` 关键字？  
你是否记得在调用 `fetchRole()` 之前使用 `await` 关键字？  
记住：`reportUserRole` 需要返回一个 `Future`。  

**解决方案**  

```dart  
Future<String> reportUserRole() async {  
  final username = await fetchRole();  
  return 'User role: $username';  
}  

Future<String> reportLogins() async {  
  final logins = await fetchLoginAmount();  
  return 'Total number of logins: $logins';  
}  
```  

---

### 错误处理  
[#](#handling-errors)  

要在 `async` 函数中处理错误，请使用 try-catch：  

```dart  
try {  
  print('Awaiting user order...');  
  var order = await fetchUserOrder();  
} catch (err) {  
  print('Caught error: $err');  
}  
```  

在 `async` 函数中，你可以像在同步代码中一样编写 [try-catch 子句](/language/error-handling#catch)。  

#### 示例：async 和 await 与 try-catch  
[#](#example-async-and-await-with-try-catch)  

运行以下示例以查看如何处理异步函数中的错误。你认为输出会是什么？  

#### 练习：练习处理错误  
[#](#exercise-practice-handling-errors)  

以下练习提供了使用异步代码处理错误的练习，使用前面描述的方法。为了模拟异步操作，你的代码将调用以下提供的函数：  

| 函数 | 类型签名 | 描述 |  
|------|----------|------|  
| `fetchNewUsername()` | `Future<String> fetchNewUsername()` | 返回可用于替换旧用户名的新用户名。 |  

使用 `async` 和 `await` 实现一个异步的 `changeUsername()` 函数，使其执行以下操作：  

*   调用提供的异步函数 `fetchNewUsername()` 并返回其结果。  
    *   `changeUsername()` 的示例返回值：`"jane_smith_92"`  
*   捕获发生的任何错误并返回错误的字符串值。  
    *   你可以使用 [toString()](https://api.dart.cn/dart-core/ArgumentError/toString.html) 方法将 [Exceptions](https://api.dart.cn/dart-core/Exception-class.html) 和 [Errors](https://api.dart.cn/dart-core/Error-class.html) 字符串化。  

**提示**  
实现 `changeUsername` 以返回 `fetchNewUsername` 的字符串，或者如果失败，则返回任何发生的错误的字符串值。  
记住：你可以使用 [try-catch 语句](/language/error-handling#catch) 来捕获和处理错误。  

**解决方案**  

```dart  
Future<String> changeUsername() async {  
  try {  
    return await fetchNewUsername();  
  } catch (err) {  
    return err.toString();  
  }  
}  
```  

---

### 综合练习  
[#](#exercise-putting-it-all-together)  

是时候练习你所学的内容了。为了模拟异步操作，本练习提供了异步函数 `fetchUsername()` 和 `logoutUser()`：  

| 函数 | 类型签名 | 描述 |  
|------|----------|------|  
| `fetchUsername()` | `Future<String> fetchUsername()` | 返回与当前用户关联的名称。 |  
| `logoutUser()` | `Future<String> logoutUser()` | 执行当前用户的注销并返回注销的用户名。 |  

编写以下内容：  

#### 第 1 部分：`addHello()`  
[#](#part-1-addhello)  

*   编写一个函数 `addHello()`，它接受一个 `String` 参数。  
*   `addHello()` 返回其 `String` 参数前加上 `'Hello '`。  
    示例：`addHello('Jon')` 返回 `'Hello Jon'`。  

#### 第 2 部分：`greetUser()`  
[#](#part-2-greetuser)  

*   编写一个函数 `greetUser()`，它不接受任何参数。  
*   为了获取用户名，`greetUser()` 调用提供的异步函数 `fetchUsername()`。  
*   `greetUser()` 通过调用 `addHello()` 并传递用户名来创建用户的问候语，然后返回结果。  
    示例：如果 `fetchUsername()` 返回 `'Jenny'`，则 `greetUser()` 返回 `'Hello Jenny'`。  

#### 第 3 部分：`sayGoodbye()`  
[#](#part-3-saygoodbye)  

*   编写一个函数 `sayGoodbye()`，使其执行以下操作：  
    *   不接受任何参数。  
    *   捕获任何错误。  
    *   调用提供的异步函数 `logoutUser()`。  
*   如果 `logoutUser()` 失败，`sayGoodbye()` 返回任何你喜欢的字符串。  
*   如果 `logoutUser()` 成功，`sayGoodbye()` 返回字符串 `'<result> Thanks, see you next time'`，其中 `<result>` 是调用 `logoutUser()` 返回的字符串值。  

**提示**  
`greetUser` 和 `sayGoodbye` 函数应该是异步的，而 `addHello` 应该是一个普通的同步函数。  
记住：你可以使用 [try-catch 语句](/language/error-handling#catch) 来捕获和处理错误。  

**解决方案**  

```dart  
String addHello(String user) => 'Hello $user';  

Future<String> greetUser() async {  
  final username = await fetchUsername();  
  return addHello(username);  
}  

Future<String> sayGoodbye() async {  
  try {  
    final result = await logoutUser();  
    return '$result Thanks, see you next time';  
  } catch (e) {  
    return 'Failed to logout user: $e';  
  }  
}  
```  

---

### 哪些代码检查规则适用于 Future？  
[#](#which-lints-work-for-futures)  

为了捕获在使用 async 和 Future 时常见的错误，请 [启用](/tools/analysis#individual-rules) 以下代码检查规则：  

*   [`discarded_futures`](/tools/linter-rules/discarded_futures)  
*   [`unawaited_futures`](/tools/linter-rules/unawaited_futures)  

---

### 下一步  
[#](#whats-next)  

恭喜，你已经完成了本教程！如果你想了解更多，以下是一些建议：  

*   使用 [DartPad](https://dartpad.cn) 进行练习。  
*   尝试另一个 [教程](/tutorials)。  
*   了解更多关于 Dart 中的 Future 和异步代码：  
    *   [流处理教程](/libraries/async/using-streams)：学习如何处理一系列异步事件。  
    *   [Dart 中的并发](/language/concurrency)：了解并学习如何在 Dart 中实现并发。  
    *   [异步支持](/language/async)：深入了解 Dart 的语言和库对异步编程的支持。  
    *   [Google 的 Dart 视频](https://www.youtube.com/playlist?list=PLjxrf2q8roU0Net_g1NT5_vOO3s_FR02J)：观看一个或多个关于异步编程的视频。  
*   获取 [Dart SDK](/get-dart)！  

除非另有说明，文档内容适用于 Dart 3.7.1 版本，本页面最后更新时间：2025-02-14。  
[查看文档源码](https://github.com/cfug/dart.cn/tree/main/src/content/libraries/async/async-await.md) 或 [报告页面问题](https://github.com/cfug/dart.cn/issues/new?template=1_page_issue.yml&page-url=https://dart.cn/libraries/async/async-await/&page-source=https://github.com/cfug/dart.cn/tree/main/src/content/libraries/async/async-await.md "为本页面内容提出建议")。