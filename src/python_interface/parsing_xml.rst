XML 代码解析
=================

Mitsuba 提供了三个用于在Python中创建场景和对象的函数：

- :py:func:`mitsuba.core.xml.load_file`: load files on disk
- :py:func:`mitsuba.core.xml.load_string`: load arbitrary strings
- :py:func:`mitsuba.core.xml.load_dict`: load from Python dictionary (`dict`)

有关 Mitsuba 场景描述语言的更多信息，请参阅 :ref:`chapter <sec-file-format>` 章节。

Loading a scene
---------------

下面是有关如何从 XML 文件中加载 Mitsuba 场景的详细 Python 示例。

.. code-block:: python

    import os
    import mitsuba
    mitsuba.set_variant('scalar_rgb')
    from mitsuba.core import Thread
    from mitsuba.core.xml import load_file

    # Absolute or relative path to the XML file
    filename = 'path/to/my/scene.xml'

    # Add the scene directory to the FileResolver's search path
    Thread.thread().file_resolver().append(os.path.dirname(filename))

    # Load the scene for an XML file
    scene = load_file(filename)

因为XML 文件中场景可能通过相对路径引用了一些外部资源，如网格和纹理，所以必须告诉 Mitsuba 在哪里寻找这些文件。
上面的代码通过 thread-local :py:class:`mitsuba.core.FileResolver` 类实现了这一点


Passing arguments to the scene
------------------------------

正如在 :ref:`Mitsuba's scene description language
<sec-scene-file-format-params>` 章节中讨论的，场景文件可以包含前缀为 ``$`` 符号的命名参数：

.. code-block:: xml
    :name: cbox-xml

    <!-- sample per pixel parameter -->
    <default name="spp" value="8"/>
    ...
    <sampler type="independent">
        <integer name="sample_count" value="$spp"/>
    </sampler>

这些带前缀的参数在加载场景时，可以使用关键字显式定义。

.. code-block:: python

    scene = load_file(filename, spp=1024)


Loading a Mitsuba object
------------------------

可以使用 XML 解释器创建单个 Mitsuba 对象的实例（例如 BSDF），参考下面的 Python 代码片段：

.. code-block:: python

    from mitsuba.core.xml import load_string

    diffuse_bsdf = load_string("<bsdf version='2.0.0' type='diffuse'></bsdf>")

Mitsuba 的测试套件经常通过这种方式检查单个系统组件的行为。


Creating objects using Python dictionaries
------------------------------------------

在 Python 中构建 Mitsuba 对象的一种更方便的方法是使用 :py:func:`mitsuba.core.xml.load_dict` ，
该函数接受一个 Python 字典类型的参数。该字典应该遵循和 Mitsuba 场景描述用的 XML 文件一样的结构。

字典中应该始终包含一个条目 ``"type"`` 来指定实例化的插件名称。字典的键必须是字符串，表示的是属性的名称。
属性的类型可以从 Pythn 中的简单类型 (例如： ``bool``，``float``，``int``，``string`` 等）推导出来。
可以提供一个字典作为值来使用，这会被用来创建嵌套对象，如同 XML 场景文件中描述的一样。

下面的代码片段说明了 XML 代码和 Python 字典结构之间的相似性：

*XML:*

.. code-block:: xml

    <shape type="obj">
        <string name="filename" value="dragon.obj"/>
        <bsdf type="roughconductor">
            <float name="alpha" value="0.01"/>
        </bsdf>
    </shape>


*Python dictionary:*

.. code-block:: python

    {
        "type" : "obj",
        "filename" : "dragon.obj",
        "something" : {
            "type" : "roughconductor",
            "alpha" : 0.01
        }
    }


下面是一个有关如何使用该函数的更具体示例：

.. code-block:: python

    from mitsuba.core.xml import load_dict

    sphere = load_dict({
        "type" : "sphere",
        "center" : [0, 0, -10],
        "radius" : 10.0,
        "flip_normals" : False,
        "bsdf" : {
            "type" : "dielectric"
        }
    })

可以在 Python 字典中提供另一个 Mitsuba 对象，而不用通过嵌套字典：

.. code-block:: python

    # First create a BSDF (could use xml.load_string(..) as well)
    my_bsdf = load_dict({
        "type" : "roughconductor",
        "alpha" : 0.14,
    })

    # Pass the BSDF object in the dictionary
    sphere = load_dict({
        "type" : "sphere",
        "something" : my_bsdf
    })

为方便起见，嵌套字典可以提供一个等于 ``"rgb"`` 或 ``"spectrum"`` 的 ``"type"`` 条目。 
与 XML 解析器类似，该字典中的 ``“value”`` 条目将用于实例化正确的 `Spectrum` 插件。
(详情请参阅 :ref:`corresponding section <sec-spectra>`)

下面是嵌套字典中使用 ``"value"`` 条目的一些示例：

.. code-block:: python

    # Passing gray-scale value
    "color_property" : {
        "type": "rgb",
        "value": 0.44
    }

    # Passing tri-stimulus values
    "color_property" : {
        "type": "rgb",
        "value": [0.7, 0.1, 0.5]
    }

    # Providing a spectral file
    "color_property" : {
        "type": "spectrum",
        "filename": "filename.spd"
    }

    # Providing a list of (wavelength, value) pairs
    "color_property" : {
        "type": "spectrum",
        "value": [(400.0, 0.5), (500.0, 0.8), (600.0, 0.2)]
    }

下面的示例展示了通过使用 :py:func:`mitsuba.core.xml.load_dict` 函数来构建 Mitsuba 场景：

.. code-block:: python

    scene = load_dict({
        "type" : "scene",
        "myintegrator" : {
            "type" : "path",
        },
        "mysensor" : {
            "type" : "perspective",
            "near_clip": 1.0,
            "far_clip": 1000.0,
            "to_world" : ScalarTransform4f.look_at(origin=[1, 1, 1],
                                                   target=[0, 0, 0],
                                                   up=[0, 0, 1]),
            "myfilm" : {
                "type" : "hdrfilm",
                "rfilter" : { "type" : "box"},
                "width" : 1024,
                "height" : 768,
            },
            "mysampler" : {
                "type" : "independent",
                "sample_count" : 4,
            },
        },
        "myemitter" : {"type" : "constant"},
        "myshape" : {
            "type" : "sphere",
            "mybsdf" : {
                "type" : "diffuse",
                "reflectance" : {
                    "type" : "rgb",
                    "value" : [0.8, 0.1, 0.1],
                }
            }
        }
    })

正如在 XML 场景中描述的一样，可以引用 `dict` 中的其他对象，只要在使用字典引用之前声明那些对象就行了。
为此，你可以指定一个带有 ``"type":"ref"`` 和 ``"id"`` 条目的嵌套字典。可以通过字典中的 ``key`` 值
引用对象。如果定义了 ``id`` 标识的话，也可以通过它来引用对象。

.. code-block:: python

    {
        "type" : "scene",
        # this BSDF can be referenced using its key "bsdf_id_0"
        "bsdf_key_0" : {
            "type" : "roughconductor"
        },

        "shape_0" : {
            "type" : "sphere",
            "mybsdf" : {
                "type" : "ref",
                "id" : "bsdf_key_0"
            }
        }

        # this BSDF can be referenced using its key "bsdf_key_1" or its id "bsdf_id_1"
        "bsdf_key_1" : {
            "type" : "roughconductor",
            "id" : "bsdf_id_1"
        },

        "shape_2" : {
            "type" : "sphere",
            "mybsdf" : {
                "type" : "ref",
                "id" : "bsdf_id_1"
            }
        },

        "shape_3" : {
            "type" : "sphere",
            "mybsdf" : {
                "type" : "ref",
                "id" : "bsdf_key_1"
            }
        }
    }