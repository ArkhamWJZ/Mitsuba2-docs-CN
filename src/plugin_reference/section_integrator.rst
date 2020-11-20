.. _sec-integrators:

积分器
===========

在 Mitsuba 2中，与众不同的渲染技术都和 *integrator* 的使用相关，积分器是在高维度计算积分的。
每个积分器表示的都是解决光线传播方程的一种特定方法，通常在某些情况下有利，但同时又受其自身固有的局限性的影响。
因此，根据指定精度的要求以及待渲染场景的特性认真选择合适的积分器求解非常重要。

在 XML 场景描述语言中，一个简单的积分器通常应在场景顶部声明来实例化，例如：

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Instantiate a unidirectional path tracer,
             which renders paths up to a depth of 5 -->
        <integrator type="path">
            <integer name="max_depth" value="5"/>
        </integrator>

        <!-- Some geometry to be rendered -->
        <shape type="sphere">
            <bsdf type="diffuse"/>
        </shape>
    </scene>

这一小节给出的是积分器参数可用选项的概览。

几乎所有的积分器都使用了 *path depth* 的概念。在这里，一条路径是指从光源开始到相机结束的一系列散射行为。
在预览渲染场景时限制路径深度是非常有用的，因为这减少了每个像素需要的计算量。除此之外，这类渲染通常收敛速度更快，
因此每个像素需要的样本会更少。如果随后想要参考渲染质量，此时应该允许路径深度无限制即可。

下面 Cornell box 的渲染图展示了最大路径深度的视觉效果。随着允许光路长度的增加，更多的散射和着色表面进行交互从而
颜色的饱和度也在增加，同时渲染用时也会相应增加。

.. subfigstart::
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_1.jpg
   :caption: max. depth = 1
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_2.jpg
   :caption: max. depth = 2
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_3.jpg
   :caption: max. depth = 3
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_inf.jpg
   :caption: max. depth = :math:`\infty`
.. subfigend::
   :width: 0.23
   :label: fig-integrators-depth

Mitsuba 从 1 开始计算路径深度，这与可见光源相对应（即，光线从光源开始到相机结束，其间没有与任何物体交互）。
深度为 2 的路径（即我们所熟知的 “直接照明”）包含了如下图一样一个简单的散射事件：

.. image:: ../../resources/data/docs/images/integrator/path_explanation.jpg
    :width: 80%
    :align: center