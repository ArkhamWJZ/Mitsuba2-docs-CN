Plugins & Macros
================

该章节简明的回顾了基本插件定义时的框架，以阐明各种 ``MTS_* `` 宏的作用，比如有些宏负责导入类型、
实例化变量和提供运行时类型的信息。

Example code
------------

考虑以下假设的插件 ``MyPlugin``：

.. code-block:: cpp

    NAMESPACE_BEGIN(mitsuba)

    template <typename Float, typename Spectrum>
    class MyPlugin : public PluginInterface<Float, Spectrum> {
    public:
        MTS_IMPORT_BASE(PluginInterface, m_some_member, some_method)
        MTS_IMPORT_TYPES()

        MyPlugin();

        Spectrum foo(..., Mask active) const override {
            MTS_MASKED_FUNCTION(ProfilerPhase::MyEval, active)
            // ...
        }

        /// Declare RTTI data structures
        MTS_DECLARE_CLASS()
    protected:
        /// Important: declare a protected virtual destructor
        virtual ~MyPlugin();

    };

    /// Implement RTTI data structures
    MTS_IMPLEMENT_CLASS_VARIANT(MyPlugin, PluginInterface)
    MTS_EXPORT_PLUGIN(MyPlugin, "Description of my plugin")
    NAMESPACE_END(mitsuba)

Macros
------

需要注意的是，接下来所列出来的宏并非都出现在了上面的示例代码中。

:monosp:`MTS_MASK_ARGUMENT(mask)`
*********************************

这个宏通常出现在以掩码为参数的函数开头。

.. code-block:: cpp

    void my_method(..., Mask active) {
        MTS_MASK_ARGUMENT(active);
    }

scalar 变体实际上不需要掩码：如果一个函数被调用了，那么我们假设其 ``active=true`` 。不幸的是，
掩码在此例中可能会因为额外分支的产生而降低性能。因此，``MTS_MASK_ARGUMENT`` 扩展为：

.. code-block:: cpp

    if constexpr (is_scalar_v<Float>)
        active = true;

它将掩码参数转换为 scalar 目标上的编译时常量，从而允许编译器优化掉不需要的分支。``MTS_MASK_ARGUMENT`` 掩码
参数也是下一个宏中的一部分。


:monosp:`MTS_MASKED_FUNCTION(phase, mask)`
------------------------------------------

这个宏是建立在 ``MTS_MASK_ARGUMENT`` 宏基础上的，通常出现在以掩码为参数的函数开头。

.. code-block:: cpp

    void my_method(..., Mask mask) {
        MTS_MASKED_FUNCTION(ProfilerPhase::MyMethod, active);
    }

scalar 变体实际上不需要掩码：如果一个函数被调用了，那么我们假设其 ``active=true`` 。不幸的是， 
掩码在此例中可能会因为额外分支的产生而降低性能。因此，``MTS_MASKED_FUNCTION`` 扩展为：

.. code-block:: cpp

    if constexpr (is_scalar_v<Float>)
        active = true;
    ScopedPhase _(ProfilerPhase::MyMethod);

它将掩码参数转换为 scalar 目标上的编译时常量，从而允许编译器优化掉不需要的分支。

Mitsuba 带有一个强大的采样侦查器以帮助追踪渲染过程中的热点区域。这个宏的最后一行（``ScopedPhase``）通知了该侦查器，
我们当前正在执行一个属于侦查器阶段 ``phase`` 的函数。


:monosp:`MTS_IMPORT_BASE(Name, ...)`
------------------------------------
因为大多数 Mitsuba 的类都是模版，所以父类的属性和方法在默认情况下是不可见的。它们可以通过 ``using Base::some_method;`` 语
句显式导入，但是重复编写很多这样的语句是很麻烦的。使用可变宏 ``MTS_IMPORT_BASE`` 可以扩展为任意多这种 ``using`` 语句。例如：

.. code-block:: cpp

    MTS_IMPORT_BASE(Name, m_some_member, some_method)

相当于扩展为：

.. code-block:: cpp

    using Base = Name<Float, Spectrum>;
    using Base::m_some_member;
    using Base::some_method;

.. _macro-import-core-types:

:monosp:`MTS_IMPORT_CORE_TYPES()`
---------------------------------

该宏将生成一系列用于导入 Mitsuba 核心库的 ``using`` 声明（例如，``Vector{1/2/3}{i/u/f/d}``，``Point{1/2/3}{i/u/f/d}`` 等）。
它们是从 ``Float`` 定义中自动推断出来的。

.. note::

    在计算此宏之前，必须存在一个名叫 ``Float`` 的参数。

例如：

.. code-block:: cpp

    using Float = float;

    MTS_IMPORT_CORE_TYPES()

    // expands to:

    // ...
    using Point2f = Point<Float, 2>;
    using Point3f = Point<Float, 3>;
    // ...
    using BoundingBox3f = BoundingBox<Point3f>;
    // ...


.. _macro-import-types:

:monosp:`MTS_IMPORT_TYPES(...)`
-------------------------------

该宏调用 ``MTS_IMPORT_CORE_TYPES()`` ，并进一步导入如  ``Ray3f``， ``SurfaceInteraction3f``， ``BSDF`` 等渲染相关类型。
这些模版的别名依赖于之前声明的 ``Float`` 和 ``Spectrum``。


还可以将其他类型作为参数进行传递给那些创建的模版别名。

.. code-block:: cpp

    using Float    = float;
    using Spectrum = Spectrum<Float, 4>;

    MTS_IMPORT_TYPES(MyType1, MyType2)

    // expands to:

    MTS_IMPORT_CORE_TYPES()
    // ...
    using Ray3f = Ray<Point<Float, 3>, Spectrum>;
    // ...
    using SurfaceInteraction3f = SurfaceInteraction<Float, Spectrum>;
    // ...
    using MyType1 = MyType1<Float, Spectrum>; // alias for the optional parameters
    using MyType2 = MyType2<Float, Spectrum>;


:monosp:`MTS_DECLARE_CLASS()`
-----------------------------

这个宏应该在插件的 :monosp:`class` 声明中进行调用。 它将声明 RTTI（运行时类型信息）的数据结构，这些
数据结构对于下列操作非常有用：

- 检查对象是否派生自某个 :monosp:`class` 
- 在运行时确定 :monosp:`class` 的父类
- 通过名称实例化一个 :monosp:`class`
- 从二进制数据流中取消序列化一个 :monosp:`class`


:monosp:`MTS_IMPLEMENT_CLASS_VARIANT(Name, Parent)`
---------------------------------------------------

应该在自定义插件类的 ``.cpp`` 文件底部调用此宏。它的作用是为该插件初始化 RTTI 数据结构，之前这些数据
结构是使用 ``MTS_DECLARE_CLASS()`` 声明的。

- ``Name`` 参数应该是插件的 :monosp:`class` 名。
- ``Parent`` 参数应该取插件接口的 :monosp:`class` 名。


:monosp:`MTS_EXPORT_PLUGIN(Name, Descr)`
----------------------------------------

此宏将显式实例化插件启用的所有变体：

.. code-block:: cpp

    MTS_EXPORT_PLUGIN(Name, "My fancy plugin")

    // expands to:

    template class MTS_EXPORT Name<float, Color<float, 1>>    // scalar_mono
    template class MTS_EXPORT Name<float, Spectrum<float, 4>> // scalar_spectral
    // ...

它还将可读性的 ``Descr`` 描述和 Mitsuba 图形用户界面将来可能使用的插件关联了起来。
