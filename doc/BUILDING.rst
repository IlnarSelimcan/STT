.. _build-native-client:

Building Binaries
=================

This section describes how to build 🐸STT binaries.

It is strongly recommended that you always use our pre-built 🐸STT binaries (available with every `release <https://github.com/coqui-ai/STT/releases>`_) unless you have a reason to do build them yourself.

If you would still like to build the 🐸STT binaries yourself, you'll need the following pre-requisites downloaded and installed:

* `Bazel 3.1.0 <https://github.com/bazelbuild/bazel/releases/tag/3.1.0>`_
* `General TensorFlow r2.3 requirements <https://www.tensorflow.org/install/source#tested_build_configurations>`_
* `libsox <https://sourceforge.net/projects/sox/>`_

It is required to use our fork of TensorFlow since it includes fixes for common problems encountered when building the native client files.

If you'd like to build the language bindings or the decoder package, you'll also need:

* `SWIG >= 3.0.12 <http://www.swig.org/>`_.
  Unfortunately, NodeJS / ElectronJS after 10.x support on SWIG is a bit behind, and while there are pending patches proposed to upstream, it is not yet merged.
  The proper prebuilt patched version (covering linux, windows and macOS) of SWIG should get installed under `native_client/ <native_client/>`_ as soon as you build any bindings that requires it.

* `node-pre-gyp <https://github.com/mapbox/node-pre-gyp>`_ (for Node.JS bindings only)

For information on building on Windows, please refer to: :ref:`Windows Building <build-native-client-dotnet>`.

Dependencies
------------

If you follow these instructions, you should compile your own binaries of 🐸STT (built on TensorFlow using Bazel).

For more information on configuring TensorFlow, read the docs up to the end of `"Configure the Build" <https://www.tensorflow.org/install/source#configure_the_build>`_.

Checkout source code
^^^^^^^^^^^^^^^^^^^^

Clone 🐸STT source code (TensorFlow will come as a submdule):

.. code-block::

   git clone https://github.com/coqui-ai/STT.git STT
   cd STT
   git submodule sync tensorflow/
   git submodule update --init tensorflow/

Bazel: Download & Install
^^^^^^^^^^^^^^^^^^^^^^^^^

First, install Bazel 3.1.0 following the `Bazel installation documentation <https://docs.bazel.build/versions/3.1.0/install.html>`_.

TensorFlow: Configure with Bazel
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After you have installed the correct version of Bazel, configure TensorFlow:

.. code-block::

   cd tensorflow
   ./configure

Compile Coqui STT
-----------------

Compile ``libstt.so``
^^^^^^^^^^^^^^^^^^^^^

Within your TensorFlow directory, there should be a symbolic link to the 🐸STT ``native_client`` directory. If it is not present, create it with the follow command:

.. code-block::

   cd tensorflow
   ln -s ../native_client

You can now use Bazel to build the main 🐸STT library, ``libstt.so``. Add ``--config=cuda`` if you want a CUDA build.

.. code-block::

   bazel build --workspace_status_command="bash native_client/bazel_workspace_status_cmd.sh" --config=monolithic -c opt --copt=-O3 --copt="-D_GLIBCXX_USE_CXX11_ABI=0" --copt=-fvisibility=hidden //native_client:libstt.so

The generated binaries will be saved to ``bazel-bin/native_client/``.

.. _build-generate-scorer-package:

Compile ``generate_scorer_package``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Following the same setup as for ``libstt.so`` above, you can rebuild the ``generate_scorer_package`` binary by adding its target to the command line: ``//native_client:generate_scorer_package``.
Using the example from above you can build the library and that binary at the same time:

.. code-block::

   bazel build --workspace_status_command="bash native_client/bazel_workspace_status_cmd.sh" --config=monolithic -c opt --copt=-O3 --copt="-D_GLIBCXX_USE_CXX11_ABI=0" --copt=-fvisibility=hidden //native_client:libstt.so //native_client:generate_scorer_package

The generated binaries will be saved to ``bazel-bin/native_client/``.

Compile Language Bindings
^^^^^^^^^^^^^^^^^^^^^^^^^

Now, ``cd`` into the ``STT/native_client`` directory and use the ``Makefile`` to build all the language bindings (C++ client, Python package, Nodejs package, etc.).

.. code-block::

   cd ../STT/native_client
   make stt

Installing your own Binaries
----------------------------

After building, the library files and binary can optionally be installed to a system path for ease of development. This is also a required step for bindings generation.

.. code-block::

   PREFIX=/usr/local sudo make install

It is assumed that ``$PREFIX/lib`` is a valid library path, otherwise you may need to alter your environment.

Install Python bindings
^^^^^^^^^^^^^^^^^^^^^^^

Included are a set of generated Python bindings. After following the above build and installation instructions, these can be installed by executing the following commands (or equivalent on your system):

.. code-block::

   cd native_client/python
   make bindings
   pip install dist/stt-*

The API mirrors the C++ API and is demonstrated in `client.py <python/client.py>`_. Refer to `coqui-stt.h <coqui-stt.h>`_ for documentation.

Install NodeJS / ElectronJS bindings
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

After following the above build and installation instructions, the Node.JS bindings can be built:

.. code-block::

   cd native_client/javascript
   make build
   make npm-pack

This will create the package ``stt-VERSION.tgz`` in ``native_client/javascript``.

.. _build-ctcdecoder-package:

Install the CTC decoder package
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

To build the ``coqui_stt_ctcdecoder`` package, you'll need the general requirements listed above (in particular SWIG). The command below builds the bindings using eight (8) processes for compilation. Adjust the parameter accordingly for more or less parallelism.

.. code-block::

   cd native_client/ctcdecode
   make bindings NUM_PROCESSES=8
   pip install dist/*.whl


Building CTC Decoder for training on unsupported platforms
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We only support building CTC Decoder on x86-64 architectures. However, we offer some hints on building the CTC decoder on other architectures, and you might find some help in our `GitHub Discussions <https://github.com/coqui-ai/STT/discussions>`.

Feedback on improving this section or usage on other architectures is welcome.

First, you need to build SWIG from scratch. Given that `SWIG >= 3.0.12 <http://www.swig.org/>`_ does not include our patches please use https://github.com/lissyx/swig/tree/taskcluster for building SWIG from source.

You can supply your prebuild SWIG using ``SWIG_DIST_URL``

Moreover you may have to change ``PYTHON_PLATFORM_NAME`` corresponding to your platform.

.. code-block::

    # PowerPC (ppc64le)
    PYTHON_PLATFORM_NAME="--plat-name linux_ppc64le"

Complete build command:

.. code-block::

    SWIG_DIST_URL=[...] PYTHON_PLATFORM_NAME=[...] make bindings
    pip install dist/*.whl

Cross-building
--------------

RPi3 ARMv7 and LePotato ARM64
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We do support cross-compilation. Please refer to our ``coqui-ai/tensorflow`` fork, where we define the following ``--config`` flags:

* ``--config=rpi3`` and ``--config=rpi3_opt`` for Raspbian / ARMv7
* ``--config=rpi3-armv8`` and ``--config=rpi3-armv8_opt`` for ARMBian / ARM64

So your command line for ``RPi3`` and ``ARMv7`` should look like:

.. code-block::

   bazel build --workspace_status_command="bash native_client/bazel_workspace_status_cmd.sh" --config=monolithic --config=rpi3 --config=rpi3_opt -c opt --copt=-O3 --copt=-fvisibility=hidden //native_client:libstt.so

And your command line for ``LePotato`` and ``ARM64`` should look like:

.. code-block::

   bazel build --workspace_status_command="bash native_client/bazel_workspace_status_cmd.sh" --config=monolithic --config=rpi3-armv8 --config=rpi3-armv8_opt -c opt --copt=-O3 --copt=-fvisibility=hidden //native_client:libstt.so

While we test only on RPi3 Raspbian Buster and LePotato ARMBian Buster, anything compatible with ``armv7-a cortex-a53`` or ``armv8-a cortex-a53`` should be fine.

The ``stt`` binary can also be cross-built, with ``TARGET=rpi3`` or ``TARGET=rpi3-armv8``. This might require you to setup a system tree using the tool ``multistrap`` and the multitrap configuration files: ``native_client/multistrap_armbian64_buster.conf`` and ``native_client/multistrap_raspbian_buster.conf``.
The path of the system tree can be overridden from the default values defined in ``definitions.mk`` through the ``RASPBIAN`` ``make`` variable.

.. code-block::

   cd ../STT/native_client
   make TARGET=<system> stt

Android devices support
-----------------------

We have support for Android relying on TensorFlow Lite, with Java and JNI bindinds. For more details on how to experiment with those, please refer to the section below.

Please refer to TensorFlow documentation on how to setup the environment to build for Android (SDK and NDK required).

Using the library from Android project
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Due to the discontinuation of Bintray JCenter we do not have pre-built Android packages published for now. We are working to move to Maven Central and will update this section when it's available.

.. We provide uptodate and tested ``libstt`` usable as an ``AAR`` package,
  for Android versions starting with 7.0 to 11.0. The package is published on
  `JCenter <https://bintray.com/coqui/ai.coqui.stt/libstt>`_,
  and the ``JCenter`` repository should be available by default in any Android
  project.  Please make sure your project is setup to pull from this repository.
  You can then include the library by just adding this line to your
  ``gradle.build``, adjusting ``VERSION`` to  the version you need:
  
  .. code-block::
  
     implementation 'stt.coqui.ai:libstt:VERSION@aar'

Building ``libstt.so`` for Android
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

You can build the ``libstt.so`` using (ARMv7):

.. code-block::

   bazel build --workspace_status_command="bash native_client/bazel_workspace_status_cmd.sh" --config=monolithic --config=android --config=android_arm --define=runtime=tflite --action_env ANDROID_NDK_API_LEVEL=21 --cxxopt=-std=c++14 --copt=-D_GLIBCXX_USE_C99 //native_client:libstt.so

Or (ARM64):

.. code-block::

   bazel build --workspace_status_command="bash native_client/bazel_workspace_status_cmd.sh" --config=monolithic --config=android --config=android_arm64 --define=runtime=tflite --action_env ANDROID_NDK_API_LEVEL=21 --cxxopt=-std=c++14 --copt=-D_GLIBCXX_USE_C99 //native_client:libstt.so

Building ``libstt.aar``
^^^^^^^^^^^^^^^^^^^^^^^

In the unlikely event you have to rebuild the JNI bindings, source code is
available under the ``libstt`` subdirectory.  Building depends on shared
object: please ensure to place ``libstt.so`` into the
``libstt/libs/{arm64-v8a,armeabi-v7a,x86_64}/`` matching subdirectories.

Building the bindings is managed by ``gradle`` and should be limited to issuing
``./gradlew libstt:build``, producing an ``AAR`` package in
``./libstt/build/outputs/aar/``.

Please note that you might have to copy the file to a local Maven repository
and adapt file naming (when missing, the error message should states what
filename it expects and where).

Building C++ ``stt`` binary
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Building the ``stt`` binary will happen through ``ndk-build`` (ARMv7):

.. code-block::

   cd ../STT/native_client
   $ANDROID_NDK_HOME/ndk-build APP_PLATFORM=android-21 APP_BUILD_SCRIPT=$(pwd)/Android.mk NDK_PROJECT_PATH=$(pwd) APP_STL=c++_shared TFDIR=$(pwd)/../tensorflow/ TARGET_ARCH_ABI=armeabi-v7a

And (ARM64):

.. code-block::

   cd ../STT/native_client
   $ANDROID_NDK_HOME/ndk-build APP_PLATFORM=android-21 APP_BUILD_SCRIPT=$(pwd)/Android.mk NDK_PROJECT_PATH=$(pwd) APP_STL=c++_shared TFDIR=$(pwd)/../tensorflow/ TARGET_ARCH_ABI=arm64-v8a

Android demo APK
^^^^^^^^^^^^^^^^

Provided is a very simple Android demo app that allows you to test the library.
You can build it with ``make apk`` and install the resulting APK file. Please
refer to Gradle documentation for more details.

The ``APK`` should be produced in ``/app/build/outputs/apk/``. This demo app might
require external storage permissions. You can then push models files to your
device, set the path to the file in the UI and try to run on an audio file.
When running, it should first play the audio file and then run the decoding. At
the end of the decoding, you should be presented with the decoded text as well
as time elapsed to decode in miliseconds.

This application is very limited on purpose, and is only here as a very basic
demo of one usage of the application. For example, it's only able to read PCM
mono 16kHz 16-bits file and it might fail on some WAVE file that are not
following exactly the specification.

Running ``stt`` via adb
^^^^^^^^^^^^^^^^^^^^^^^

You should use ``adb push`` to send data to device, please refer to Android
documentation on how to use that.

Please push 🐸STT data to ``/sdcard/STT/``\ , including:


* ``output_graph.tflite`` which is the TF Lite model
* External scorer file (available from one of our releases), if you want to use
  the scorer; please be aware that too big scorer will make the device run out
  of memory

Then, push binaries from ``native_client.tar.xz`` to ``/data/local/tmp/ds``\ :

* ``stt``
* ``libstt.so``
* ``libc++_shared.so``

You should then be able to run as usual, using a shell from ``adb shell``\ :

.. code-block::

   user@device$ cd /data/local/tmp/ds/
   user@device$ LD_LIBRARY_PATH=$(pwd)/ ./stt [...]

Please note that Android linker does not support ``rpath`` so you have to set
``LD_LIBRARY_PATH``. Properly wrapped / packaged bindings does embed the library
at a place the linker knows where to search, so Android apps will be fine.

Delegation API
^^^^^^^^^^^^^^

TensorFlow Lite supports Delegate API to offload some computation from the main
CPU. Please refer to `TensorFlow's documentation
<https://www.tensorflow.org/lite/performance/delegates>`_ for details.

To ease with experimentations, we have enabled some of those delegations on our
Android builds: * GPU, to leverage OpenGL capabilities * NNAPI, the Android API
to leverage GPU / DSP / NPU * Hexagon, the Qualcomm-specific DSP

This is highly experimental:

* Requires passing environment variable ``STT_TFLITE_DELEGATE`` with values of
  ``gpu``, ``nnapi`` or ``hexagon`` (only one at a time)
* Might require exported model changes (some Op might not be supported)
* We can't guarantee it will work, nor it will be faster than default
  implementation

Feedback on improving this is welcome: how it could be exposed in the API, how
much performance gains do you get in your applications, how you had to change
the model to make it work with a delegate, etc.

See :ref:`the support / contact details <support>`
