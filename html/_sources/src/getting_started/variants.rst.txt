.. _sec-variants:

选择变体
=================

Mitsuba 2是一个可被用户重新定义的渲染系统，通过提供一系列不同的 "*变体 variants*" 以改变系统在模拟时的表现，
诸如改变颜色的表现形式为单色光，RGB，频谱，甚至可以是偏振光照明（polarized illumination）。 
类似的，在渲染过程下的数值处理形式也可以进行更换，如使用更高的准确度，矢量化一次处理多条光路，或者在数学上求解
反问题。 所有这些变体（variants）都是从通用代码库中自动创建的。

目前为止有多达36种不同的渲染器变体可用，如下面列表所示。 因此，在构建 Mitsuba 2之前，你需要确定其中哪些是
与你想要的应用功能相关的。

  .. container:: toggle

      .. container:: header

          **Show/hide available variants**

      - :monosp:`scalar_mono`
      - :monosp:`scalar_mono_double`
      - :monosp:`scalar_mono_polarized`
      - :monosp:`scalar_mono_polarized_double`
      - :monosp:`scalar_rgb`
      - :monosp:`scalar_rgb_double`
      - :monosp:`scalar_rgb_polarized`
      - :monosp:`scalar_rgb_polarized_double`
      - :monosp:`scalar_spectral`
      - :monosp:`scalar_spectral_double`
      - :monosp:`scalar_spectral_polarized`
      - :monosp:`scalar_spectral_polarized_double`
      - :monosp:`packet_mono`
      - :monosp:`packet_mono_double`
      - :monosp:`packet_mono_polarized`
      - :monosp:`packet_mono_polarized_double`
      - :monosp:`packet_rgb`
      - :monosp:`packet_rgb_double`
      - :monosp:`packet_rgb_polarized`
      - :monosp:`packet_rgb_polarized_double`
      - :monosp:`packet_spectral`
      - :monosp:`packet_spectral_double`
      - :monosp:`packet_spectral_polarized`
      - :monosp:`packet_spectral_polarized_double`
      - :monosp:`gpu_mono`
      - :monosp:`gpu_mono_polarized`
      - :monosp:`gpu_rgb`
      - :monosp:`gpu_rgb_polarized`
      - :monosp:`gpu_spectral`
      - :monosp:`gpu_spectral_polarized`
      - :monosp:`gpu_autodiff_mono`
      - :monosp:`gpu_autodiff_mono_polarized`
      - :monosp:`gpu_autodiff_rgb`
      - :monosp:`gpu_autodiff_rgb_polarized`
      - :monosp:`gpu_autodiff_spectral`
      - :monosp:`gpu_autodiff_spectral_polarized`


需要注意的是，编译时间和编译内存的占用量与启用的变体数目成正比， 因此包含过多的变体数目（如超过5个）
是不可取的。 Mitsuba 2 开发者通常会想要限制当前实验使用1-2个系统变体，以最大程度的减少编辑后的重新编译时间。
每一个变体都与一个标识名称相关联，该名称由几个部分组成：

.. image:: ../../images/variant.svg
    :width: 80%
    :align: center

接下来我们将依次讨论每一部分的含义。

Part 1: Computational backend
-----------------------------

计算后端部分控制系统如何去实现加法或者乘法等基本运算模块。有以下选项可以选择：

-  ``scalar`` 后端在 CPU 上使用和旧版 Mitsuba 类似的普通浮点数运算。
  这是在命令行上使用 :monosp:`mitsuba` 指令或者通过图形用户界面生成渲染图时的默认设置。

-  ``packet`` 后端使用SSE4.2、AVX、AVX2 及 AVX512扩展指令集，可以高效的对4、8或16的
  浮点数组进行计算。因此在 packet 模式下, 渲染算法中的每个单一操作（光线追踪， BSDF 采样等）
  将一次对多个输入进行操作。下面👇可视化出来的光路很好的展示出了scalar 和 packet 两种模式的
  不同之处：

  .. image:: ../../../resources/data/docs/images/variants/vectorization.jpg
      :width: 100%
      :align: center

  然而，需要注意的是，packet 模式不是万能解药：标准算法不会就此简单的加快8倍或16倍。
  Packet 模式需要特殊的算法设计，而且意在给那些能利用并行特性的软件开发者使用。

-  ``gpu`` 后端通过使用 `Enoki's   <https://github.com/mitsuba-renderer/enoki>`_ 
  just-in-time (JIT) 编译器将计算转移到 CUDA 内核中。 通过使用这个后端，每个操作通常需要
  同时操作数百万个输入。之后 Mitsuba 就变成了我们熟知的 *波阵面(wavefront)路径追踪器* 其中
   GPU 上的光线追踪部分委托给了 NVIDIA 的 OptiX 库。 需要注意的是，我们需要相对较新的 NVIDIA GPU 架构: 
  最好是 *Turing* 或者其他更新的版本。较旧的 *Pascal* 架构也是支持的，但速度往往会比较慢，因为它
  缺乏光线跟踪的硬件加速。

- 在 ``gpu`` 后端构建项目时，此外可以使用 ``gpu_autodiff`` 通过模拟器传播导数信息, 这是渲染算法
  解决 *逆问题* 的关键要素。

  下面展示了来自 :cite:`NimierDavidVicini2019Mitsuba2`的示例。在这里，Mitsuba 2 被用来计算透明玻璃
  板的高度轮廓，该玻璃板可以折射红、绿、蓝光，以此来重现指定的彩色图像。

  .. image:: ../../../resources/data/docs/images/autodiff/caustic.jpg
      :width: 100%
      :align: center

   ``gpu_autodiff`` 后端的主要使用示例就是 *可微渲染*， 可微渲染将渲染算法解释为一个函数
  :math:`f(\mathbf{x})` 将输入 :math:`\mathbf{x}` (场景描述信息) 转化为输出 :math:`\mathbf{y}` (渲染结果)。
  随后对此函数 :math:`f` 求微分得到 :math:`\frac{\mathrm{d}\mathbf{y}}{\mathrm{d}\mathbf{x}}`，这提供了一个
  关于输出 :math:`\mathbf{y}` (渲染结果) 随着输入 :math:`\mathbf{x}` (场景描述信息) 改变而理想化改变的一阶近似。
  再加上一个可微 *目标函数* :math:`g(\mathbf{y})` 来量化场景参数的适用性再再搞一个梯度优化算法，这样一个可微渲染器就
  可以用来解决涉及到光的复杂逆问题了。

  .. image:: ../../../resources/data/docs/images/autodiff/autodiff.jpg
      :width: 100%
      :align: center
  文档提供了几个关于 :ref:`differentiable and inverse rendering <sec-inverse-rendering>`。的应用示例。

一个关于 ``packet``， ``gpu``，和 ``gpu_autodiff`` 模式的吸引人的地方在于，它们公开了对任意大的输入集进行矢量化操作
的 Python 接口（即使在小容量序列上工作 ``packet`` 模式下，C++ 还是实现覆盖了较大的输入）。 这意味着可以通过
一个 Python 函数调用执行数百万次光线追踪操作或 BSDF 计算，从而在 Python 或 Jupyter notebook中实现高效的原型设计，
而无需对许多元素进行昂贵的迭代操作。

如何去选择合适的变体模式？
^^^^^^^^^^^^^^

我们一般建议通过命令行渲染时编译 ``scalar`` 变体，至于 ``packet`` 或 ``gpu_autodiff`` 变体则是为 Python 开发准备
的---而后者只有在用到可微渲染时才会被需要。

Part 2: Color representation
----------------------------

下一部分决定了 Mitsuba 如何表示颜色信息。有以下选项可供选择：

- ``mono`` 模式完全禁用了颜色的概念，这在模拟具有固定单一颜色的场景时会十分有用（例如激光照明）。
  这种模式非常适合编写那些颜色完全不相关的测试用例。当输入场景提供颜色信息时，:monosp:`mono` 
  模式会自动将其转换为灰度。

- ``rgb`` 模式使用基于 RGB 的颜色表示。这是一个个合乎逻辑的默认选项，并且符合上一代 Mitsuba 的默认
  行为。而在另一方面，RGB 模式可能与现实世界中的颜色工作方式不太接近。 请点击以下内容查看更详细的解释。

    .. container:: toggle

        .. container:: header

            **基于 RGB 渲染的问题 (点击展开)**

        **基于 RGB 表示 的颜色存在的问题：** RGB 渲染算法通常执行两种与颜色相关的操作:逐分量相加以组
        合不同的光源，在模型间相互反射时 RGB 颜色向量逐分量相乘。虽然加法很好，但 RGB 乘法结果是一个荒
        谬的操作，根据底层 RGB 颜色空间，它可以给出非常不同的答案。

        假设我们在 sRGB 颜色空间中渲染一个场景，有一个辐射度为:math:`[0, 0, 1]` 的绿色光源，其在一个
        绿色平面上的反射率是 :math:`[0, 0, 1]`。逐分量相乘 :math:`[0, 0, 1] \otimes [0, 0, 1] = [0, 0, 1]` 
        的结果诉我们没有光被该绿色表面吸收。到目前为止还不错。

        接下来让我们切换到一个更大一些的颜色空间 *Rec. 2020*。这时同样的绿色不在位于色域的边界上，而是处于色域内部的
        某个地方。

        .. image:: ../../../resources/data/docs/images/variants/rgb-mode-issue.svg
            :width: 100%
            :align: center

        简单起见，我们假设其坐标为 :math:`[0, 0, \frac{1}{2}]` 。根据与之前相同的计算过程 :math:`[0, 0, \frac{1}{2}]\otimes[0, 0, \frac{1}{2}]=[0, 0, \frac{1}{4}]` 
        我们能得到一个结论：有一半的光被表面吸收掉了，而这恰恰说明了 RGB 乘法的问题。这个问题的解决方案就是在光谱域中进行颜色相乘。


- 最后要说的是，``spectral`` 模式下生成了一个横贯可见范围的全光谱颜色表示 :math:`(360\ldots 830 \mathrm{nm})`。
  波长的维度可以被简单地看作是渲染算法整合的光路空间的另一个维度。

  这提高了计算的精度，特别是在光谱测量数据可用的情况下。例如，以下两个 Cornell box 的渲染结果：
  左侧的渲染图，首先将所有材质的光谱反射率数据转换成 RGB ，随后使用 ``scalar_rgb`` 变体进行渲染，
  从而生成了具有欺骗性的彩色渲染图。相比之下，能够正确解释光谱特性的 ``scalar_spectral`` 变体模式，
  渲染出来的结果更为柔和。

  .. subfigstart::
  .. subfigure:: ../../../resources/data/docs/images/render/variants_cbox_rgb.jpg
     :caption: RGB Mode
  .. subfigure:: ../../../resources/data/docs/images/render/variants_cbox_spectral.jpg
     :caption: Spectral Mode
  .. subfigend::
     :label: fig-cbox-spectral

  需要注意的是，即使在光谱模式被激活时，Mitsuba 在默认情况下仍会生成 RGB 输出图像。除此之外，许多
  现有的 Mitsuba 场景只提供 RGB 颜色信息。不过光谱模式下 Mitsuba 仍然可以渲染这些的场景，
  它可以确定指定 RGB 颜色所对应的近似平滑光谱 :cite:`Jakob2019Spectral`。

Part 3: Polarization
--------------------

如果需要的话，Mitsuba 2 可以追踪光的整个偏振状态。偏振是指光的振荡方向垂直于传播方向的特性，属于电磁波特性。
振荡的形状有无数多种，下面的图片展示的是 *水平型* 和 *椭圆型* 偏振的例子。

.. image:: ../../images/polarization_wave_variations.svg
    :width: 90%
    :align: center

因为人类感觉不到偏振，所以在渲染视觉效果逼真的图像时，通常不需要考虑偏振。然而，使用各种测量设备和
摄像机可以很容易地观察到偏振，并且偏振往往可以提供丰富的关于可见物体材料和形状之类的信息。因此，偏
振是解决渲染逆问题的有力工具，这也是我们在 Mitsuba 2 中选择支持这一功能特性的原因之一。需要注意的是，
让系统考虑到偏振是有时间代价的---大约要花费渲染时间的1.5到2倍。

在光传播的模拟中，*Stokes vectors* 被用来模拟横向椭圆型振动的参数化，
 *Mueller matrices* 被用来计算表面散射对偏振对影响 :cite:`Collett1993PolarizedLight`。
更多有关偏振渲染模式的实现细节，请参考开发者手册 :ref:`developer_guide-polarization` 章节。


Part 4: Precision
-----------------

Mitsuba 2通常依赖于单精度(32位)运算，但也可以选择双精度(64位)。我们发现这在 debug 时会特别有用：
在切换到双精度后，通常可以确定观察到的问题是否是由于浮点不精确而引发的。需要注意的是，双精度目前不适
用于 ``gpu_*`` 变体。这是因为 OptiX 是以单精度执行光线追踪部分的。

Configuring :monosp:`mitsuba.conf`
----------------------------------

Mitsuba 2 的变体模式在 :monosp:`mitsuba.conf` 文件中指定。开始之前，首先将默认模板复制到 Mitsuba 2
本地存储仓库的根目录。

.. code-block:: bash

    cd <..mitsuba repository..>
    cp resources/mitsuba.conf.template mitsuba.conf

接下来，在你喜欢的文本编辑器中打开 :monosp:`mitsuba.conf` ，然后滚动到启用变体的声明
部分（第70行附近）

.. code-block:: text

    "enabled": [
        # The "scalar_rgb" variant *must* be included at the moment.
        "scalar_rgb",
        "scalar_spectral"
    ],

默认情况下，指定了两个 scalar 变体，你可能想要根据你的需要和上面的解释来扩展这两个变体。
请注意，可以移除 ``scalar_spectral`` ，但``scalar_rgb`` *目前必须* 是列表的一部分，
因为 Mitsuba 的一些核心组件依赖于它。你也许想要改变在没有显式指定变体时执行的 *默认变体* 
（这必须在 ``enabled`` 列表中）

.. code-block:: text

    # If mitsuba is launched without any specific mode parameter,
    # the configuration below will be used by default

    "default": "scalar_spectral",

该配置文件的其余部分定义的是 C++ 类型的可用变体，你可以安全地忽略它们。

简单的讲：如果你计划在 Python 中使用 Mitsuba，我们建议在 CPU 渲染时添加
``packet_rgb`` 或 ``packet_spectral`` 其中之一，相应的对于 GPU 渲染添加
``gpu_autodiff_rgb`` 或 ``gpu_autodiff_spectral`` 之一。

一旦你完成了 :monosp:`mitsuba.conf` 中变体的设置，请继续下一节 :ref:`编译整个系统 <sec-compiling>`。
