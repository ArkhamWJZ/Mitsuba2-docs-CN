.. _sec-shapes:

形状插件 Shape
======

本小节呈现的是 Shape 类插件的概览，这些插件随着渲染器发布。

在 Mitsuba 2中，shape 定义了不同类型材质间转换的曲面。例如，shape 可以描述空气和固体介质（如一块岩石）之间的边界。
或者，一个 shape 可以标记一块非固态空间区域的开始，这个空间区域中是粒子介质，如烟雾或蒸汽等。最后，一个 shape 可以
用于创建自发光的对象。

Shape 通常情况下和名为 *BSDF* 的表面散射模型一起进行声明（详情请参阅 :ref:`respective section <sec-bsdfs>` 章节）。
BSDF 描述了曲面与光的交互。在 XML 场景描述语言中可以如下编写：

.. code-block:: xml

    <scene version=2.0.0>
        <shape type=".. shape type ..">
            .. shape parameters ..

            <bsdf type=".. BSDF type ..">
                .. bsdf parameters ..
            </bsdf>

            <!-- Alternatively: reference a named BSDF that
                 has been declared previously

                 <ref id="my_bsdf"/>
            -->
        </shape>
    </scene>

接下来的小节更加详细的讨论了可用的 shape 类型。