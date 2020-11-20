.. _sec-bsdfs:

BSDFs
=====

    .. image:: ../../resources/data/docs/images/bsdf/bsdf_overview.jpg
        :width: 100%
        :align: center

    上面的图解概括了 Mitsuba2 中最重要的表面散射模型。箭头表示了光线与各自模型表面交互的可能出射结果。

表面散射模型描述了光线与场景中的各种介质表面交互的行为。该模型很方便地总结了材料内部发生的微观散射过程，并使它
看起来非常自然。这种表示方式是 Mitsuba 2 材质系统的核心组件之一，渲染器关注的另一部分是光线在表面介质间的交互
时发生了什么。关于这方面更多的信息，请参阅 participating media 这一章节。该章节介绍的是所有支持的表面散射模型
以及它们对应的参数。

为了达到真是感渲染的效果，Mitsuba 2 提供了一个通用介质的表面散射模型库，包括 玻璃材质、金属材质以及塑料材质。
一些模型插件也可以作为编辑器应用在一个或多个散射模型的文件的顶部。

尽管在文档中和场景描述语言中，BSDF 这个单词和 *表面散射模型* 是等价的，但这实际上是 *双向散射分布* 的缩写，在
技术上更精准一点的话。

在 Mitsuba 2 中，BSDF 与描述场景中可见表面的 *shapes* 相关联。在场景描述语言中，关联的过程可以通过将 BSDF 嵌
套到 shape 中来进行，也可以对它们进行命名，随后再根据它们的名称进行引用。下面的代码片段展示了这两种用法的示例：

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Creating a named BSDF for later use -->
        <bsdf type=".. BSDF type .." id="my_named_material">
            <!-- BSDF parameters go here -->
        </bsdf>

        <shape type="sphere">
            <!-- Example of referencing a named material -->
            <ref id="my_named_material"/>
        </shape>

        <shape type="sphere">
            <!-- Example of instantiating an unnamed material -->
            <bsdf type=".. BSDF type ..">
                <!-- BSDF parameters go here -->
            </bsdf>
        </shape>
    </scene>

当在多个地方使用已命名的 BSDF 时，通常使用引用的方法会更经济，因为这样可以减少内存的使用。

.. _bsdf-correctness:

Correctness considerations
--------------------------

在基于物理渲染场景的一个至关重要的考量是，不能违反物理规则，并且每个安排都要是有意义的。
例如，想象一个建筑场景的内部，这个场景视觉效果很棒，除了有一张白桌子看上去有点暗。仔细一点观察发现，
它使用的 Lambertian 材质的漫发射系数是 0.9。

在许多渲染系统中，针对这种情况将反射率调整到 1.0 以上是可行的。但是在 Mitsuba 中，即使是一个很小的
表面反射出比自身接收到的还要多的光强都是不允许的，这可能会破坏渲染算法的可用性导致预计之外的结果。
事实上，这种情况的正确解决方案是调整光源的设置，使桌子能够接受到更多的光照明，除此之外还要 *减少材质的反射率* 
毕竟在真实世界中找到一张能反射 90% 入射光的桌子还是挺困难的一件事。

关于材料描述要有意义这一观点必要性的另一个例子是下图所示的玻璃模型。在这里，需要仔细思考不同渲染介质之间折射率变化的边界。
如果做的不正确，一束光可能潜在通过一系列的不可能的折射率变化（例如 1.00 到 1.33 其次是 1.50 到 1.33），这种情况下的输出是未定义的，
从而相当程度上在场景中可能存在一些错误，导致渲染结果远远不像是玻璃。

.. figtable::
    :label: fig-glass-explanation
    :caption: Mitsuba 中的一些散射模型需要知道介质表面向外部和面向内部的折射率参数。因此，根据折射率的变化，将网格分解为有意义的独立表面
              是很重要的。这里的例子展示了一个装满水的玻璃杯的分解过程。
    :alt: Glass interfaces explanation

    .. figure:: ../../resources/data/docs/images/bsdf/glass_explanation.svg
        :alt: Glass interfaces explanation
        :width: 95%
        :align: center


