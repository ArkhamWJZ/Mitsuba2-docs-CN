.. _developer_guide-polarization:

Polarization
==========================

当在 *polarized* 模式下（例如， `scalar_spectral_polarized` ）渲染模型时， Mitsuba 2将会在整个模拟过程中一直追踪光线的偏振状态。
这些偏振状态会记录在四维的 *Stokes vector* :math:`\mathbf{s} = [s_0, s_1, s_2, s_3]` 中，此向量将振荡截面参数化为椭圆形状 :cite:`Collett1993PolarizedLight`。

- :math:`s_0` 相当于辐射度，不携带有关偏振的信息。
- :math:`s_1` 用于区分水平偏振和垂直偏振，其中 :math:`s_1 = \pm 1` 分别代表完全水平偏振或垂直偏振。
- :math:`s_2` 类似于:math:`s_1` ，但是是在 45 度对角线上区分偏振。
- :math:`s_3` 用于区分左右圆偏振，其中 :math:`s_3 = \pm 1` 分别代表完全右或左圆偏振。
- 物理上合理的 stokes vector 需要满足条件 :math:`s_0 \ge \sqrt{s_1^2 + s_2^2 + s_3^2}`。

值得注意的是，此信息需要按照波长单独跟踪。

.. image:: ../../images/polarization_stokes_vector.svg
    :width: 90%
    :align: center

这里的一个关键细节是如何选择执行测量过程的参考坐标系（或标架）？只要与光束正交即可，这纯粹是一种规定。
Mitsuba 2遵循标准教科书中对于偏振的描述，并且选择 Z 轴沿着光线传播方向的右手系，如下所示：

.. image:: ../../images/polarization_wave.svg
    :width: 60%
    :align: center

换句话说，当使用 Stokes vector 描述偏振状态时，我们是从传感器一侧观察光束内部的。
下面所示的水平偏振光（形式如：:math:`\mathbf{s} = [1, 1, 0, 0]`）只是在特定的 :math:`{\mathbf{x}, \mathbf{y}}` 
标架下描述的。完全等价于在另一个不同标架 :math:`\mathbf{x}', \mathbf{y}'` 中的另一种相应描述，此时用 Stokes vector
描述的偏振状态被替代为 :math:`\mathbf{s} = [1, 0, -1, 0]` ：

.. image:: ../../images/polarization_stokes_rotation.svg
    :width: 50%
    :align: center

在渲染过程中，光线当然也会与其他影响偏振的物体发生交互。这些变更由 *Mueller matrix* :math:`\mathbf{M} \in \mathbb{R}^{4\times4}`
描述。在发生反射（或透射）之后，入射（:math:`\mathbf{s}_i`）和出射（:math:`\mathbf{s}_o`）的 Stokes vector 值可以被
替换为 :math:`\mathbf{s}_o = \mathbf{M}\mathbf{s}_i`。这引发了与 Stokes vector 下关于参考标架相同的问题，只是现在
需要为交互中的入射方向和出射方向定义 *两个标架*：

.. image:: ../../images/polarization_mueller_matrix.svg
    :width: 100%
    :align: center

（需要注意的是，这里 :math:`\omega_i` 是指向光源的向量，因此标架应该基于 :math:`-\omega_i` 建立，这是沿着光的传播方向的）

对于 Mueller 矩阵，非极化渲染中使用的标准 BSDF 定义 :math:`f_r(\lambda, \omega_i, \omega_o)` 可以被推广为极化 pBSDF  :math:`\mathbf{M}(\lambda, \omega_i, \omega_o)`。
Mitsuba 2中包含了导体和绝缘体 pBSDF 的实现（遵循了 polarized Fresnel equations），以及一些标准光学元件，例如线性偏振器和延迟器。

具有多相互作用的完整光线传播模拟会让事情变得更加复杂。只有当它们的参考标架与光路匹配时，后续光路上的米勒矩阵乘法才有意义。 
在实践中，需要通过额外的坐标系旋转（下图中的红色箭头）来确保这一点。

.. image:: ../../images/polarization_light_transport.svg
    :width: 100%
    :align: center

幸运的是，标架旋转本身就可以用一个简单的 Mueller matrix 表示。在 Mitsuba 2中，pBSDFs 的采样或估算程序总是返回已
经在出射入射两端预旋转对齐了的矩阵，使整个过程更为简单。


极化渲染中的最后一个重要细节就是 reciprocity 问题。不幸的是，pBSDFs 一般不会像标准 BSDFs 一样遵循 *Helmholtz reciprocity*，
pBSDFs 只会沿着光线方向定义。这使得一些渲染算法（如双向 BRDF）的实现略有复杂 :cite:`Jarabo2018BidirectionalPol` :cite:`Mojzik2016BidirectionalPol`:

- 当跟踪光源发出的辐射时（例如，光线追踪器），一切都遵循标准的光线传播流程。在这里模拟过程可以简单的追踪 Stokes vectors 而不是
辐射度，并且执行必要的 Mueller matrix 乘法。
- 与追踪从传感器 “发出” 光线时（例如，在路径追踪器中）的情况恰恰相反，我们需要从传感器一端追踪发射量（Mueller matrix 的值）。
为了在光线相交的曲面乘以 Mueller matrices 时能保持正确的运算顺序，这一点需要额外注意。
