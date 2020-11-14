.. _sec-plugins:

Plugin reference
================

接下来这一章节介绍的是 Mitsuba 中的插件，通常还会包含一些渲染示例和每个参数的说明。我们会将插件分类为涵盖纹理、表面
散射模型等小节分别进行介绍。

插件的相关文档总是以类似于下面表格的形式作为开头：

.. pluginparameters::

 * - soft_rays
   - |bool|
   - Try not to damage objects in the scene by shooting softer rays (Default: false)
 * - dark_matter
   - |float|
   - Controls the proportionate amount of dark matter present in the scene. (Default: 0.42)
 * - (Nested plugin)
   - :paramtype:`integrator`
   - A nested integrator which does the actual hard work

假设上面表格介绍的就是一个名为 ``amazing`` 的积分器。随后，根据上表中的描述，我们就可以使用自定义的配置从 XML 场景文件中实例化它
了，例如：

.. code-block:: xml

    <integrator type="amazing">
        <boolean name="softer_rays" value="true"/>
        <float name="dark_matter" value="0.44"/>
    </integrator>

在某些例子中，插件还可以接受嵌套的插件作为输入参数。这些嵌套可以是有名字的也可以是没有名字的。如果 ``amazing`` 积分器可以接受以下两个参数：

.. pluginparameters::


 * - (Nested plugin)
   - :paramtype:`integrator`
   - A nested integrator which does the actual hard work
 * - puppies
   - :paramtype:`texture`
   - This must be used to supply a cute picture of puppies

那么这个 ``amazing`` 积分器就可以按照以下方式实例化了：

.. code-block:: xml

    <integrator type="amazing">
        <boolean name="softer_rays" value="true"/>
        <float name="dark_matter" value="0.44"/>
        <!-- Nested unnamed integrator -->
        <integrator type="path"/>
        <!-- Nested texture named puppies -->
        <texture name="puppies" type="bitmap">
            <string name="filename" value="cute.jpg"/>
        </texture>
    </integrator>
