.. _sec-variants-cpp:

Variants in C++
===============

正如 :ref:`choosing variants <sec-variants>` 一节中所介绍的， Mitsuba 2 可以按照不同的变体模式
进行编译，而这些变体代表了不同的计算后端和颜色表示方式。若要从某一实现中启用这种重定向功能，系统需要依赖于
C++ 模版和元编程。实际上，Mitsuba 2中大多数的 C++ 类和函数都是带有以下两个参数的模版：

- ``Float`` 
- ``Spectrum``.

这两个参数完全对应于之前提到的计算后端和颜色表示方式。在编译过程中， Mitsuba 的构建系统读取  ``mitsuba.conf`` 文件，
并将所选变体模式替换到模版参数中。例如：

.. code-block:: text

    "scalar_rgb": {
        "float": "float",
        "spectrum": "Color<Float, 3>"
    },

这会显式的引发
`模版实例化 <https://en.cppreference.com/w/cpp/language/class_template#Explicit_instantiation>`_

.. code-block::

    Float    = float;
    Spectrum = Color<Float, 3>;

生成的 C++ 符号将添加到共享链接库中（ ``dist/libmitsuba-core.so``， ``dist/libmitsuba-render.so``， ``dist/plugin/*.so`` 等）。
在运行时，由用户指定的变体模式将决定渲染器所使用的符号集。

Type aliases
------------

当然单单 ``Float`` 和 ``Spectrum`` 两个类型不会构成渲染器使用的所有类型。渲染器还可以访问整型算数类型，向量，
法线，点，光线，矩阵等各种结构。对于一些更复杂的结构（如交点和采样记录）也是可以随意使用的。

在给定的变体模式的上下文中去定义这些类型会有些繁琐。为了简化这一过程，Mitsuba 提供了一个辅助宏，从 ``Float`` 和 ``Spectrum`` 的定义
中推断并导入合适的类型。

例如，在辅助宏开始推断模版函数后，我们就可以使用其他 Mitsuba 模版类型（比如，`Vector2f`、`Ray3f`、`SurfaceInteraction` 等）就好像
它们没有模版化一样。

.. code-block:: c++

    template <typename Float, typename Spectrum>
    void my_function() {
        /// Import type aliases (e.g. using Vector3f = Vector<Float, 3>;)
        MTS_IMPORT_TYPES()

        // Can now use those types as if they were not templated
        Point3f p(4.f, 3.f, 0.f);
        Vector3f v(1.f, 0.f, 0.f);
        Ray3f ray(p, v);
        std::cout << ray << std::endl;
    }

.. note:: 

    ``MTS_VARIANT`` 宏常用作速记符号，而不是相较冗长的 ``template <typename Float, typename Spectum>``。

下一章节讲更详细的介绍这些宏：

- :ref:`macro-import-types`


Branching and masking
---------------------

在处理矢量化计算后台时（例如，``packet_*``，``gpu_*``），需要额外检查 C++ 的分支逻辑（特别是 ``if`` 结构）。

回想一下 scalar 模式下的光线求交结果，生成的 :py:class:`~mitsuba.render.SurfaceInteraction3f` 记录的是与 *单个曲面* 的相交信息。
在这种情况下，使用普通 ``if`` 语句的条件逻辑可以很好工作。

在另一方面，在矢量化后端（如 ``gpu_rgb``）中相同的数据结构保存的是与 *多个曲面的* 相交信息。由于绝大多数的条件只可能对其中部分元素为真，
所以不能再使用普通的 ``if`` 语句来执行条件逻辑。

替换操作 ``enoki::select(mask, arg1, arg2)`` 接受一个 *mask* 参数（通常是比较操作的结果）并对每个通道并
行计算 ``(mask ? arg1 : arg2)`` 。更多关于使用 mask 的信息，请参阅 `Enoki's documentation <https://enoki.readthedocs.io/en/master/basics.html#working-with-masks>`_。
下面是一个对比示例：

.. code-block:: cpp

    // --------------------
    // Scalar code (discouraged)

    Scene scene = ...;
    Ray3f ray = ...;
    SurfaceInteraction3f si = scene->ray_intersect(ray);

    if (si.is_valid())
        return 1.f;
    else
        return 0.f;

    // --------------------
    // Generic code

    Scene scene = ...;
    Ray3f ray = ...;
    SurfaceInteraction3f si = scene->ray_intersect(ray);

    return enoki::select(si.is_valid(), 1.0f, 0.f);

此外，大多数的函数/方法都有一个 *可选的* `active` 参数，用于编码哪些通道是激活的。比如在上面的例子中，
我们可以将此信息提供给 ``ray_intersect`` 例程，从而避免无效条目的相关计算。这样写的话，代码则更新为：

.. code-block:: cpp

    // Mask specifying the active lanes
    Mask active = ...;

    Scene scene = ...;
    Ray3f ray = ...;
    SurfaceInteraction3f si = scene->ray_intersect(ray, active);

    return enoki::select(active & si.is_valid(), 1.0f, 0.f);

CUDA backend synchronization point
----------------------------------

正如 Enoki 文档 `GPU arrays <https://enoki.readthedocs.io/en/master/gpu.html#suggestions-regarding-horizontal-operations>`_ 这
一章节所描述的， ``gpu_*`` 计算后端依赖于通过 NVIDIA 的中间语言 PTX 来动态生成内核的 JIT 编译器。 该 JIT 编译器对于 *纵向操作*
（加法、乘法、聚集、发散等）非常高效。然而 *横向操作* 的应用（比如 ``enoki::any()``，``enoki::all()``，``enoki::hsum()`` 等）
会刷新 ``CUDAArray<T>`` 的所有当前计算队列，这会限制并行度。

在很多情况下，如果能带来性能优势的话，可以安全的跳过与横向 mask 有关的操作。出于这一原因，Mitsuba 2代码库频繁的使用
替换语句（``any_or<>()``，``all_or<>()`` 等）以在 GPU 标签下跳过估算来简化操作。

例如，下面示例中的代码 ``...`` 只会在 ``scalar_*`` 变体模式下 ``condition`` 为 ``true`` 才会执行。

.. code-block:: cpp

    Mask condition = ...;
    if (any_or<true>(condition)) {
        ...
    }

而在 ``packet_*`` 模式下，至少 *一个元素* 的 ``condition`` 为 ``true`` 才会执行。在 ``gpu_*`` 变体模式下，
我们通常要处理包含数百万个元素的数组，并且很可能在任何情况下至少都有一个元素都会触发执行 ``...`` 。那么 ``any_or<true>(condition)`` 会跳过代价
昂贵的横向操作，并始终假设条件为真。

Pointer types
-------------

``MTS_IMPORT_TYPES`` 宏还为指针的导入提供了变体模式特定的类型别名。有一点很重要：例如，考虑下与曲面交点
相关的 ``BSDF`` 。 在 scalar 变体中，使用  ``const BSDF *`` 指针就是很好的表示方式。然而在矢量化变体（``gpu_*``， ``packet_*``）
中，会有多个曲面交点构成的交点集，因此单一指针被 *指针数组* 所替代。这些指针的别名如下所示：

.. code-block:: c++

    // Imports BSDFPtr, EmitterPtr, etc..
    MTS_IMPORT_TYPES()

    Scene scene = ...;
    Mask active = ...;
    Ray3f ray = ...;
    SurfaceInteraction3f si = scene->ray_intersect(ray, active);

    // Array of pointers if Float is an array
    BSDFPtr bsdf = si.bsdf();

    // Enoki is able to dispatch method calls involving arrays of pointers
    bsdf->eval(..., active);

关于矢量化方法调用的更多信息，请参阅 `Enoki documentation <https://enoki.readthedocs.io/en/master/calls.html>`_ 。

Variant-specific code
---------------------

在整个代码库中，大量使用 C++17 的 ``if conexpr`` 语句将代码片段约束为特定的变体。例如，下面的 C++ 代码片段将光谱值转换成了 XYZ 三
色值，这取决于当前编译变体决定的颜色表示方式。

.. code-block:: c++

    Ray3f ray = ...;
    Mask active = ...;
    Spectrum result = compute_stuff(ray, active);

    Color3f xyz;
    if constexpr (is_monochromatic_v<Spectrum>)
        xyz = result.x();
    else if constexpr (is_rgb_v<Spectrum>)
        xyz = srgb_to_xyz(result, active);
    else
        xyz = spectrum_to_xyz(result, ray.wavelengths, active);

由于 ``if constexpr`` 是在编译时解析的，所以该分支不会导致任何运行时开销。``if constexpr`` 的另一个有用特性是，
它会禁止未启用分支中的编译错误。这是因为一般代码使用的普通（不带 ``constexpr``） ``if`` 语句可能有潜在的编译错误（例如，试图访问一个不存
在于所有变体的类成员时）。

Mitsuba 提供了各种版本的 *type-traits* 例如 ``is_monochromatic_v`` 用来查询特定变体的属性。你可以在 :file:`include/mitsuba/core/traits.h` 中找到它们。
