# FileFuzzer.xml 逐行精读（真·文件 fuzz PIT 样例）

> 基于 `peach-3.0.202-source/Peach/samples/FileFuzzer.xml`。
> 这是最经典的文件 fuzz 套路：**写文件 → 拿程序打开它 → 调试器盯着看崩不崩**。
> 配套阅读：`Peach工作流手把手教程.md`。

> ⚠️ 注意：`FileFuzzer.xml` 顶部注释里写的 `Debugger.xml`、`test 47` 是从别的样例复制过来忘了改的**过时注释**，别被带偏，看代码本身。

---

## 这份 PIT 在干什么（全景）

```
每一轮：
  ① 把（被变异的）数据写成文件 fuzzfile.bin      ← Action output + File publisher
  ② 关闭文件                                     ← Action close
  ③ 通知 agent「文件写好了，启动 notepad 打开它」 ← Action call ScoobySnacks
  ④ WindowsDebugger 挂在 notepad 上，崩了就抓栈   ← Monitor
```

---

## 逐块逐行读

### 数据模板（32-34）

```xml
<DataModel name="TestTemplate">
    <String value="Hello World!" />
</DataModel>
```

- 这是**要被变异的数据结构**。这里偷懒只放了一个字符串——**真实文件 fuzz 时，这里应该建模真正的文件格式**（PNG 的 chunk、PDF 的 object 等），格式建得越细，变异越有针对性。
- `name="TestTemplate"` 是它的身份证，后面 `ref` 靠这个名字找它。

### 状态机：一轮要做的动作序列（38-54）

```xml
<StateModel name="State" initialState="Initial">
    <State name="Initial">
        <Action type="output">              <!-- 动作①：把数据发出去 -->
            <DataModel ref="TestTemplate" />
        </Action>
        <Action type="close" />             <!-- 动作②：关闭文件 -->
        <Action type="call" method="ScoobySnacks" publisher="Peach.Agent"/>  <!-- 动作③ -->
    </State>
</StateModel>
```

三个动作按顺序执行，这就是执行层：

- `type="output"`：走 `publisher.output()`，把 `dataModel.Value.Stream` 写进文件（因为下面 Test 里 publisher 是 `File`）。
- `type="close"`：`publisher.close()`，确保文件**完整落盘**——不关文件，notepad 可能读到半截。
- `type="call"` + `publisher="Peach.Agent"`：**这一个特殊**。`publisher="Peach.Agent"` 是保留写法，表示「这不是对目标做 IO，而是给 agent 发一条消息」（`Call` 若 publisher 是 `Peach.Agent` 就走 `AgentManager.Message`）。这里发的消息名是 `ScoobySnacks`。

### Agent + Monitor：怎么发现崩溃（57-79）

```xml
<Agent name="LocalAgent">
    <Monitor class="WindowsDebugger">
        <Param name="CommandLine" value="c:\windows\system32\notepad.exe fuzzfile.bin" />
        <Param name="StartOnCall" value="ScoobySnacks" />
    </Monitor>
    <Monitor class="PageHeap">
        <Param name="Executable" value="notepad.exe"/>
    </Monitor>
</Agent>
```

- **`WindowsDebugger` monitor**：用调试器**启动并挂住** notepad，一旦目标异常（访问违例等），它 `DetectedFault()==true`，然后 `GetMonitorData()` 把崩溃栈交回去（走 `CollectFaults` 链路）。
  - `CommandLine`：调试器要启动的命令——`notepad.exe fuzzfile.bin`，即拿 notepad 打开我们写的那个 fuzz 文件。
  - **`StartOnCall="ScoobySnacks"`**：⭐关键同步机制。它告诉调试器**别急着启动 notepad，等状态机里那个 `method="ScoobySnacks"` 的 call 动作触发了再启动**。为什么？保证**文件已经 output+close 写好了**，notepad 才去读，避免竞态。
- **`PageHeap` monitor**：开启页堆调试，让堆越界这类内存错误**更早、更容易**暴露成崩溃（否则可能悄悄写坏内存不崩）。
- 一个 Agent 下**挂多个 Monitor** 是常态：一个负责启动+抓崩溃，一个负责放大内存错误。

### Test：把所有东西接线组装（81-94）

```xml
<Test name="Default">
    <Agent ref="LocalAgent" />            <!-- 用哪个 agent -->
    <StateModel ref="State"/>             <!-- 用哪个状态机 -->
    <Publisher class="File">              <!-- ★ 关键：IO 方式 = 写文件 -->
        <Param name="FileName" value="fuzzfile.bin" />
    </Publisher>
    <Logger class="Filesystem">           <!-- 崩溃/日志存哪 -->
        <Param name="Path" value="logtest" />
    </Logger>
</Test>
```

- `<Publisher class="File">`：这就是为什么 `output` 动作变成了「写文件」而不是「打印控制台」（HelloWorld 里是 `Console`）。**换目标就换这一行。**
- `<Logger class="Filesystem">`：抓到 fault 时，把崩溃样本、栈信息落盘到 `logtest/` 目录。

---

## ⭐ 5 处「接线」——你自己写 PIT 最容易错的地方

这份 PIT 能跑通，靠的是 5 组**名字必须对上**。这也是 DOM 对象图里那些交叉引用的来源：

| # | 引用 → 被引用 | 值 |
| --- | --- | --- |
| 1 | `Action DataModel ref` → `DataModel name` | `TestTemplate` |
| 2 | `Test StateModel ref` → `StateModel name` | `State` |
| 3 | `Test Agent ref` → `Agent name` | `LocalAgent` |
| 4 | **`Action call method`** → **`Monitor StartOnCall`** | `ScoobySnacks` |
| 5 | **`Publisher FileName`** → **`Monitor CommandLine` 里的文件名** | `fuzzfile.bin` |

**第 4、5 组尤其致命：**

- 第 5 组：publisher 写的文件名和调试器命令行里要打开的文件名**必须是同一个**，否则 notepad 打开的是个空文件，永远 fuzz 不到东西。
- 第 4 组：call 的 method 名和 `StartOnCall` **必须一致**，否则调试器不知道什么时候该启动 notepad，同步就断了。

---

## 这份 PIT 跑一轮，在 runTest 里怎么走

```
Engine.runTest 第 N 轮：
  块D  mutationStrategy.Iteration = N        → 决定这轮变 TestTemplate 里哪个字节
  块E  stateModel.Run()
        └ State "Initial":
            Action output → File.output()     写 fuzzfile.bin（含本轮变异）
            Action close  → File.close()       落盘
            Action call   → AgentManager.Message("ScoobySnacks")
                            → WindowsDebugger 收到，启动 notepad.exe fuzzfile.bin
  块F  IterationFinished()
  块H  OnCollectFaults()
        └ WindowsDebugger.DetectedFault()?
            崩了 → GetMonitorData() 交回崩溃栈 → context.faults
  块I  有 fault → OnReproFault → 开复现，continue 重放这一轮确认
  块N  收尾：stop publisher / 停 monitor
```

**你前面学的主循环，在这份真实 PIT 上就这么具体地跑起来了。**

---

## 实战改造提示

如果你要拿它改成 fuzz 自己的目标，通常只动 4 个地方：

1. **`DataModel`**：换成你真实文件格式的建模（最影响效果）；
2. **`Data`**：加 `<Data fileName="种子样本"/>` 给个真实起点（这份样例没写，等于从 `"Hello World!"` 硬变）；
3. **`Publisher FileName` + `Monitor CommandLine`**：改成你的文件名和你的目标程序；
4. **`Monitor StartOnCall`**：保持和 call method 一致即可。
