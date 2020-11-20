.. _sec-emitters:

光源
========

    .. image:: ../../resources/data/docs/images/emitter/emitter_overview.jpg
        :width: 70%
        :align: center

    Schematic overview of the emitters in Mitsuba 2. The arrows indicate
    the directional distribution of light.

Mitsuba 2 支持大量不同类型的光源，这些光源大致上可以分成两类：坐落在场景内的光源，围绕在场景外模拟在远处的光源。

一般来说，光源是 ``<scene>`` 这一元素的子类；例如，下面的代码片段实例化了一个点光源发射器：

.. code-block:: xml

    <scene version="2.0.0">
        <emitter type="point">
            <spectrum name="intensity" value="1"/>
            <point name="position" x="0" y="0" z="-2"/>
        </emitter>

        <shape type="sphere"/>
    </scene>

有一个例外是面光源，面光源是将一个几何物体变为了光源。所以它们被指定为 ``<shape>`` 这一元素的子类：

.. code-block:: xml

    <scene version="2.0.0">
        <shape type="sphere">
            <emitter type="area">
                <spectrum name="radiance" value="1"/>
            </emitter>
        </shape>
    </scene>
