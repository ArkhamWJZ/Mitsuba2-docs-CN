.. _sec-compiling:

Compiling the system
====================

在继续之前，请却表你已经完成阅读并遵循了以下两个章节 :ref:`cloning Mitsuba 2 and its dependencies <sec-cloning>` 
和 :ref:`choosing desired variants <sec-variants>`。

从头开始编译 Mitsuba 2需要最新版的 CMake(至少是 **3.9.0**)和 Python（至少是 **3.6**）。
下面为每种操作系统提供了平台特定的依赖项和编译说明。本节末尾介绍的是基于 GPU 后端需要的一些额外步骤。

Linux
-----

Linux 下的构建过程需要几个外部依赖项，这些依赖项可以使用系统提供的包管理器轻松安装（例如，Ubuntu 下的 :monosp:`apt-get`）。

请注意，最近的 Linux 发行版包括两个不同的编译器，它们都可以用于 C++ 软件开发。
`GCC <https://gcc.gnu.org>`_ 通常是默认设置，而 `Clang <https://clang.llvm.org>`_ 是可以选择安装的。
在本项目的开发过程中，我们遇到了很多关于 GCC 的问题(mis-compilations, compiler errors, segmentation faults)，
强烈建议你使用 Clang 进行编译。

要获取所有依赖项和 Clang，请在 Ubuntu 上输入以下命令：

.. code-block:: bash

    # Install recent versions build tools, including Clang and libc++ (Clang's C++ library)
    sudo apt install -y clang-9 libc++-9-dev libc++abi-9-dev cmake ninja-build

    # Install libraries for image I/O and the graphical user interface
    sudo apt install -y libz-dev libpng-dev libjpeg-dev libxrandr-dev libxinerama-dev libxcursor-dev

    # Install required Python packages
    sudo apt install -y python3-dev python3-distutils python3-setuptools

运行包含的测试套件或生成 HTML 文档需要的其他包（请参阅 :ref:`Developer guide <sec-d evguide>`）。
如果你对这些内容感兴趣，输入以下命令：

.. code-block:: bash

    # For running tests
    sudo apt install -y python3-pytest python3-pytest-xdist python3-numpy

    # For generating the documentation
    sudo apt install -y python3-sphinx python3-guzzle-sphinx-theme python3-sphinxcontrib.bibtex

接下来，确保导入了两个环境变量 :monosp:`CC` 和 :monosp:`CXX` 。你可以在使用 CMake 之前手动运行这两个命令---或者
更进一步，将它们添加到你的 :monosp:`~/.bashrc` 文件中。这可以保证 CMake 总是使用正确的编译器。


.. code-block:: bash

    export CC=clang-9
    export CXX=clang++-9

如果你安装了另一个版本的 Clang，版本后缀需要相应做出调整。随后，在 :monosp:`mitsuba2` 根目录中运行以下命令，编译就可以
很简单的完成了。

.. code-block:: bash

    # Create a directory where build products are stored
    mkdir build
    cd build
    cmake -GNinja ..
    ninja


Tested version
^^^^^^^^^^^^^^

上述的过程可以适用于许多不同风格的 Linux（只需对包管理器和包名进行轻微的调整）。我们主要使用的是下面列出的软件环境，
在这种情况下，我们的说明应该可以正常工作并且不需要修改。

* Ubuntu 19.10
* clang 9.0.0-2 (tags/RELEASE_900/final)
* cmake 3.13.4
* ninja 1.9.0
* python 3.7.5

Windows
-------

Windows 平台上, 需要最新版的 `Visual Studio 2019
<https://visualstudio.microsoft.com/vs/>`_ 。 还有一些其他工具，例如git、
CMake、还有 Python (e.g. 通过 `Miniconda 3
<https://docs.conda.io/en/latest/miniconda.html>`_) 等需要手动安装。 Mitsuba 的构建系统
 *需要接入*  Python >= 3.6 ，即使你并不打算使用 Mitsuba 的 Python 接口。

在 `mitsuba2` 的根目录下，构建过程可以如此配置：

.. code-block:: bash

    # To be safe, explicitly ask for the 64 bit version of Visual Studio
    cmake -G "Visual Studio 16 2019" -A x64


在这之后，打开生成的 ``mitsuba.sln`` 文件，接下来在 Visual Studio 像往常一样进行构建即可。
你可能还需要将构建模式设置为 *Release*。

运行包含的测试套件或生成 HTML 文档需要的其他包（请参阅 :ref:`Developer guide <sec-d evguide>`）。
如果你对这些内容感兴趣，输入以下命令：

.. code-block:: bash

    conda install pytest numpy sphinx


Tested version
^^^^^^^^^^^^^^
* Windows 10
* Visual Studio 2019 (Community Edition) Version 16.4.5
* cmake 3.16.4 (64bit)
* git 2.25.1 (64bit)
* Miniconda3 4.7.12.1 (64bit)


macOS
-----

macOS 平台上，你需要安装 Xcode，CMake 和 `Ninja <https://ninja-build.org/>`_。
除此之外，还需要运行一次 Xcode 命令行工具。

.. code-block:: bash

    xcode-select --install

注意，macOS 中默认 Python 版本与 Mitsuba 2不兼容，因此你需要安装更新的版本 Python（至少3.6，通过 `Miniconda 3 <https://docs.conda.io/en/latest/miniconda.html>`_ 或 `Homebrew <https://brew.sh/>`_)。

随后，在 :monosp:`mitsuba2` 根目录中运行以下命令，编译就可以 很简单的完成了：

.. code-block:: bash

    mkdir build
    cd build
    cmake -GNinja ..
    ninja


Tested version
^^^^^^^^^^^^^^
* macOS Catalina 10.15.2
* Xcode 11.3.1
* cmake 3.16.4
* Python 3.7.3


Running Mitsuba
---------------

一旦 Mitsuba 的编译完成之后， 运行 ``setpath.sh/bat`` 脚本以配置环境变量(``PATH/LD_LIBRARY_PATH/PYTHONPATH``) that are required
才能运行 Mitsuba。

.. code-block:: bash

    # On Linux / Mac OS
    source setpath.sh

    # On Windows
    C:/.../mitsuba2> setpath

然后就可以顺利通过命令行输入来使用 Mitsuba 来渲染场景了。

.. code-block:: bash

    mitsuba scene.xml

``scene.xml`` 是一个 Mitsuba 场景文件。或者有，

.. code-block:: bash

    mitsuba -m scalar_spectral_polarized scene.xml

这是使用之前在 :monosp:`mitsuba.conf` 启用的指定变体模式进行渲染，调用 ``mitsuba --help`` 以
打印关于命令行选项的额外信息详细查看。


GPU variants
------------

运行在 GPU 上的 Mitsuba 变体（例如， :monosp:`gpu_rgb`,
:monosp:`gpu_autodiff_spectral` 等）都另外依赖于 `NVIDIA CUDA
Toolkit <https://developer.nvidia.com/cuda-downloads>`_ 和 `NVIDIA OptiX
<https://developer.nvidia.com/designworks/optix/download>`_。
需要手动安装 CUDA，而 OptiX 7 自带了最新的 GPU 驱动。如果框架无法编译 Mitsuba 
的 GPU 变体，请确保拥有最新的 GPU 驱动程序。

我们测试过的 CUDA 版本包括 10.0，10.1，和 10.2。 当前只有 OptiX 7 是被支持的。

.. warning::

    目前，基于 GPU 渲染 和 可微渲染都不支持在 macOS 运行，遗憾的是这一点在未来也不太可能改变。
    几年前，Apple 己经将 NVIDIA 显卡（包含其 API，如 Mitsuba 所依赖的 CUDA）从 Mac 生态中驱逐出去了。
    如果你对这一情况不满意，请向 Apple 发声。

如果 CMake 未能自动找到你安装的 CUDA(例如，目录不在 `PATH` 中)，你需要设置环境变量 `CUDACXX` 或者将 CMake 缓存
条目 `CMAKE_CUDA_COMPILER` 的全路径设置到编译器，例如：

.. code-block:: bash

    # Environment variable
    export CUDACXX=/usr/local/cuda/bin/nvcc

    # or

    # As part of the CMake process
    cmake .. -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc

默认情况下， Mitsuba 能够解析 OptiX API 本身，因此不依赖于 ``optix.h`` 头文件。
如果开发者想要实验 OptiX API 中尚未添加到框架的部分的话，可通过 CMake 
标志 ``MTS_USE_OPTIX_HEADERS`` 来关闭此功能，。


Embree
------

Mitsuba 的 ``scalar`` 和 ``packet`` 后端可以选择使用 Intel 的 Embree
库负责计算光线追踪部分，而不是使用 Mitsuba 2 中内置的 kd-tree。
调用 CMake 时通过 ``-DMTS_ENABLE_EMBREE=1`` 参数或者在 ``cmake-gui`` 或 ``ccmake``  
这样的 CMake 可视化工具中，反转这一参数的值，来实现后端计算库的调整。
Embree 速度往往更快，但会缺乏一些功能，例如支持双精度光线相交等。

