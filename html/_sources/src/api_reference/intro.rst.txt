.. _sec-api:

介绍
============

本章的 API 相关文档将通过 `Autodoc <http://www.sphinx-doc.org/en/master/usage/extensions/autodoc.htm>`_ Sphinx
扩展自动生成。

Autodoc 自动处理 Mitsuba 的 Python 绑定文档，因此所有 C++ 函数和类签名都是从相应 Python 副本文档中记录的。
Mitsuba 绑定尽可能地模拟 C++ API，因此即使对 C++ 开发人员该文档也应该是有价值的。

整个文档分为三个子模块：

- :ref:`sec-api-core`: 与渲染算法没有直接关系的核心功能模块。
- :ref:`sec-api-render`: 组件接口，比如： rendering algorithm， sensor， emitter， texture， participating media 等。
- :ref:`sec-api-python`: Python 编写的高层功能模块：自动微分的基础结构，测试（卡方测试）等。

.. warning::

    本文档是在 ``scalar_rgb`` 变体模式下提取出来的。其他变体模式在许多函数和属性签名中使用的是不同的类型。
    对于这些不同之处从上下文来看应该是显而易见的，所以问题不大。例如，``scalar_spectral`` 变体模式下应该使用
    光谱颜色代替所有的 RGB 类型颜色，``gpu_rgb`` 应该使用 Enoki CUDA 数组代替所有的浮点数标量等等。