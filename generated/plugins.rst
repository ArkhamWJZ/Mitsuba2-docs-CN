.. _sec-shapes:

Shapes
======

This section presents an overview of the shape plugins that are released along with the renderer.

In Mitsuba 2, shapes define surfaces that mark transitions between different types of materials. For
instance, a shape could describe a boundary between air and a solid object, such as a piece of rock.
Alternatively, a shape can mark the beginning of a region of space that isn’t solid at all, but
rather contains a participating medium, such as smoke or steam. Finally, a shape can be used to
create an object that emits light on its own.

Shapes are usually declared along with a surface scattering model named *BSDF* (see the :ref:`respective section <sec-bsdfs>`). This BSDF characterizes what happens at the surface. In the XML scene description language, this might look like the following:

.. code-block:: xml

    <scene version=2.0.0>
        <shape type=".. shape type ..">
            .. shape parameters ..

            <bsdf type=".. BSDF type ..">
                .. bsdf parameters ..
            </bsdf>

            <!-- Alternatively: reference a named BSDF that
                 has been declared previously

                 <ref id="my_bsdf"/>
            -->
        </shape>
    </scene>

The following subsections discuss the available shape types in greater detail... _sec-bsdfs:

BSDFs
=====

    .. image:: ../../resources/data/docs/images/bsdf/bsdf_overview.jpg
        :width: 100%
        :align: center

    Schematic overview of the most important surface scattering models in Mitsuba 2.
    The arrows indicate possible outcomes of an interaction with a surface that has
    the respective model applied to it.

Surface scattering models describe the manner in which light interacts
with surfaces in the scene. They conveniently summarize the mesoscopic
scattering processes that take place within the material and
cause it to look the way it does.
This represents one central component of the material system in Mitsuba 2---another
part of the renderer concerns itself with what happens
*in between* surface interactions. For more information on this aspect,
please refer to the sections regarding participating media.
This section presents an overview of all surface scattering models that are
supported, along with their parameters.

To achieve realistic results, Mitsuba 2 comes with a library of general-purpose
surface scattering models such as glass, metal, or plastic.
Some model plugins can also act as modifiers that are applied on top of one or
more scattering models.

Throughout the documentation and within the scene description
language, the word BSDF is used synonymously with the term *surface
scattering model*. This is an abbreviation for *Bidirectional
Scattering Distribution Function*, a more precise technical
term.

In Mitsuba 2, BSDFs are assigned to *shapes*, which describe the visible surfaces in
the scene. In the scene description language, this assignment can
either be performed by nesting BSDFs within shapes, or they can
be named and then later referenced by their name.
The following fragment shows an example of both kinds of usages:

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Creating a named BSDF for later use -->
        <bsdf type=".. BSDF type .." id="my_named_material">
            <!-- BSDF parameters go here -->
        </bsdf>

        <shape type="sphere">
            <!-- Example of referencing a named material -->
            <ref id="my_named_material"/>
        </shape>

        <shape type="sphere">
            <!-- Example of instantiating an unnamed material -->
            <bsdf type=".. BSDF type ..">
                <!-- BSDF parameters go here -->
            </bsdf>
        </shape>
    </scene>

It is generally more economical to use named BSDFs when they
are used in several places, since this reduces the internal memory usage.

.. _bsdf-correctness:

Correctness considerations
--------------------------

A vital consideration when modeling a scene in a physically-based rendering
system is that the used materials do not violate physical properties, and
that their arrangement is meaningful. For instance, imagine having designed
an architectural interior scene that looks good except for a white desk that
seems a bit too dark. A closer inspection reveals that it uses a Lambertian
material with a diffuse reflectance of 0.9.

In many rendering systems, it would be feasible to increase the
reflectance value above 1.0 in such a situation. But in Mitsuba, even a
small surface that reflects a little more light than it receives will
likely break the available rendering algorithms, or cause them to produce otherwise
unpredictable results. In fact, the right solution in this case would be to switch to
a different the lighting setup that causes more illumination to be received by
the desk and then *reduce* the material's reflectance---after all, it is quite unlikely that
one could find a real-world desk that reflects 90% of all incident light.

As another example of the necessity for a meaningful material description, consider
the glass model illustrated in the figure below. Here, careful thinking
is needed to decompose the object into boundaries that mark index of
refraction-changes. If this is done incorrectly and a beam of light can
potentially pass through a sequence of incompatible index of refraction changes (e.g. 1.00 to 1.33
followed by 1.50 to 1.33), the output is undefined and will quite likely
even contain inaccuracies in parts of the scene that are far
away from the glass.

.. figtable::
    :label: fig-glass-explanation
    :caption: Some of the scattering models in Mitsuba need to know the indices of refraction on the exterior and
              interior-facing side of a surface. It is therefore important to decompose the mesh into meaningful
              separate surfaces corresponding to each index of refraction change. The example here shows such a
              decomposition for a water-filled Glass.
    :alt: Glass interfaces explanation

    .. figure:: ../../resources/data/docs/images/bsdf/glass_explanation.svg
        :alt: Glass interfaces explanation
        :width: 95%
        :align: center


.. _sec-phase:

Phase functions
===============

This section contains a description of all implemented medium scattering models,
which are also known as phase functions. These are very similar in principle to
surface scattering models (or BSDFs), and essentially describe where light
travels after hitting a particle within the medium. Currently, only the most
commonly used models for smoke, fog, and other homogeneous media are implemented.
.. _sec-emitters:

Emitters
========

    .. image:: ../../resources/data/docs/images/emitter/emitter_overview.jpg
        :width: 70%
        :align: center

    Schematic overview of the emitters in Mitsuba 2. The arrows indicate
    the directional distribution of light.

Mitsuba 2 supports a number of different emitters/light sources, which can be
classified into two main categories: emitters which are located somewhere within the scene, and emitters that surround the scene to simulate a distant environment.

Generally, light sources are specified as children of the ``<scene>`` element; for instance,
the following snippet instantiates a point light emitter that illuminates a sphere:

.. code-block:: xml

    <scene version="2.0.0">
        <emitter type="point">
            <spectrum name="intensity" value="1"/>
            <point name="position" x="0" y="0" z="-2"/>
        </emitter>

        <shape type="sphere"/>
    </scene>

An exception to this are area lights, which turn a geometric object into a light source.
These are specified as children of the corresponding ``<shape>`` element:

.. code-block:: xml

    <scene version="2.0.0">
        <shape type="sphere">
            <emitter type="area">
                <spectrum name="radiance" value="1"/>
            </emitter>
        </shape>
    </scene>
.. _sec-sensors:

Sensors
=======

In Mitsuba 2, *sensors*, along with a *film*, are responsible for recording
radiance measurements in some usable format.

In the XML scene description language, a sensor declaration looks as follows:

.. code-block:: xml

    <scene version=2.0.0>
        <!-- .. scene contents .. -->

        <sensor type=".. sensor type ..">
            <!-- .. sensor parameters .. -->

            <sampler type=".. sampler type ..">
                <!-- .. sampler parameters .. -->
            </samplers>

            <film type=".. film type ..">
                <!-- .. film parameters .. -->
            </film>
        </sensor>
    </scene>

In other words, the ``sensor`` declaration is a child element of the ``<scene>``
(the particular position in the scene file does not play a role). Nested within
the sensor declaration is a sampler instance (see :ref:`Samplers <sec-samplers>`)
and a film instance (see :ref:`Films <sec-films>`).

Sensors in Mitsuba 2 are *right-handed*. Any number of rotations and translations
can be applied to them without changing this property. By default, they are located
at the origin and oriented in such a way that in the rendered image, :math:`+X`
points left, :math:`+Y` points upwards, and :math:`+Z` points along the viewing
direction.
Left-handed sensors are also supported. To switch the handedness, flip any one
of the axes, e.g. by passing a scale transform like ``<scale x="-1"/>`` to the
sensor's :monosp:`to_world` parameter... _sec-textures:

Textures
========

The following section describes the available texture data sources. In Mitsuba 2,
textures are objects that can be attached to certain surface scattering model
parameters to introduce spatial variation. In the documentation, these are listed
as supporting the :paramtype:`texture` type. See the last sections about
:ref:`BSDFs <sec-bsdfs>` for many examples.

Textures take an (optional) ``<transform>`` called :paramtype:`to_uv` which can
be used to translate, scale, or rotate the lookup into the texture accordingly.

An example in XML looks the following:

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Create a BSDF that supports textured parameters -->
        <bsdf type=".. BSDF type .." id="my_textured_material">
            <texture type=".. texture type .." name=".. parameter name ..">
                <!-- .. Texture parameters go here .. -->

                <transform name="to_uv">
                    <!-- Scale texture by factor of 2 -->
                    <scale x="2" y="2"/>
                    <!-- Offset texture by [0.5, 1.0] -->
                    <translate x="0.5" y="1.0"/>
                </transform>
            </texture>

            <!-- .. Non-spatially varying BSDF parameters ..-->
        </bsdf>
    </scene>

Similar to BSDFs, named textures can alternatively defined at the top level of the scene
and later referenced. This is particularly useful if the same texture would be loaded
many times otherwise.

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Create a named texture at the top level -->
        <texture type=".. texture type .." id="my_named_texture">
            <!-- .. Texture parameters go here .. -->

            <transform name="to_uv">
                <!-- .. Transform parameters .. -->
            </transform>
        </texture>

        <!-- Create a BSDF that supports textured parameters -->
        <bsdf type=".. BSDF type ..">
            <!-- Example of referencing a named texture -->
            <ref id="my_named_texture" name=".. parameter name .."/>

            <!-- .. Non-spatially varying BSDF parameters ..-->
        </bsdf>
    </scene>



.. _sec-spectra:

Spectra
=======

This section describes the plugins behind spectral reflectance or emission used
in Mitsuba 2. On an implementation level, these behave very similarly to the
:ref:`texture plugins <sec-textures>` described earlier (but lacking their
spatially varying property) and can thus be used similarly as either BSDF or
emitter parameters:

.. code-block:: xml

    <scene version=2.0.0>
        <bsdf type=".. BSDF type ..">
            <!-- Explicitly add a uniform spectrum plugin -->
            <spectrum type=".. spectrum type .." name=".. parameter name ..">
                <!-- Spectrum parameters go here -->
            </spectrum>
        </bsdf>
    </scene>

In practice, it is however discouraged to instantiate plugins in this explicit way
and the XML scene description parser directly parses a number of common (shorter)
``<spectrum>`` and ``<rgb>`` tags See the corresponding section about the
:ref:`scene file format <sec-file-format>` for details.

The following two tables summarize which underlying plugins get instantiated
in each case, accounting for differences between reflectance and emission properties
and all different color modes. Each plugin is briefly summarized below.

.. figtable::
    :label: spectrum-reflectance-table-list
    :caption: Spectra used for reflectance (within BSDFs)
    :alt: Spectrum reflectance table

    .. list-table::
        :widths: 35 25 25 25
        :header-rows: 1

        * - XML description
          - Monochrome mode
          - RGB mode
          - Spectral mode
        * - ``<spectrum name=".." value="0.5"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`uniform <spectrum-uniform>`
        * - ``<spectrum name=".." value="400:0.1, 700:0.2"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<spectrum name=".." filename=".."/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<rgb name=".." value="0.5, 0.2, 0.5"/>``
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`srgb <spectrum-srgb>`
          - :ref:`srgb <spectrum-srgb>`

.. figtable::
    :label: spectrum-emission-table-list
    :caption: Spectra used for emission (within emitters)
    :alt: Spectrum emission table

    .. list-table::
        :widths: 35 25 25 25
        :header-rows: 1

        * - XML description
          - Monochrome mode
          - RGB mode
          - Spectral mode
        * - ``<spectrum name=".." value="0.5"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`d65 <spectrum-d65>`
        * - ``<spectrum name=".." value="400:0.1, 700:0.2"/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<spectrum name=".." filename=".."/>``
          - :ref:`uniform <spectrum-uniform>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`regular <spectrum-regular>`/:ref:`irregular <spectrum-irregular>`
        * - ``<rgb name=".." value="0.5, 0.2, 0.5"/>``
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
          - :ref:`srgb_d65 <spectrum-srgb_d65>`
.. _sec-integrators:

Integrators
===========

In Mitsuba 2, the different rendering techniques are collectively referred to as
*integrators*, since they perform integration over a high-dimensional space.
Each integrator represents a specific approach for solving the light transport
equation---usually favored in certain scenarios, but at the same time affected
by its own set of intrinsic limitations. Therefore, it is important to carefully
select an integrator based on user-specified accuracy requirements and
properties of the scene to be rendered.

In the XML description language, a single integrator is usually instantiated by
declaring it at the top level within the scene, e.g.

.. code-block:: xml

    <scene version=2.0.0>
        <!-- Instantiate a unidirectional path tracer,
             which renders paths up to a depth of 5 -->
        <integrator type="path">
            <integer name="max_depth" value="5"/>
        </integrator>

        <!-- Some geometry to be rendered -->
        <shape type="sphere">
            <bsdf type="diffuse"/>
        </shape>
    </scene>

This section gives an overview of the available choices along with their parameters.

Almost all integrators use the concept of *path depth*. Here, a path refers to
a chain of scattering events that starts at the light source and ends at the
camera. It is oten useful to limit the path depth when rendering scenes for
preview purposes, since this reduces the amount of computation that is necessary
per pixel. Furthermore, such renderings usually converge faster and therefore
need fewer samples per pixel. Then reference-quality is desired, one should always
leave the path depth unlimited.

The Cornell box renderings below demonstrate the visual effect of a maximum path
depth. As the paths are allowed to grow longer, the color saturation increases
due to multiple scattering interactions with the colored surfaces. At the same
time, the computation time increases.

.. subfigstart::
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_1.jpg
   :caption: max. depth = 1
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_2.jpg
   :caption: max. depth = 2
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_3.jpg
   :caption: max. depth = 3
.. subfigure:: ../../resources/data/docs/images/render/integrator_depth_inf.jpg
   :caption: max. depth = :math:`\infty`
.. subfigend::
   :width: 0.23
   :label: fig-integrators-depth

Mitsuba counts depths starting at 1, which corresponds to visible light sources
(i.e. a path that starts at the light source and ends at the camera without any
scattering interaction in between.) A depth-2 path (also known as "direct
illumination") includes a single scattering event like shown here:

.. image:: ../../resources/data/docs/images/integrator/path_explanation.jpg
    :width: 80%
    :align: center.. _sec-samplers:

Samplers
========

When rendering an image, Mitsuba 2 has to solve a high-dimensional integration problem that involves
the geometry, materials, lights, and sensors that make up the scene. Because of the mathematical
complexity of these integrals, it is generally impossible to solve them analytically — instead, they
are solved numerically by evaluating the function to be integrated at a large number of different
positions referred to as samples. Sample generators are an essential ingredient to this process:
they produce points in a (hypothetical) infinite dimensional hypercube :math:`[0, 1]^{\infty}`
that constitute the canonical representation of these samples.

To do its work, a rendering algorithm, or integrator, will send many queries to the sample
generator. Generally, it will request subsequent 1D or 2D components of this infinite-dimensional
*point* and map them into a more convenient space (for instance, positions on surfaces). This allows
it to construct light paths to eventually evaluate the flow of light through the scene.
.. _sec-films:

Films
=====

A film defines how conducted measurements are stored and converted into the final
output file that is written to disk at the end of the rendering process.

In the XML scene description language, a normal film configuration might look
as follows:

.. code-block:: xml

    <scene version=2.0.0>
        <!-- .. scene contents -->

        <sensor type=".. sensor type ..">
            <!-- .. sensor parameters .. -->

            <!-- Write to a high dynamic range EXR image -->
            <film type="hdrfilm">
                <!-- Specify the desired resolution (e.g. full HD) -->
                <integer name="width" value="1920"/>
                <integer name="height" value="1080"/>

                <!-- Use a Gaussian reconstruction filter. For details
                     on these, refor to the next subsection -->
                <rfilter type="gaussian"/>
            </film>
        </sensor>
    </scene>

The ``<film>`` plugin should be instantiated nested inside a ``<sensor>``
declaration. Note how the output filename is never specified---it is automatically
inferred from the scene filename and can be manually overridden by passing the
configuration parameter ``-o`` to the ``mitsuba`` executable when rendering
from the command line.
.. _sec-rfilters:

Reconstruction filters
======================

Image reconstruction filters are responsible for converting a series of radiance samples generated
jointly by the sampler and integrator into the final output image that will be written to disk at
the end of a rendering process. This section gives a brief overview of the reconstruction filters
that are available in Mitsuba. There is no universally superior filter, and the final choice depends
on a trade-off between sharpness, ringing, and aliasing, and computational efficiency.

Desireable properties of a reconstruction filter are that it sharply captures all of the details
that are displayable at the requested image resolution, while avoiding aliasing and ringing.
Aliasing is the incorrect leakage of high-frequency into low-frequency detail, and ringing denotes
oscillation artifacts near discontinuities, such as a light-shadow transiton.