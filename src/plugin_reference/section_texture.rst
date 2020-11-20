.. _sec-textures:

纹理
========


接下来的小节描述的是纹理数据资源。在 Mitsuba 2 中纹理作为对象存在，纹理对象可以作为附加到某些表面散射模型的参数，以
引入空间变化。在本文档中，以下列出了支持的 :paramtype:`texture` 类型。有关 BSDF 的更多示例，请参阅最后几节。

Texture 有一个可选参数 ``<transform>`` 叫做 :paramtype:`to_uv` ，该参数可以用作平移、缩放、旋转以查找纹理。

一个 XML 的例子如下所示：

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Create a BSDF that supports textured parameters -->
        <bsdf type=".. BSDF type .." id="my_textured_material">
            <texture type=".. texture type .." name=".. parameter name ..">
                <!-- .. Texture parameters go here .. -->

                <transform name="to_uv">
                    <!-- Scale texture by factor of 2 -->
                    <scale x="2" y="2"/>
                    <!-- Offset texture by [0.5, 1.0] -->
                    <translate x="0.5" y="1.0"/>
                </transform>
            </texture>

            <!-- .. Non-spatially varying BSDF parameters ..-->
        </bsdf>
    </scene>

与 BSDF 类似，命名了的纹理可以在场景的顶层定义并在之后进行引用。如果相同的纹理需要多次加载，这将会特别有用。

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Create a named texture at the top level -->
        <texture type=".. texture type .." id="my_named_texture">
            <!-- .. Texture parameters go here .. -->

            <transform name="to_uv">
                <!-- .. Transform parameters .. -->
            </transform>
        </texture>

        <!-- Create a BSDF that supports textured parameters -->
        <bsdf type=".. BSDF type ..">
            <!-- Example of referencing a named texture -->
            <ref id="my_named_texture" name=".. parameter name .."/>

            <!-- .. Non-spatially varying BSDF parameters ..-->
        </bsdf>
    </scene>



