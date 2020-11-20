.. _sec-differentiable-rendering-advanced:

进阶的实例
======================

Differentiating textures
------------------------

前面的示例展示了标量颜色参数的微分。现在，我们将以环境贴图发射器为例，展示如何使用纹理化参数。
接下来我们将展示微分渲染是如何在纹理参数上工作的，重点介绍环境贴图发射器的例子。

.. subfigstart::
.. subfigure:: ../../../resources/data/docs/images/autodiff/bunny.jpg
   :caption: A metallic Stanford bunny surrounded by an environment map.
.. subfigure :: ../../../resources/data/docs/images/autodiff/museum.jpg
   :caption: The ground-truth environment map that we will attempt to recover from the image on the left.
.. subfigend::
   :label: fig-bunny-autodiff

示例场景资源可以从 `这里下载 <http://mitsuba-renderer.org/scenes/bunny.zip>`_ 包含了一个金属的 Stanford bunny 周围是
博物馆一样的环境。和之前一样，我们先加载场景并枚举其中的可微参数：

.. code-block:: python

    import enoki as ek
    import mitsuba
    mitsuba.set_variant('gpu_autodiff_rgb')

    from mitsuba.core import Float, Thread
    from mitsuba.core.xml import load_file
    from mitsuba.python.util import traverse
    from mitsuba.python.autodiff import render, write_bitmap, Adam

    # Load example scene
    Thread.thread().file_resolver().append('bunny')
    scene = load_file('bunny/bunny.xml')

    # Find differentiable scene parameters
    params = traverse(scene)

然后，我们制作真实环境图的备份副本并生成参考渲染

.. code-block:: python

    # Make a backup copy
    param_res = params['my_envmap.resolution']
    param_ref = Float(params['my_envmap.data'])

    # Discard all parameters except for one we want to differentiate
    params.keep(['my_envmap.data'])

    # Render a reference image (no derivatives used yet)
    image_ref = render(scene, spp=16)
    crop_size = scene.sensors()[0].film().crop_size()
    write_bitmap('out_ref.png', image_ref, crop_size)

现在让我们把环境图更改为统一的白色光照明环境。``my_envmap.data`` 参数是一个 RGBA bitmap ，该参数被线性化为尺寸为  ``param_res[0] x param_res[1] x 4`` （125'000）
的 1D 数组。

.. code-block:: python

    # Change to a uniform white lighting environment
    params['my_envmap.data'] = ek.full(Float, 1.0, len(param_ref))
    params.update()

最后，我们使用梯度优化一起估算所有的 125K 个参数。优化循环与前面的示例相同，除此之外我们现在还可以在每次迭代中输出当前环境图像。

.. code-block:: python

    # Construct an Adam optimizer that will adjust the parameters 'params'
    opt = Adam(params, lr=.02)

    for it in range(100):
        # Perform a differentiable rendering of the scene
        image = render(scene, optimizer=opt, unbiased=True, spp=1)
        write_bitmap('out_%03i.png' % it, image, crop_size)
        write_bitmap('envmap_%03i.png' % it, params['my_envmap.data'],
                     (param_res[1], param_res[0]))

        # Objective: MSE between 'image' and 'image_ref'
        ob_val = ek.hsum(ek.sqr(image - image_ref)) / len(image)

        # Back-propagate errors to input parameters
        ek.backward(ob_val)

        # Optimizer: take a gradient step
        opt.step()

        # Compare iterate against ground-truth value
        err_ref = ek.hsum(ek.sqr(param_ref - params['my_envmap.data']))
        print('Iteration %03i: error=%g' % (it, err_ref[0]))

接下来的视频显示了前100次迭代的收敛行为。图像迅速分解为目标图像。 图像中的小黑色区域对应了由于光的最大反弹次数限制而
忽略相互反射的网格部分。

.. raw:: html

    <center>
        <video controls loop autoplay muted
        src="https:////rgl.s3.eu-central-1.amazonaws.com/media/uploads/wjakob/2020/03/03/bunny_render.mp4"></video>
    </center>

下面的图像展示了从每个步骤中重建出来的图像。未观察到的区域不受梯度步长的影响保持了白色。

.. raw:: html

    <center>
        <video controls loop autoplay muted
        src="https://rgl.s3.eu-central-1.amazonaws.com/media/uploads/wjakob/2020/03/03/bunny_envmap.mp4"></video>
    </center>

该图像的噪声仍然很多，甚至包含了一些负（！）区域。这是因为上面前向渲染时发生的信息丢失问题，我们定义的优化问题是相当不明确的。
我们发现的解决方案可以很好地优化目标（即，渲染图像与目标匹配），但是重建出来的纹理可能不符合我们的预期。 在这种情况下，建议进
一步进行正则化（加入非负性，平滑等操作）。
