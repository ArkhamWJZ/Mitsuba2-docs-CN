.. _sec-differentiable-rendering:

可微渲染
========================

现在我将逐步构建一个简单的示例程序，该示例程序是一个展现光线传播模拟中的微分与优化的例子，
模拟过程成中使用的场景是著名的 Cornell Box 可以 `这里下载 <http://mitsuba-renderer.org/scenes/cbox.zip>`_。

请按照下面的教程对 ``cbox.xml`` 文件做出三处改动：

1. ``ldsampler`` 必须被替换为 ``independent`` （``ldsampler`` 尚未移植到 Mitsuba 2）。 
   请注意，此处指定的样本计数将在接下来的优化脚本中被覆盖。

2. 头部的积分器应该如下进行定义：

   .. code-block:: xml

       <integrator type="path">
           <integer name="max_depth" value="3"/>
       </integrator>

3. film 应该指定一个 box 重建滤波而不是 ``gaussian``。

   .. code-block:: xml

       <rfilter type="box"/>

在可微渲染的情况中，我们通常会对尽快渲染许多低质量的图像感兴趣，出于这种考量上述更改针对的降低了渲染图像质量。

Enumerating scene parameters
----------------------------

由于可微渲染需要一个初始估计值（即初始场景配置），因此大多数应用程序先加载 Cornell box 场景随后枚举并
选择待微分的场景参数：

.. code-block:: python

    import enoki as ek
    import mitsuba
    mitsuba.set_variant('gpu_autodiff_rgb')

    from mitsuba.core import Thread
    from mitsuba.core.xml import load_file
    from mitsuba.python.util import traverse

    # Load the Cornell Box
    Thread.thread().file_resolver().append('cbox')
    scene = load_file('cbox/cbox.xml')

    # Find differentiable scene parameters
    params = traverse(scene)
    print(params)

倒数第二行调用的函数 :py:func:`~mitsuba.python.util.traverse()` 就完成了这一任务。
它会返回一个类似于字典的 :py:class:`mitsuba.python.util.ParameterMap` 实例，该实例
公开出来可修改和可微分的场景参数。将其传递给 ``print()`` 打印出来，会输出以下摘要信息：

.. code-block:: text

    ParameterMap[
        ...
      * box.reflectance.value,
      * white.reflectance.value,
      * red.reflectance.value,
      * green.reflectance.value,
      * light.reflectance.value,
        ...
      * OBJMesh.emitter.radiance.value,
        OBJMesh.vertex_count,
        OBJMesh.face_count,
        OBJMesh.faces,
      * OBJMesh.vertex_positions,
      * OBJMesh.vertex_normals,
      * OBJMesh.vertex_texcoords,
        OBJMesh_1.vertex_count,
        OBJMesh_1.face_count,

        OBJMesh_1.faces,
      * OBJMesh_1.vertex_positions,
      * OBJMesh_1.vertex_normals,
      * OBJMesh_1.vertex_texcoords,
        ...
    ]

Cornell box 场景包含了一个光源和多个多重网格和 BRDF 组成。每个组件都为上述列表提供了某些条目---例如，
网格除了指定它们所属的面元和顶点序号外还有 per-face (``.faces``) 以及 per-vertex data (``.vertex_positions``，
``.vertex_normals``， ``.vertex_texcoords``)。每种 BRDF 都添加了 ``.reflectance.value`` 这一条目。
不是所有的函数都是可微分的---例如存储的一些整数值。左侧带有星号（``*``）的参数表示它们是可微的。

参数的命名是使用基于场景图中的位置和底层实现的类名的简单命名方案。
只要通过 XML 场景描述中的 ``id="..."`` 属性为对象分配了唯一标识符，则该标识符具有优先权。 
例如，``red.reflectance.value`` 条目对应于下面所示的原始场景描述中声明的反射率：

.. code-block:: xml

    <bsdf type="diffuse" id="red">
        <spectrum name="reflectance" value="400:0.04, 404:0.046, ..., 696:0.635, 700:0.642"/>
    </bsdf>

我们还可以参阅 :py:class:`~mitsuba.python.util.ParameterMap` 以查看实际参数值：

.. code-block:: python

    print(params['red.reflectance.value'])

    # Prints:
    # [[0.569717, 0.0430141, 0.0443234]]

这里可以看到，由于我们是使用了 ``gpu_autodiff_rgb`` 变体来运行此示例，Mitsuba 将上述的 XML 片段的原始
光谱曲线转换为 RGB 值。

在大多数的情况下，我们只对可微参数映射的一小部分（这一小部分通常也很大）感兴趣。通过使用 :py:meth:`ParameterMap.keep() <mitsuba.python.util.ParameterMap.keep()>` 
方法，丢弃除了指定列表外的所有条目。

.. code-block:: python

    params.keep(['red.reflectance.value'])
    print(params)

    # Prints:
    # ParameterMap[
    #   * red.reflectance.value
    # ]

Let's also make a backup copy of this color value for later use.

.. code-block:: python

    from mitsuba.core import Color3f
    param_ref = Color3f(params['red.reflectance.value'])


Problem statement
-----------------

与使用 Python API 的 :ref:`previous example <sec-rendering-scene>` 相比，可微渲染路径涉及到了另一个
渲染函数 :py:func:`mitsuba.python.autodiff.render()` 对于这用例会更为优化。它直接返回包含生成图像的 GPU 序列。
函数 :py:func:`~mitsuba.python.autodiff.write_bitmap()` 将输出重新整型为正确大小的图像，并导出为任何支持的图像
格式（OpenEXR，PNG，JPG，RGBE，PFM）同时在 8-bit 输出格式下自动执行格式转换和伽马矫正。

使用此功能，我们现在需要使用每像素8采样（``spp``）来生成参考图。

.. code-block:: python

    # Render a reference image (no derivatives used yet)
    from mitsuba.python.autodiff import render, write_bitmap
    image_ref = render(scene, spp=8)
    crop_size = scene.sensors()[0].film().crop_size()
    write_bitmap('out_ref.png', image_ref, crop_size)


我们的第一个实验非常简单：我们将会改变红墙上的颜色，随后尝试使用微分以及上面生成的参考图来恢复一开始的颜色。

为此，让我们首先改变当前颜色值：参数映射允许我们不用重新加载场景就可以做到这种改变。最后调用的 :py:meth:`ParameterMap.update() <mitsuba.python.util.ParameterMap.update()>` 方法
强制通知场景对象做出刷新它们内部状态的修改。

.. code-block:: python

    # Change the left wall into a bright white surface
    params['red.reflectance.value'] = [.9, .9, .9]
    params.update()

Gradient-based optimization
---------------------------

Mitsuba 可以使用在 Enoki 中实现的优化算法在 *standalone mode* 下优化场景参数，
也可以将其作为一个较大的 PyTorch 计算图中的可微节点。PyTorch 和 Enoki 之间的通信
通常会导致某些开销，因此我们通常会建议你能够坚持使用 standalone mode 除非你的计算会
涉及到 PyTorch 能够提供的具有明显优势的模块（例如，神经网络构建模块像是全连接层或卷积）。
本节的其余部分将继续讨论 standalone mode，:ref:`PyTorch integration <sec-pytorch>`
的相关章节会展示如何在 PyTorch 中适配示例代码。

Mitsuba 附带的标准优化器包括有无动量的 *随机梯度下降* （:py:class:`~mitsuba.python.autodiff.SGD`），
就像 :py:class:`~mitsuba.python.autodiff.Adam` :cite:`kingma2014adam` 一样我们将在稍后进行实例化，
并按照 0.2 的学习率通过优化减少 :py:class:`~mitsuba.python.util.ParameterMap` ``params`` 。
优化器类自动请求所选参数的梯度信息，并在每一步之后更新它们的值，因此不需要直接修改 ``params`` 或调用  ``ek.set_requires_gradient`` 
，正如引言中所述。

.. code-block:: python

    # Construct an Adam optimizer that will adjust the parameters 'params'
    from mitsuba.python.autodiff import Adam
    opt = Adam(params, lr=.2)

其余命令都是执行 100 次可微渲染迭代的一部分。

.. code-block:: python

    for it in range(100):
        # Perform a differentiable rendering of the scene
        image = render(scene, optimizer=opt, unbiased=True, spp=1)

        write_bitmap('out_%03i.png' % it, image, crop_size)


.. note::

    **关于梯度的偏置**: 在对渲染算法进行简单的微分时，有一个潜在的问题就是使用相同的蒙特卡洛样本集
    来生成原始输出（即图像）和导数输出。当渲染算法和目标都可微时，我们最终得到的输出期望是不满足等式  :math:`\mathbb{E}[X Y]=\mathbb{E}[X]\,\mathbb{E}[Y]`
     这是因为 :math:`X` 和 :math:`Y` 之间的相关性，这是重用样本集的结果。

    :py:func:`~mitsuba.python.autodiff.render()` 函数的 ``unbiased=True`` 参数将函数切换到一种特殊的无偏模式，
    使原始组件和导数组件不相关，这可以归结为进行来两次渲染图像，自然要付出一定的性能代价 :math:`(\sim 1.6 \times\!)` 。
    通常情况下有偏的梯度已经很好了，在这种情况下应该将参数设置为 ``unbiased=False`` 。

.. note::

    **关于每样本采样数**: 一个非常极端的例子是在上述微分渲染的迭代中使用很少的采样数（``spp=1``），
    这会导致渲染结果和梯度都是充满噪声的。或者，我们可以对于更大的梯度步数相应采用更多的采样数（即，在优化中使用更高的 ``lr=..`` 参数）。
    一般情况下我们发现低采样数表现的很好，因为它大大减少了内存使用并且更适应参数值的变化。

在 for 循环内，我们现在可以评估合适的目标函数了，按照梯度步数传播目标函数的导数。

.. code-block:: python

    # for loop body (continued)
        # Objective: MSE between 'image' and 'image_ref'
        ob_val = ek.hsum(ek.sqr(image - image_ref)) / len(image)

        # Back-propagate errors to input parameters
        ek.backward(ob_val)

        # Optimizer: take a gradient step
        opt.step()

我还可以把每次迭代中的误差画成图表。需要注意的是，可视化目标对象的 ``ob_val`` 是几乎没有任何意义的，
因为 ``image`` 和 ``image_ref`` 之间的差异主要由蒙特卡洛噪声控制与优化的参数是没有关系的。
由于我们知道场景中的 "true" 目标参数（之前储存在了 ``param_ref`` 中），因此我们可以验证迭代的敛散性：

.. code-block:: python

        err_ref = ek.hsum(ek.sqr(param_ref - params['red.reflectance.value']))
        print('Iteration %03i: error=%g' % (it, err_ref[0]))

接下来的视频展示了前 100 次迭代过程的收敛记录。随着梯度步数的增加左边红色墙的颜色被迅速恢复。

.. raw:: html

    <center>
        <video controls loop autoplay muted
        src="https://rgl.s3.eu-central-1.amazonaws.com/media/uploads/wjakob/2020/03/02/convergence.mp4"></video>
    </center>

注意振荡行为，在下面所示的收敛图中可以看到。这表明学习率的值设置过大了。

.. image:: ../../../resources/data/docs/images/autodiff/convergence.png
    :width: 50%
    :align: center

.. note::

    **性能相关**: 此优化应该很快完成。在 NVIDIA Titan RTX 上，当 ``write_bitmap`` 例程被注释掉时每次迭代大概需要
    50 ms，当设置 ``unbiased=False`` 时每次迭代大约需要 27 ms。

    我们注意到了其他应用程序也会同时使用 GPU （比如 Chrome 或 Firefox）这种看上去无害的使用（比如在打开一个 YouTube 标签等）
    会降低可微渲染的性能，有十倍之多。如果发现你的数值与上述数值有很大差异，请尝试关闭所有其他软件。

.. note::

    上述教程的完整 Python 脚本可以在 :file:`docs/examples/10_diff_render/invert_cbox.py` 文件中找到。


Forward-mode differentiation
----------------------------

之前的示例展现的是 *反向模式微分* （别名，反向传播），将输出图像的所需的细微变化转化为了场景参数的细微变化。
Mitsuba 和 Enoki 也可以按照另一个方向传播导数，例如，从输入参数到输出的渲染图。这种技术被称为 *正向模式微分* ，
该技术对参数优化是没有用的，因为每个参数必须使用独立的渲染过程进行处理。即便如此，这种模式也是非常有参考价值的，因为它可视化

正向模式的微分渲染类似与反向模式，由声明参数并将其标记为可微的（我们将手动进行，而不是使用 :py:class:`mitsuba.python.autodiff.Optimizer`）。

.. code-block:: python

    # Keep track of derivatives with respect to one parameter
    param_0 = params['red.reflectance.value']
    ek.set_requires_gradient(param_0)

    # Differentiable simulation
    image = render(scene, spp=32)

一但完成记录计算后，我们可以对之前标记的参数进行一个扰动，并通过图进行正向传播。

.. code-block:: python

    # Assign the gradient [1, 1, 1] to the 'red.reflectance.value' input
    ek.set_gradient(param_0, [1, 1, 1], backward=False)

    from mitsuba.core import Float

    # Forward-propagate previously assigned gradients. The set_gradient call
    # above already indicated which derivatives to propagate, hence we use the
    # static FloatD.forward() function. See the Enoki documentation for further
    # explanations of the various ways in which derivatives can be propagated.
    Float.forward()

参阅 Enoki 文档中 `automatic differentiation
<https://enoki.readthedocs.io/en/master/autodiff.html>`_ 章节以获得关于这一步的更多技术细节。
最后我们将生成的可视化梯度写入磁盘中。

.. code-block:: python

    # The gradients have been propagated to the output image
    image_grad = ek.gradient(image)

    # .. write them to a PNG file
    crop_size = scene.sensors()[0].film().crop_size()
    write_bitmap('out.png', image_grad, crop_size)

这将产生类似于下图的结果：

.. image:: ../../../resources/data/docs/images/autodiff/forward.jpg
    :width: 50%
    :align: center

观察在全局光照中，改变红墙颜色是如何对整幅图像产生影响的。由于我们在反射率方面做出的变更，
因此红色消失了。

.. note::

    上述教程的完整 Python 脚本可以在 :file:`docs/examples/10_diff_render/forward_diff.py` 文件中找到。
