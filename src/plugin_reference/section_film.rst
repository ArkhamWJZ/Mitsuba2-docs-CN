.. _sec-films:

胶片
=====

film 定义了如何存储所进行的测量并将其转换为最终的输出文件，该文件将在渲染过程结束时写入磁盘。

在 XML 场景描述语言中，一个常规的 film 配置应该看上去如下所示：

.. code-block:: xml

    <scene version=2.0.0>
        <!-- .. scene contents -->

        <sensor type=".. sensor type ..">
            <!-- .. sensor parameters .. -->

            <!-- Write to a high dynamic range EXR image -->
            <film type="hdrfilm">
                <!-- Specify the desired resolution (e.g. full HD) -->
                <integer name="width" value="1920"/>
                <integer name="height" value="1080"/>

                <!-- Use a Gaussian reconstruction filter. For details
                     on these, refor to the next subsection -->
                <rfilter type="gaussian"/>
            </film>
        </sensor>
    </scene>

``<film>`` 插件应该嵌套在 ``<sensor>`` 声明中实例化。请注意输出文件名是不指定的，它从场景文件名自动推断出来，
同时也可以在命令行渲染时通过将配置参数 ``-o`` 传递给 ``mitsuba`` 可执行文件来手动覆盖。
