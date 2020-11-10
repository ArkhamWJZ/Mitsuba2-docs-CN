.. _sec-devguide:

Introduction
============

我们希望在开发者查阅任何 Mitsuba 代码之前，先 *仔仔细细* 阅读 `Enoki <https://enoki.readthedocs.io/en/master/index.html>`_ 
的文档。Enoki 是一个用于向量和矩阵运算的模版库，它构成了 Mitsuba 的基础。它还可以驱动代码转换，从而实现系统矢量化和自动微分的渲染器。

Mitsuba 2是一个全新的代码库，现有的 Mitsuba 0.6 插件需要进行重大修改才能与新的系统架构兼容。除了整体架构不同之外，一个小小的变化就是
Mitsuba 2代码中对函数名和变量的命名约定使用的是 ``下划线命名`` （形成鲜明对比的是，Mitsuba 0.4到处使用的是 ``驼峰命名``）。
实际上我们还将 Python 端的 `PEP 8 <https://www.python.org/dev/peps/pep-0008>`_ 规范带入到了 C++ 端（没有特别指定命名约定），
这确保了两种语言的代码看起来都特别自然。


Code structure
--------------

Mitsuba 分为3个基本支持库：

* 核心库（在 :code:`src/libcore` 中）实现了一系列基本功能如，跨平台文件、bitmap I/O、数据
  结构、调度以及日志和插件管理。
* 渲染库（在 :code:`src/librender` 中）包含了加载和表示光源、形状、材质和参与介质等场景资源
  所需的抽象类。
* Python 库（在 中）包含了用 Python 编写的系统组件，这些组件通过绑定访问 Mitsuba 。包含了统计
  测试（ Chi^2 等）以及可微渲染的工具。


