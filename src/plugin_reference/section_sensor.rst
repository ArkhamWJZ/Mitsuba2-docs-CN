.. _sec-sensors:

传感器
=======

在 Mitsuba 2 中  *sensors* 以及 *film* 都是以一些可用格式来纪录辐射度的表达方式。

在 XML 场景描述语言中，一个 sensor 可以如下进行定义：

.. code-block:: xml

    <scene version=2.0.0>
        <!-- .. scene contents .. -->

        <sensor type=".. sensor type ..">
            <!-- .. sensor parameters .. -->

            <sampler type=".. sampler type ..">
                <!-- .. sampler parameters .. -->
            </samplers>

            <film type=".. film type ..">
                <!-- .. film parameters .. -->
            </film>
        </sensor>
    </scene>

换句话说，``sensor`` 是  ``<scene>`` 的子类（对于场景文件中的特定位置不起作用）。嵌套在 sensor 声明中的
是一个 sampler 实例（详见 :ref:`Samplers <sec-samplers>`）和一个 film 实例（详见 :ref:`Films <sec-films>`）。

在 Mitsuba 2 中 sensor 遵循的是 *右手系* 。对他们进行任意旋转或平移操作都无法动摇这一属性。默认情况下，
它们位于坐标系原点并朝着渲染图规定的方向，其中 :math:`+X` 指向左方，:math:`+Y` 指向上方，:math:`+Z` 则沿着观察方向。
当然左手系也是支持的。改变坐标系性质，只需要反转任意轴即可，例如，通过对 sensor 的 :monosp:`to_world` 参数执行一个缩放变换 ``<scale x="-1"/>`` 。