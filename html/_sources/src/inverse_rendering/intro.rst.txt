.. _sec-inverse-rendering:

引言
============

Mitsuba 2 可以使用一种称为 *可微渲染* 的技术来解决涉及光线的逆向问题。该技术将渲染算法解释为一个函数 :math:`f(\mathbf{x})` 
这个函数将输入 :math:`\mathbf{x}` （场景描述）转化为输出 :math:`\mathbf{y}` （渲染图）。那么这个函数 :math:`f` 在数学上的
微分就很显然了 :math:`\frac{\mathrm{d}\mathbf{y}}{\mathrm{d}\mathbf{x}}` ，这个微分提供了如何通过改变输入 :math:`\mathbf{x}`（场景描述）
来改变输出 :math:`\mathbf{y}` （渲染图）的一阶近似。结合量化场景参数适用性的可微 *目标函数* :math:`g(\mathbf{y})` ，随后可以
使用梯度优化算法诸如随机梯度下降或者 Adam 来查找一系列场景参数 :math:`\mathbf{x}_0`，:math:`\mathbf{x}_1`， :math:`\mathbf{x}_2` 等，从而
逐步改善目标函数。如下面图片所示：

.. image:: ../../../resources/data/docs/images/autodiff/autodiff.jpg
    :width: 100%
    :align: center

Mitsuba 2 中的可微渲染是基于 ``gpu_autodiff`` 变体模式下的后端。需要注意的是，由于性能原因目前不支持在
CPU 上进行可微渲染，但是我们有一些想法可以加快渲染速度并且计划在将来合并进来。

Differentiable calculations using Enoki
---------------------------------------

Mitsuba 能够自动微分整个渲染算法的能力来自于 Enoki 库提供的可微 CUDA 数组类型。两者在 Enoki 的文档的 `GPU arrays <https://enoki.readthedocs.io/en/master/gpu.html>`_ 章节
中都有详细解释，描述了基本的 *just-in-time* （JIT）编译器，该编译器将加法和乘法等简单操作融合到可以在具有 CUDA 功能的 GPU 上执行的较大计算内核中。上面链接的文档
还讨论了和 PyTorch 以及 TensorFlow 这种表面上相似的框架都有哪些区别。`Automatic differentiation <https://enoki.readthedocs.io/en/master/autodiff.html>`_ 
这一章节描述了 Enoki 如何记录和简化计算图并使用它们以正向或反向模式传播导数。我们建议你可以熟读 Mitsuba 和 Enoki 的文档。

当变体模式是 ``gpu_autodiff_*`` 时，会自动导入 Enoki 的微分类型。它们在 C++ 和 Python 中都有使用，因此在每种语言下
都可以实现大量微分计算。以下程序展示了使用 Python 编写一个简单的传播计算，来微分函数 :math:`f(\mathbf{x})=x_0^2 + x_1^2 + x_2^2` 

.. code-block:: python

    import mitsuba
    mitsuba.set_variant('gpu_autodiff_rgb')

    # The C++ type associated with 'Float' is enoki::DiffArray<enoki::CUDAArray<float>>
    from mitsuba.core import Float
    import enoki as ek

    # Initialize a dynamic CUDA floating point array with some values
    x = Float([1, 2, 3])

    # Tell Enoki that we'll later be interested in gradients of
    # an as-of-yet unspecified objective function with respect to 'x'
    ek.set_requires_gradient(x)

    # Example objective function: sum of squares
    y = ek.hsum(x * x)

    # Now back-propagate gradient wrt. 'y' to input variables (i.e. 'x')
    ek.backward(y)

    # Prints: [2, 4, 6]
    print(ek.gradient(x))

