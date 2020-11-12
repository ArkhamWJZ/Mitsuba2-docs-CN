Testing
=======

运行测试套件，只需调用 ``pytest`` ：

.. code-block:: bash

    pytest

    # or to run a single test file
    pytest src/bsdfs/tests/test_diffuse.py

需要注意的是，运行完整的测试套件可能需要很长的时间，特别是在内核比较少的机器上。可以通过以下方式跳过执行较慢的测试：

.. code-block:: bash

    pytest -m 'not slow'

构建系统还开放了一个 ``pytest`` ，以执行 ``setpath`` 和并行执行测试。

.. code-block:: bash

    ninja pytest


Chi^2 tests
-----------

``mitsuba.python.chi2`` 模块实现了 Pearson 卡方检验，用于检测一个分布与已知参考分布的拟合程度。

该模块十分具体的实现了 2D（或更低维度）上的蒙特卡洛采样策略与一个分布做比较，该分布的概率密度函数是从
分布参数域上的网格通过数值积分获得的。

它在整个测试套件中被广泛使用，以验证 ``BSDFs`` 、 ``Emitters`` 以及其他样例代码的实现有效。

可以通过以下方式测试你自己编写的样例：

.. code-block:: python

    import mitsuba
    mitsuba.set_variant('packet_rgb')

    from mitsuba.python.chi2 import ChiSquareTest, SphericalDomain

    # some sampling code
    def my_sample(sample):
        return mitsuba.core.warp.square_to_cosine_hemisphere(sample)

    # the corresponding probability density function
    def my_pdf(p):
        return mitsuba.core.warp.square_to_cosine_hemisphere_pdf(p)

    chi2 = ChiSquareTest(
        domain=SphericalDomain(),
        sample_func=my_sample,
        pdf_func=my_pdf,
        sample_dim=2
    )

    assert chi2.run()

在失败的情况下，目标密度和直方图被写入到 ``chi2_data.py`` ，只需运行该文件即可绘制数据：

.. code-block:: bash

    python chi2_data.py


``mitsuba.python.chi2`` 模块还提供了一组 ``Adapter`` 函数，可用于包装不同的插件（例如，``BSDF``， ``Emitter`` 等），可以通过以下方式
测试它们：

.. code-block:: python

    import mitsuba
    import enoki as ek

    mitsuba.set_variant('packet_rgb')

    from mitsuba.python.chi2 import BSDFAdapter, ChiSquareTest, SphericalDomain

    xml = """<float name="alpha" value="0.5"/>
             <boolean name="sample_visible" value="false"/>
             <string name="distribution" value="ggx"/>
          """
    wi = ek.normalize([0.2, -0.6, -0.5])
    sample_func, pdf_func = BSDFAdapter("roughdielectric", xml, wi=wi)

    chi2 = ChiSquareTest(
        domain=SphericalDomain(),
        sample_func=sample_func,
        pdf_func=pdf_func,
        sample_dim=3
    )

    assert chi2.run()

    # Forces the chi2 test to dump the plotting script (optional)
    chi2._dump_tables()


下面是由上例中的 ``chi2_data.py`` 脚本生成的图像：

.. image:: ../../images/chi2_example.png
    :align: center
    :width: 100%

左边的图展示了通过数值积分方式解析的来自 ``roughdielectric`` BSDF 材质内部的入射光线的 ``pdf()``。
可以看到大部分的能量离开了表面（图的上半部分），而一部分能量被反射回来了（图的下半部分）。

中间的图展示了相同的密度函数，但这一次计算的是 ``roughdielectric`` BSDF 中 ``sample()`` 方
法的采样方向的直方图。

右图显示了两个密度函数之间的差异。BSDF 的采样程序是随机的，由于直方图仍有噪声，预计能观察到正负值混合的现象。
``ChiSquareTest`` 的主要作用在于决定观察到的偏差是否可以归于随机噪声的范围中，或者发现是否存在导致测试失败
的系统偏差。

更多详细信息，请参阅 :py:class:`mitsuba.python.chi2.ChiSquareTest`。


Rendering test suite and Z-test
---------------------------------------

在 *单元测试* 之上，我们的框架实现了一种自动渲染一组测试场景的机制，并通过应用 `Z-test <https://en.wikipedia.org/wiki/Z-test>`_ 来
比较渲染的结果图和其他相关图像。

这些测试对于揭示渲染器各个独立组件间的交互 BUG 非常有用。

开启所有变体模式渲染测试场景时，请确保用例对于 ``scalar_rgb`` 和 ``gpu_rgb`` 渲染模式的匹配。

如需仅运行渲染测试套件，可以使用以下命令行指令：

.. code-block:: bash

    pytest src/librender/tests/test_renders.py

把场景添加到 ``resources/data/tests/scenes/`` 文件夹中，就将其添加到渲染测试套件中了，非常简便。
随后，可以使用以下命令生成想要的相关图：

.. code-block:: bash

    python src/librender/tests/test_renders.py