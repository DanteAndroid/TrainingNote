# Test apps on Android

测试你的app是整个app开发过程中必不可少的部分。通过对你的app持续地跑tests，可以在发布之前验证其正确性，功能性行为以及可用性。

测试还有下列优点：

- 出故障的时候有**快速的反馈**
- 开发周期中**及早发现故障**
- **更安心的代码重构**，让你不必担心代码倒退（regressions）
- **稳定的开发速度**，帮助你最小化技术上的亏欠



# Fundamentals of Testing

用户会与你的app在各种各样的层面上交互，从点击提交按钮到下载东西到他们的设备上。相应地，在迭代地开发app的时候，你应该进行多样化的用例测试和交互。

## 使用迭代的开发流程

当你的app不断扩展，你会发现种种需求：有从服务器获取数据，与设备传感器交互，访问本地存储，或是渲染复杂的UI。App的多功能需要一个复杂的测试策略。

迭代地开发一个新的特性的时候，你要么写个新测试要么在已有的单元测试里加新的用例和断言。测试一开始会失败，因为新特性还没实现。

设计新特性的时候去考虑职责的单元很重要。对于每个单元，你都得写个相应的单元测试。你的单元测试应该几乎涵盖与该单元所有可能性的交互，包括标准交互，无效输入和资源不可用的情况。

![testing development cycle](https://developer.android.com/images/training/testing/testing-workflow.png?hl=zh-cn)

如图一，两个与迭代的、测试驱动开发紧密相关的循环

## 理解测试金字塔

![The testing pyramid](https://developer.android.com/images/training/testing/pyramid.png?hl=zh-cn)

如图二，测试金字塔演示了你的app应该如何包括三种类别的测试：

- 小型测试：可以从生产系统中独立出来运行的测试。一般模拟了每个主要组件，并可以快速地在你的机器上运行
- 中型测试：介于小和大型测试中间的集成测试。整合了一些控件，运行在模拟器或者真机上。
- 大型测试：运行完整的UI流程的集成测试。可以确保终端用户的任务在模拟器或真机上如期运行。

尽管小型测试快速且集中，可以让你快速定位问题，但是他们低保真且独立，使得很难确信通过小测试的app就是正常的。写大型测试的时候你会遇到相反的一组权衡。

由于每个测试类别的不同特性，你的测试应该覆盖金字塔的每个层级。尽管每个类别的部分测试可能基于你app的用例而相异，我们还是推荐这样测试类别的比例：**70%小测试，20中型测试，10%大型测试**。

关于android测试金字塔可以查看 [Test-Driven Development on Android](https://www.youtube.com/watch?v=pK7W5npkhho&start=111&hl=zh-cn) 系列视频。

## 写小型测试

当你添加和更改app的功能的时候，通过针对性地创建和运行单元测试来确保这些特性如预期运行。尽管也可在设备或模拟器上评估单元(evaluate units)，通常在开发环境中测试更快捷，需要与android系统交互的时候可以添加短方法(stubbed)或者模拟方法(mocked)。

### Robolectric

如果你app的测试环境需要与android框架频繁交互，可以用 [Robolectric](http://robolectric.org/) 。这个工具可以执行测试友好，模拟android框架的基于java逻辑的短方法(stubs)。这些短方法由社区来维护。

Robolectric 测试几乎匹配了在android设备上跑的测试的所有功能(fidelity)，同时还比设备上测试执行地更快。还支持android平台的下列方面：

- 组件的生命周期
- 事件循环
- 所有资源

### 模拟对象

你可通过针对修改版的`android.jar`跑单元测试来实现监控你app与android框架交互的元素（也就是app调用Android API的方法）。这个 JAR 文件不包含任何代码，所以你的app与android框架的调用会默认抛出异常。为了测试与android系统交互的代码，使用 [Mockito](http://mockito.org/) 之类的框架来设置模拟对象。

如果你的代码包含对资源的引用或是与android系统复杂的交互，你应该用其他形式的单元测试，如 Robolectric.

### 仪器型单元测试

你还可以跑 instrumented 单元测试在真机或者模拟器上，这不会涉及任何模拟或者 stubbing of the framework. 因为这种测试比本地测试执行的慢得多。然而，仅当针对实际设备评估你的app的行为是必需的时候，用这个方法最好。

## 写中型测试

你在开发环境里测试完app的每个单元之后，应该验证运行在模拟器真机上组件的行为是否正常。中型测试就是为此而生。如果你的app的某些组件依赖于硬件，创建并进行这些测试尤其重要。

中性测试评估你的app如何协调多个单元，但是不测完整app。中型测试的示例包括服务测试，整合测试和可以模拟外部依赖行为的封闭UI测试。

典型地，最好在模拟器或者云服务如[Firebase Test Lab](https://firebase.google.com/docs/test-lab/?hl=zh-cn)上面测试你的app，而不是在实机上，这样你快捷地可以测试多种屏幕尺寸和硬件配置的设备。

## 写大型测试

尽管在你的app里独立地测试每个层级和特性很重要，涉及完整的任务的常规流程和用例测试也同样重要，依照商业逻辑，从UI到数据层面。

如果你的app足够小，你可能只需要一套大型测试来评估app的整体功能。否则，你应该通过团队负责人，功能层级，或者用户目标来划分大型测试。

[`AndroidJUnitRunner`](https://developer.android.com/reference/android/support/test/runner/AndroidJUnitRunner.html?hl=zh-cn) 类定义了基于仪器型的 JUnit test runner，可以让你在android设备上跑 JUnit 3- or JUnit 4 的测试类。这个 test runner 加快了你的测试包和app在设备上的测试速度。

[`AndroidJUnitRunner`](https://developer.android.com/reference/android/support/test/runner/AndroidJUnitRunner.html?hl=zh-cn) 还支持下列android测试的工具和框架：

### JUnit4 Rules

ATSL 包含了你测试里涉及的关键组件的生命周期的管理代码，例如 Activities 和 Services。要学习如何定义这些 rules，请看 [JUnit4 Rules](https://developer.android.com/training/testing/junit-rules.html?hl=zh-cn)。

### Espresso

[Espresso](https://developer.android.com/training/testing/espresso/index.html?hl=zh-cn) 在自动化下列app内交互的时候同步了异步的任务：

- 在view上执行操作
- 完成跨进程的流程（仅支持 Android8.0 +）
- 估算障碍人士如何才能使用你的app
- 定位和激活 RecyclerView 与 [AdapterView](https://developer.android.com/reference/android/widget/AdapterView.html?hl=zh-cn) 对象内的项目
- 证实送出(outgoing)的intent的状态
- 验证 Webview 对象内的 DOM 结构
- 追踪你app内的长时间运行的后台操作

### UI Automator

> 注意：仅当你的app必须与系统交互来满足苛刻的用例时我们才推荐用 UI Automator。因为 UI Automator 与系统app和UI交互，每次系统更新你都得修复和重新跑  UI Automator 测试。这些更新包括 Android 版本更新和 Google Play services 更新。
>
> 作为使用 UI Automator 的替代品，推荐添加密闭测试或者分割大型测试为一套小的和中型测试。特别地，一次关注测试一整块app内通信，比如比如向其他app发消息并处理相应的intent结果。[Espresso-Intents](https://developer.android.com/training/testing/espresso/intents.html?hl=zh-cn) 可以帮你写这些小的测试。

UI Automator 框架代表你的app来与系统app交互，如检查当前界面的层级，截屏，和分析当前设备的状态。进一步学习 UI Automator 如何观察测试下的app，查看 [UI Automator](https://developer.android.com/training/testing/ui-automator.html?hl=zh-cn)。

