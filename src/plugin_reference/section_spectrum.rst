.. _sec-spectra:

光谱
=======

本节介绍的是 Mitsuba 2 在光谱反射或发光背后的插件。在实现的层面上，它们的行为与之前 :ref:`texture plugins <sec-textures>` 
中的描述非常相似（但是要缺少空间变化属性），因此，可以类似地作为 BSDF 或 emitter 的参数使用：

.. code-block:: xml

    <scene version=2.0.0>
        <bsdf type=".. BSDF type ..">
            <!-- Explicitly add a uniform spectrum plugin -->
            <spectrum type=".. spectrum type .." name=".. parameter name ..">
                <!-- Spectrum parameters go here -->
            </spectrum>
        </bsdf>
    </scene>

但是在实际工作中，不鼓励以这种显式方式实例化插件，XML 场景描述解析器会直接解析许多常用的插件（较短的）
如 ``<spectrum>`` 和 ``<rgb>`` 标签。请参阅 :ref:`scene file format <sec-file-format>` 相关章节
以获得更多信息。

接下来的两个表格总结了底层插件在每种情况下的实例化，考虑到了反射和发射属性以及不同颜色模式之间的差异。
下面对于了每个插件都有简要的介绍。

.. figtable::
    :label: spectrum-reflectance-table-list
    :caption: Spectra used for reflectance (within BSDFs)
    :alt: Spectrum reflectance table

    .. list-table::
        :widths: 35 25 25 25
        :header-rows: 1

        * - XML description
          - Monochrome mode
          - RGB mode
          - Spectral mode
        * - ``<spectrum name=".." value="0.5"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`uniform <spectrum-uniform>`
        * - ``<spectrum name=".." value="400:0.1, 700:0.2"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<spectrum name=".." filename=".."/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<rgb name=".." value="0.5, 0.2, 0.5"/>``
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`srgb <spectrum-srgb>`

.. figtable::
    :label: spectrum-emission-table-list
    :caption: Spectra used for emission (within emitters)
    :alt: Spectrum emission table

    .. list-table::
        :widths: 35 25 25 25
        :header-rows: 1

        * - XML description
          - Monochrome mode
          - RGB mode
          - Spectral mode
        * - ``<spectrum name=".." value="0.5"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`d65 <spectrum-d65>`
        * - ``<spectrum name=".." value="400:0.1, 700:0.2"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<spectrum name=".." filename=".."/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<rgb name=".." value="0.5, 0.2, 0.5"/>``
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
