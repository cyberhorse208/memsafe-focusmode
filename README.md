# Native内存安全测试-专注模式
## 为什么引入专注模式
  进行native内存安全测试时，一旦发生内存错误，通常都会导致应用崩溃。这会打断测试进程，带来测试效率的降低：

1. 发生错误后程序即崩溃，必须解决后才能继续测试发现问题，降低了测试效率，测试过程较长
2. 崩溃点如果在开源/三方so，它们难以修改；在不修改崩溃点情况下，无法运行到自有代码，也就无法识别自有代码的内存错误

针对上述问题，AOSP和APP层面目前可能的对策是：

1. AOSP：在14中引入“permissive"模式，此模式下，MTE触发内存错误后，可以关闭当前线程的MTE检测，不导致进程崩溃。其问题也显而易见，MTE的关闭导致本线程无法继续检测问题，内存安全测试相当于关闭了。
2. APP：捕获系统信号，自行处理后，不转发给系统处理，从而避免系统信号处理函数将应用崩溃。这种方法需要修改自定义信号处理方法，另外还旁路了系统信号处理，相应的处理也被跳过（比如tombstone文件不会生成）

## 专注模式有什么功能

1. 发生MTE识别的SIGSEGV时，立即关闭MTE，一段时间后重新打开MTE，继续检测
2. 崩溃堆栈包含指定so时，跳过此次崩溃，无tombstone生成
3. 崩溃堆栈与上一次崩溃堆栈一致时，跳过此次崩溃，无tombstone生成
4. 拦截发送给AMS的信号，避免AMS杀进程
5. 旁路APP自定义崩溃处理流程，避免处理流程中杀死进程，APP无需修改自身崩溃SDK的行为


## 如何使用

​	前置条件：

 	1. 支持MTE的设备
 	2. 通过OTA升级了支持专注模式的系统版本

​	专注模式通过属性开关控制，各开关功能如下：

| 属性名称|默认值|属性值|意义|生效时机|持续周期|
| ----------| ------ | ------ | ------------------- | -------- | ------------------ |
| debug.mte.failover                                  | false  | true   | 进入failover模式                                             | 立刻     | 本次开机，重启失效 |
|                                                     |        | false  | 关闭failover模式                                             | 立刻     | 本次开机，重启失效 |
| debug.mte.sigsegv.no.userhandler                    | false  | true   | 让进程设置signal handler操作不生效，发生SIGSEGV时，进程自定义的崩溃处理程序不会执行，避免有些崩溃处理程序的自杀行为。 | 重启应用 | 本次开机，重启失效 |
|                                                     |        | false  | 恢复系统默认行为，进程设置signal handler操作生效             | 重启应用 | 本次开机，重启失效 |
| debug.mte.failover.delayseconds                     | 1      | n      | 设置mte延迟开启等待n秒，默认值为1秒。仅debug.mte.failover=true时才生效。 **一般情况下无需设置** | 立刻     | 本次开机，重启失效 |
| debug.mte.notcrash.sigsegv                          | false  | true   | 开启MTE时，系统不让app在SIGSEGV上崩溃。**一般情况下无需设置**，因为这会让非MTE检测出的SIGSEGV信号也被系统忽略 | 立刻     | 本次开机，重启失效 |
|                                                     |        | false  | 非MTE检测出的SIGSEGV信号不被系统忽略，按系统原生逻辑处理，可能导致应用崩溃 | 立刻     | 本次开机，重启失效 |
| debug.mte.notcrash.any                              | false  | true   | 开启MTE时，系统不让app在任何类型信号上崩溃。**一般情况下无需设置**，因为这会让SIGSEGV外的任何信号都被系统忽略 | 立刻     | 本次开机，重启失效 |
|                                                     |        | false  | SIGSEGV外的任何信号不被系统忽略，按系统原生逻辑处理，可能导致应用崩溃 | 立刻     | 本次开机，重启失效 |
| debug.mte.sigsegv.ignoreframe                       | -      | 白名单 | 设置白名单，","分隔，当发生内存错误时的堆栈包含白名单中任意值时，忽略此次错误，无tombstone生成。**适应于忽略三方库产生的任何错误。**最多设置10个，每个长度最多30B | 立刻     | 本次开机，重启失效 |
| debug.mte.sigsegv.ignoreframe.process.{processname} | -      | 白名单 | 针对某个进程{processname}生效，将覆盖debug.mte.sigsegv.ignoreframe的值最多设置10个，每个长度最多30B | 立刻     | 本次开机，重启失效 |
| debug.mte.sigsegv.checkframes                       | -      | n      | 设置发生内存错误时，比较与上一次发生错误时的堆栈前n个帧，如果都相同，则说明在相同的位置发生内存错误，忽略本次错误，不生成tombstone文件，保存本次堆栈。n的最大值为256 | 立刻     | 本次开机，重启失效 |
| debug.mte.sigsegv.checkframes.process.{processname} | -      | n      | 针对某个进程{processname}生效，将覆盖debug.mte.sigsegv.checkframes的值 | 立刻     | 本次开机，重启失效 |

