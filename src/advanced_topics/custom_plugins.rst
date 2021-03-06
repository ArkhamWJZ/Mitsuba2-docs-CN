.. _sec-custom-plugins:

在 Python 中自定义插件
========================

Mitsuba 2提供了一种机制可以 *直接在 Python 中* 实现自定义插件。如需这么做，只需扩展一个基类（例如 `BSDF`， `Emitter`）
并且覆盖其类方法即可（例如，``sample()``， ``eval()`` 等）。这充分利用了 *pybind11* 的  `trampoline feature <https://pybind11.readthedocs.io/en/stable/advanced/classes.html#overriding-virtual-functions-in-python>`_ 特性。

随后可以通过调用 ``register_<type>(name, constructor)`` 函数向 XML 解析器中注册新的插件，使其可以像其他 C++ 插件一样
可以在 XML场景描述中随意访问。

.. warning::

    在使用 Python 实现系统的核心组件时，只有在 ``gpu_*`` 变体下才会有合理的性能表现（因为它们可以和其他系统组件一起
    被 JIT 编译）。我们计划在未来将这一功能扩展到更多变体模式。

    目前，在 Python 中只能实现 ``BSDF``，``Emitter`` 和 ``Integrator`` 插件。

本章节的其余部分将讨论这几个插件的相应示例：


BSDF example
------------

在本例中，我们将通过扩展 :code:`BSDF` 基类，在 Python 中实现一个简单的漫反射 BSDF。代码与 C++ 中实现的
漫反射 BSDF 非常相似（参见 :code:`src/bsdf/diffuse.cpp`）。

BSDF 类需要实现以下三个方法：:code:`sample`，:code:`eval` 和 :code:`pdf`：

.. literalinclude:: ../../examples/04_diffuse_bsdf/diffuse_bsdf.py
   :language: python
   :lines: 14-64

在定义这个新的 BSDF 类之后，我们需要将其注册为插件。这会允许 Mitsuba 在加载场景时实例化这个新的 BSDF：

.. literalinclude:: ../../examples/04_diffuse_bsdf/diffuse_bsdf.py
   :language: python
   :lines: 66

:code:`register_bsdf` 函数接收新插件的名称和构造新实例的函数。在这之后，我们就可以在 XML 场景文件中使用
我们自定义的 BSDF 了。

.. code-block:: xml

    <bsdf type="mydiffusebsdf"/>

随后可以通过调用标准函数 :code:`render` 来进行渲染场景：

.. literalinclude:: ../../examples/04_diffuse_bsdf/diffuse_bsdf.py
   :language: python
   :lines: 68-77

.. note:: The code for this example can be found in :code:`docs/examples/04_diffuse_bsdf/diffuse_bsdf.py`


Integrator example
------------------

在本例中，我们将通过相同的机制实现自定义的直接光照集成器。生成的图像将具有逼真的阴影和着色，但是没有全局照明。

渲染的主要过程可以由大约30行代码实现：

.. literalinclude:: ../../examples/03_direct_integrator/direct_integrator.py
   :language: python
   :lines: 22-55

代码与 C++ 代码实现的直接光照集成器非常相似（参见 :code:`src/integrators/direct.cpp`）。
该函数将当前场景、采样器和光线数组当作参数。:code:`active` 参数指定了哪些通道是被激活的。

我们首先将提供的光线与场景求交，并估算发光体的直接光强。随后，我们在发光体上显式采样位置，并估算它们的贡献。
随后我们根据 BSDF 采样光线方向，并计算这些光线是否击中了其他发光体。接着通过多重重要性采样将这些贡献组合在
一起以减少方差。

此函数将针对不同的光线序列进行调用，因为每条光线都有可能击中具有不同 BSDF 的曲面。因此 :code:`bsdf = si.bsdf(rays)` 
是一个由不同的 BSDF 指针组成的数组。为了调用这些不同 BSDF 的成员函数，我们通过矢量化方法调用这些特殊的分派函数：

.. literalinclude:: ../../examples/03_direct_integrator/direct_integrator.py
   :language: python
   :lines: 38-39

这确保每个变体都能正确调用由 C++ 或 Python 实现的 BSDF 模型。除此之外，代码和使用的接口与 C++ 版本有关。
请参阅有关 C++ 类型的文档，以获得更多关于函数和对象的信息。

当在 :ref:`custom rendering pipelines <sec-rendering-scene-custom>` 这一章节实现深度集成器时，
如要正确的从相机中采样光线和实现胶片的 “飞溅” 需要耗费大量的工作。虽然这对某些程序来说非常有用，但是也有些单调乏味。
许多情况下我们只需要实现一个自定义的集成器，并不需要考虑光线是如何准确生成的。因此，在本例中我们将使用一种更为优雅的
机制，它允许简单的扩展 :code:`SamplingIntegrator` 基类。这个类是所有蒙特卡洛积分器的基类（例如，路径追踪器）。
通过这个类来定义我们的积分器，随后我们可以依赖 Mitsuba 的现有机制来正确的实现相机光线采样和像素值“飞溅”。

由于我们已经使用正确的接口定义了函数 :code:`integrator_sample`，因此定义一个新的积分器是非常简单的：

.. literalinclude:: ../../examples/03_direct_integrator/direct_integrator.py
   :language: python
   :lines: 58-70

该积分器不仅返回颜色值，而且还会额外渲染出场景深度。当使用此积分器时，Mitsuba 将输出额外带有一个深度通道的多通
道 EXR 文件。 :code:`aov_names` 方法将用于命名任意输出变量（AOVs）。例如，可以使用相同的机制来输出曲面法线。
如果不需要 AOV，只需要返回一个空列表。

在定义一个新的积分器类之后，我们必须将其注册为插件。这允许 Mitsuba 在加载场景时实例化这个新的积分器：

.. literalinclude:: ../../examples/03_direct_integrator/direct_integrator.py
   :language: python
   :lines: 74

:code:`register_integrator` 函数接受新的插件名称和新实例的构造函数。随后，我们就可以在场景中使用指定的新积分器：


.. code-block:: xml

    <integrator type="mydirectintegrator"/>

然后可以通过标准 :code:`render` 函数渲染场景了：

.. literalinclude:: ../../examples/03_direct_integrator/direct_integrator.py
   :language: python
   :lines: 76-85

.. note:: 此示例代码可以在 :code:`docs/examples/03_direct_integrator/direct_integrator.py` 中找到。
