.. _sec-pytorch:

与 PyTorch 的集成
========================

我们简要地展示了将可微渲染与使用 PyTorch 的优化相结合时，如何使上一节 :ref:`differentiable rendering <sec-differentiable-rendering>` 
中的示例变得可行。结合这些框架的能力可以将 Mitsuba 2 夹在神经层之间，并区分端到端的组合。

请注意，与仅使用 Enoki 实现的优化相比，Enoki 和 PyTorch 之间的通信和同步以及遍历两个单独的计算图数据结构
的复杂性会导致 10-20％ 的开销。我们通常建议坚持使用 Enoki，除非问题需要神经网络构建块(例如全连接层或卷积)，
此时使用 PyTorch 会有明显的优势。

和之前一样，我们指定相关的变体，加载场景并保留相关的可微参数。

.. code-block:: python

    import mitsuba
    mitsuba.set_variant('gpu_autodiff_rgb')

    import enoki as ek
    from mitsuba.core import Thread, Vector3f
    from mitsuba.core.xml import load_file
    from mitsuba.python.util import traverse
    from mitsuba.python.autodiff import render_torch, write_bitmap
    import torch
    import time

    Thread.thread().file_resolver().append('cbox')
    scene = load_file('cbox/cbox.xml')

    # Find differentiable scene parameters
    params = traverse(scene)

    # Discard all parameters except for one we want to differentiate
    params.keep(['red.reflectance.value'])

``.torch()`` 方法可以用于将任何 Enoki CUDA 类型转化为相应的 PyTorch 张量。

.. code-block:: python

    # Print the current value and keep a backup copy
    param_ref = params['red.reflectance.value'].torch()
    print(param_ref)
:py:func:`~mitsuba.python.autodiff.render_torch()` 函数的工作方式与 :py:func:`~mitsuba.python.autodiff.render()` 类似，
只是它返回的是一个 PyTorch 张量。

.. code-block:: python

    # Render a reference image (no derivatives used yet)
    image_ref = render_torch(scene, spp=8)
    crop_size = scene.sensors()[0].film().crop_size()
    write_bitmap('out.png', image_ref, crop_size)

和之前一样，我们更改输入参数之一，并初始化优化器。

.. code-block:: python

    # Change the left wall into a bright white surface
    params['red.reflectance.value'] = [.9, .9, .9]
    params.update()

    # Which parameters should be exposed to the PyTorch optimizer?
    params_torch = params.torch()

    # Construct a PyTorch Adam optimizer that will adjust 'params_torch'
    opt = torch.optim.Adam(params_torch.values(), lr=.2)
    objective = torch.nn.MSELoss()

需要注意的是，场景参数是有效复制的：我们曾经使用 Enoki 数组（``params``）和 PyTorch 数组（``params_torch``）表示。
为了执行可微渲染，函数 :py:func:`~mitsuba.python.autodiff.render_torch()` 需要两者均作为参数。因为 PyTorch 识别
可微参数的技术特点，将  ``params_torch`` 扩展为关键字参数列表（``**params_torch``）还是很有必要的。然后该函数使两种
表示保持同步，并在底层计算图之间创建接口。

主要的优化循环如下所示：

.. code-block:: python

    for it in range(100):
        # Zero out gradients before each iteration
        opt.zero_grad()

        # Perform a differentiable rendering of the scene
        image = render_torch(scene, params=params, unbiased=True,
                             spp=1, **params_torch)

        write_bitmap('out_%03i.png' % it, image, crop_size)

        # Objective: MSE between 'image' and 'image_ref'
        ob_val = objective(image, image_ref)

        # Back-propagate errors to input parameters
        ob_val.backward()

        # Optimizer: take a gradient step
        opt.step()

        # Compare iterate against ground-truth value
        err_ref = objective(params_torch['red.reflectance.value'], param_ref)
        print('Iteration %03i: error=%g' % (it, err_ref * 3))

.. warning::

    **Memory caching**: 当 Enoki 或 PyTorch 中的 GPU 序列被销毁时，其内存不会立即释放回 GPU。 
    这样做的原因是分配和释放 GPU 内存都是非常昂贵的操作，因此，任何未使用的内存都将放入缓存中以供以后重用。


    事实是当只使用 Enoki 时或者只使用 PyTorch 时这个问题并不要紧，但是当两者同时使用时可能就会有问题了，
    因为一个系统的缓存可能会变得非常大，以至于另一个系统的分配失败，尽管技术上有大量可用内存可用。

    如果你注意到你的程序崩溃并报错 out-of-memory error，请尝试将参数 ``malloc_trim=True`` 传递给函数 ``render_torch`` 。
    这会在执行任何 Enkio 代码之前刷新 PyTorch 的缓存，反之亦然。这应该是不得已的方法---通常来讲，最好通过降低每像素采样
    来减少内存需要，因为刷新缓存会导致严重的性能损耗。

.. note::

    可以在 :file:`docs/examples/10_diff_render/invert_cbox_torch.py`. 中找到本节教程的完整脚本。
