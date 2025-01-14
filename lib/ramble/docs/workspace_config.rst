.. Copyright 2022-2023 Google LLC

   Licensed under the Apache License, Version 2.0 <LICENSE-APACHE or
   https://www.apache.org/licenses/LICENSE-2.0> or the MIT license
   <LICENSE-MIT or https://opensource.org/licenses/MIT>, at your
   option. This file may not be copied, modified, or distributed
   except according to those terms.

============================
Workspace Configuration File
============================

Ramble workspaces are controlled through their configuration files. Each
workspace has a configuration file stored at ``$workspace/configs/ramble.yaml``.

This document will describe the syntax for writing a workspace configuration file.

Within the ``ramble.yaml`` file, there are two top level dictionairies.

.. code-block:: console

   ramble:
     ...
   spack:
     ...

Each of these dictionaires is used to control different aspects of the Ramble
workspace.

------------------
Ramble Dictionary:
------------------

The ramble dictionary is used to control the experiments a workspace is
responsible for configuring, executing, analyzing, and archiving.

.. code-block:: yaml

    ramble:
      mpi:
        command: mpirun
        args:
        - '-n'
        - '{n_ranks}'
      batch:
        submit: '{execute_experiment}'
      applications:
        hostname:
          workloads:
            serial:
              experiments:
                test_exp:
                  variables:
                    n_ranks: '1'
                    n_nodes: '1'

Within a ramble configuration file, configuration scopes for an experiment
include, ``application``, ``workload``, and ``experiment``. They are denoted by
these words in the configuration file. The name ``hostname`` name of the ramble
application (as seen by ``ramble list``), while the name ``serial`` is the name of the
workload (as seen by ``ramble info hostname``).

The name ``test_exp`` is user defined, and will be explained in :ref:`experiment-names`.

The name ``variables`` defines arbitrary variables, and will be explained in
:ref:`variable-dictionaries`.

.. _experiment-names:

^^^^^^^^^^^^^^^^^
Experiment Names:
^^^^^^^^^^^^^^^^^

While the names of applications and workloads are defined by the application
definition file, experiment names are more arbitrary. Experiment names are
string, and can take variables for expansion.

.. code-block:: yaml

    ramble:
      applications:
        hostname:
          workloads:
            serial:
              experiments:
                test_{n_ranks}_{n_nodes}:
                  variables:
                    n_ranks: '1'
                    n_nodes: '1'

In the above example, the experiment name would be: ``test_1_1`` when it is created.

**NOTE:** Each experiment has a namespace that follows this pattern:
``application.workload.experiment``. Every experiment needs a unique namespace,
or ramble will throw an error.

.. _variable-dictionaries:

^^^^^^^^^^^^^^^^^^^^^^
Variable Dictionaries:
^^^^^^^^^^^^^^^^^^^^^^

Within a variable dictionary, arbitrary variables can be defined. Defined
variables apply to all experiments within their scope.

These variables can be referred to within the YAML file, or template files
using python keyword ( ``{var_name}`` ) syntax to perform variable expansion.
This syntax allows basic math operations ( ``+``, ``-``, ``/``, ``*``, and
``**`` ) to evaluate math expressions using variable definitions.

If a variable is defined within multiple dictionaries, values defined closer to
individual experiments take precendence.

.. code-block:: yaml

    ramble:
      ...
      variables:
        processes_per_node: '16'
        n_ranks: '{n_nodes}*{processes_per_node}'
      applications:
        hostname:
          variables:
            n_threads: '1'
          workloads:
            serial:
              variables:
                n_nodes: '1'
              experiments:
                test_exp:
                  variables:
                    n_ranks: '1'

In this example, ``n_ranks`` will take a value of ``1`` within the ``test_exp``
experiment. This experiment will also include definitions for
``processes_per_node``, ``n_nodes``, and ``n_threads``.


^^^^^^^^^^^^^^
List Variables:
^^^^^^^^^^^^^^
Variables can be defined as a list of values as well (again, following the same
math and variable expansion syntax as defined above).

.. code-block:: yaml

    ramble:
      ...
      variables:
        processes_per_node: '16'
        n_ranks: '{n_nodes}*{processes_per_node}'
      applications:
        hostname:
          variables:
            n_threads: '1'
          workloads:
            serial:
              variables:
                n_nodes: ['1', '2', '3', '4']
              experiments:
                test_exp_{n_nodes}:
                  variables:
                    n_ranks: '1'

There are two noteable aspects of this config file are:
1. ``n_nodes`` is a list of values
2. The experiment name refernces variable values.

All lists defined within any experiment namespace are required to be the same
length. They are zipped together, and iterated over to generate unique experiments.

^^^^^^^^^^^^^^^^^^
Variable Matrices:
^^^^^^^^^^^^^^^^^^

In addition to allowing variables, Ramble's config file has a special syntax for define variable matrices.

Matrices consume list variables, and generate a matrix of variables with it.
Each independent matrix performs the cross product of any list variables it
consumes.

.. code-block:: yaml

    ramble:
      ...
      variables:
        n_ranks: '{n_nodes}*{processes_per_node}'
      applications:
        hostname:
          variables:
            n_threads: '1'
          workloads:
            serial:
              variables:
                processes_per_node: ['16', '32']
                n_nodes: ['1', '2', '3', '4']
              experiments:
                test_exp_{n_nodes}_{processes_per_node}:
                  variables:
                    n_ranks: '1'
                  matrix:
                  - processes_per_node

In the above example, the ``processes_per_node`` variable is consumed as part
of a matrix. The result is a matrix of shape 1x2. After this matrix is
consumed, it will be crossed with the zipped vectors (creating 8 unique experiments).

Mulitple matrices are allowed to be defined:

.. code-block:: yaml
   :linenos:

    ramble:
      ...
      variables:
        n_ranks: '{n_nodes}*{processes_per_node}'
      applications:
        hostname:
          variables:
            n_threads: '1'
          workloads:
            serial:
              variables:
                processes_per_node: ['16', '32']
                partition: ['part1', 'part2']
                n_nodes: ['1', '2', '3', '4']
              experiments:
                test_exp_{n_nodes}_{processes_per_node}:
                  variables:
                    n_ranks: '1'
                  matrices:
                  - - processes_per_node
                    - partition
                  - - n_nodes

The result of this is that two matrices are created. The first is a 2x2 matrix,
while the second is a 1x4 matrix. All matrices are required to have the same
number of elements, as they are flattened and zipped together. In this case,
there would be 4 experiments, each defined by a unique
``(processes_per_node, partition, n_nodes)`` tuple.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Cross Experiment Variable References:
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Variables can be defined to pull the value of a variable out of a different
experiment. This is particularly useful when an experiment needs the path to
something ramble automatically generates in a different experiment.

.. code-block:: yaml

    ramble:
      ...
      variables:
        processes_per_node: '16'
        n_ranks: '{n_nodes}*{processes_per_node}'
      applications:
        hostname:
          variables:
            n_threads: '1'
          workloads:
            serial:
              variables:
                n_nodes: '1'
              experiments:
                test_exp1:
                  variables:
                    n_ranks: '1'
                    real_value: 'exp1_value'
                test_exp2:
                  variables:
                    n_ranks: '1'
                    test_value: real_value in hostname.serial.test_exp1

In the above example, ``test_value`` extracts the value of ``real_value`` as
defined in the experiment ``hostname.serial.test_exp1``. When evaluated, this
will set ``test_value`` to ``'exp1_value'``.

^^^^^^^^^^^^^^^^^^^^^^
Controlling Internals:
^^^^^^^^^^^^^^^^^^^^^^

Within a workspace config, an internals dictionary can be used to control
several internal aspects of the application, workload, and experiment.

An internals dictionary can be defined anywhere a variables dictionary can be
defined (i.e. within a workspace, a specific application, a specific workload,
or a specific experiment). This section will describe the features available
within the internals dictionary.

"""""""""""""""""""
Custom Executables:
"""""""""""""""""""

Custom executables can be created within the internals dictionary. Below is an
example, showing how to create a ``lscpu`` executable at the application level.

.. code-block:: yaml

    ramble:
      applications:
        hostname:
          internals:
            custom_executables:
              lscpu:
                template:
                - 'lscpu'
                use_mpi: false
                redirect: '{log_file}'
         ...

The above example creates a custom executable, named ``lscpu`` that will inject
the command ``lscpu`` into the command for an experiment when it is used. It is
important to note that this only creates the executable, and does not use it.


"""""""""""""""""""""""""""""
Controlling Executable Order:
"""""""""""""""""""""""""""""

The internals dictionary allows the ability to control the order pre-defined
executables (or custom executables) are pieced together to build an experiment.

.. code-block:: yaml

   ramble:
     applications:
       hostname:
         internals:
           custom_executables:
             lscpu:
               template:
               - 'lscpu'
               use_mpi: false
               redirect: '{log_file}'
           executables:
           - serial
           - builtin::env_vars
           - lscpu

The above example builds off of the custom executable example, and shows how
one can control the order of the executables in the ``{command}`` expansion.

The default for the hostname application is ``[builtin::env_vars,
serial/parallel]`` but this changes the order and injects ``lscpu`` into the
expansion.

^^^^^^^^^^^^^^^^^^^
Reserved Variables:
^^^^^^^^^^^^^^^^^^^

There are several reserved, auto-generated, and required variables for Ramble
to function properly. This section will describe them.

"""""""""""""""""""
Required Variables:
"""""""""""""""""""

Ramble requires the following variables to be defined:

* ``n_ranks`` - Defines the number of MPI ranks to use. If not explicitly set,
  is defined as: ``{processes_per_node}*{n_nodes}``
* ``n_nodes`` - Defines the number of machines needed for the experiment. If
  not explicitly set, is defined as:
  ``ceiling({n_ranks}/{processes_per_node})``
* ``processes_per_node`` - Defines how many ranks should be on each node. If
  not explicitly set, is defined as: ``ceiling({n_ranks}/{n_nodes})``

""""""""""""""""""""
Generated Variables:
""""""""""""""""""""

Ramble automatically generates definitions for the following varialbes:

* ``application_name`` - Set to the name of the application
* ``workload_name`` - Set to the name of the workload within the application
* ``experiment_name`` - Set to the name of the experiment
* ``spec_name`` - By default defined as ``{application_name}``. Can be
  overriden to control the spack definition to use.
* ``application_run_dir`` - Absolute path to
  ``$workspace_root/experiments/{application_name}``
* ``workload_run_dir`` - Absolute path to
  ``$workspace_root/experiments/{application_name}/{workload_name}``
* ``experiment_run_dir`` - Absolute path to
  ``$workspace_root/experiments/{application_name}/{workload_name}/{experiment_name}``
* ``application_input_dir`` - Absolute path to
  ``$workspace_root/inputs/{application_name}``
* ``workload_input_dir`` - Absolute path to
  ``$workspace_root/inputs/{application_name}/{workload_name}``
* ``spack_env`` - Absolute path to
  ``$workspace_root/software/{spec_name}.{workload_name}``
* ``log_dir`` - Absolute path to ``$workspace_root/logs``
* ``log_file`` - Absolute path to
  ``{experiment_run_dir}/{experiment_name}.out``
* ``<input_name>`` - Applications that have input files have variables defined
  that contain the absolute path to:
  ``$workspace_root/inputs/{application_name}/{workload_name}/<input_name>``
  where ``<input_name>`` is the name as defined in the ``input_file``
  directive.
* ``<template_name>`` - Any files with the ``.tpl`` extension in
  ``$workspace_root/configs`` have a variable generated that resolves to the
  absolute path to: ``{experiment_run_dir}/<template_name>`` where
  ``<template_name>`` is the filename of the template, without the extension.
* ``command`` - Set to all of the commands needed to perform an experiment.
* ``spack_setup`` - Set to the commands needed to load a spack environment for
  an experiment. Set to an empty string for non-spack applications
* ``mpi_command`` - By default, set to the contents of ``ramble:mpi```
* ``batch_submit`` - By default, set to the contents of ``ramble:batch:submit``

"""""""""""""""""""""""""""""""""""
Spack Specific Generated Variables:
"""""""""""""""""""""""""""""""""""
When using spack applications, Ramble also geneates the following variables:

* ``<software_spec_name>`` - Set to the equivalent of ``spack location -i
  <spec>`` for packages defined in a ramble ``spec_name`` package set.
  ``<software_spec_name>`` is set to the name of the package (one level lower
  than ramble's ``spec_name``).

-----------------
Spack Dictionary:
-----------------

Within a ramble.yaml file, the ``spack:`` dictionary controlls the software
stack installation that ramble performs.

Below is an annotated example of the spack dictionary.

.. code-block:: yaml

    spack:
      compilers:
        gcc9: # Abstract name to refer to this compiler
          base: gcc # Spack packge name
          version: 9.3.0 # Spack package version
          target: x86_64 # Spack target option
      mpi_libraries:
        impi2018: # Abstract name to refer to this MPI
          base: intel-mpi
          version: 2018.4.274
          target: x86_64
      applications:
        gromacs: # Ramble's spec_name variable
          gromacs: # application.py named software_spec, name of Ramble spec object
            base: gromacs # Spack package name
            version: 2022.4 # Spack package version
            compiler: gcc9 # Ramble compiler name
            mpi: impi2018 # Ramble MPI name

Application definition files can define one or more ``software_spec``
directives, which are packages the application might need to run properly. Some
are marked as required, and others might not be.

Multiple compilers and MPI libraries can be defined, even if they are not used.

^^^^^^^^^^^^^^^^^^^
Ramble Spec Format:
^^^^^^^^^^^^^^^^^^^

When writing Spack spec information in Ramble configuration files, the format
is as follows:

.. code-block:: yaml

   <software_spec:name>:
     base: # Takes the Spack package name
     version: # Takes the version, which would be passed in with @
     compiler: # Takes the name of the ramble spec object to use
               # to compile this package

     variants: # Takes any variant strings the package should be built with
     mpi: # Takes the name of the Ramble spec object to use for an MPI dependency
     arch: # Takes the input to the Spack `arch` option
     target: # Takes the input to the Spack `target` option
     dependencies: # YAML List containing Ramble spec object names this
                   # package depends on

Not all of the options are required, but generally a spec object should contain
at least ``base``, and ``version``.

--------------------------------------------
Controlling MPI Libraries and Batch Systems:
--------------------------------------------

Some workspaces might be configured with the goal of exploring the performance
of different MPI libraries (e.g. MPICH vs. Open MPI), or of performing the same
experiment in multiple batch schedulers (e.g. SLURM, PBS Pro, and Flux).

This section will show how to perform these experiments within a workspace
configuration file.


^^^^^^^^^^^^^^^^^^^^
MPI Command Control:
^^^^^^^^^^^^^^^^^^^^

When writing a ramble configuration file to perform the same experiment with
different MPI libraries, the MPI section within the Ramble dictionary is
insufficient for changing the flags used based on the MPI library used.

However, Ramble's variable definitions can be used to control this on a
per-experiment basis.

Below is an example of running a Gromacs experiment in both MPICH and OpenMPI:

.. code-block:: yaml

    ramble:
      mpi:
        command: ''
        args: []
      batch:
        submit: '{execute_experiment}'
      variables:
        mpi_command:
        - 'mpirun -n {n_ranks} -ppn {processes_per_node} ' # MPICH
        - 'mpirun -n {n_ranks} -nperhost {processes_per_node} ' # OpenMPI
      applications:
        gromacs:
          workloads:
            water_bare:
              experiments:
                '{spec_name}':
                  variables:
                    n_ranks: '1'
                    n_nodes: '1'
                    spec_name: ['gromacs-mpich', 'gromacs-ompi']
    spack:
      compilers:
        gcc9:
          base: gcc
          version: 9.3.0
          target: x86_64
      mpi_libraries:
        mpich:
          base: mpich
          version: 4.0.2
          target: x86_64
        ompi:
          base: openmpi
          version: 4.1.4
          target: x86_64
      applications:
        gromacs-ompi:
          gromacs:
            base: gromacs
            version: 2022.4
            compiler: gcc9
            mpi: ompi
        gromacs-mpich:
          gromacs:
            base: gromacs
            version: 2022.4
            compiler: gcc9
            mpi: mpich

In the above example, you can see how ``spec_name`` is used to test both an
OpenMPI and MPICH version of Gromacs. Additionally, the ``mpi_command``
variable is used to define how ``mpirun`` should look for each of the MPI
libraries.

Using the previously described Ramble vector syntax, this configuration file
will generate 2 experiments. Both ``spec_name`` and ``mpi_command`` will be
zipped together, giving each experiment a tuple of: ``(mpi_command,
spec_name)`` which allows us to pair a specific MPI command to the
corresponding Gromacs spec.


^^^^^^^^^^^^^^^^^^^^^
Batch System Control:
^^^^^^^^^^^^^^^^^^^^^

Similar to the previously describe MPI command control, experiments can use
different batch systems by overriding the ``batch_submit`` variable.

As in the MPI command example, the ``ramble:batch:submit`` definition is
insufficient for changing how each experiment is submitted to a batch system
(or even what batch system the experiment is submitted to).

Below is an example configuration file showing how the ``batch_submit``
variable can be used to submit the same experiment to multiple batch systems.

.. code-block:: yaml

    ramble:
      mpi:
        command: mpirun
        args:
        - '-n'
        - '{n_ranks}'
        - '-ppn'
        - '{processes_per_node}'
      batch:
        submit: ''
      variables:
        batch_system:
        - slurm
        - pbs
        batch_submit:
        - 'sbatch {execute_slurm}'
        - 'qsub {execute_pbs}'
      applications:
        gromacs:
          workloads:
            water_bare:
              experiments:
                '{batch_system}'
                  variables:
                    n_ranks: '1'
                    n_nodes: '1'
    spack:
      compilers:
        gcc9:
          base: gcc
          version: 9.3.0
          target: x86_64
      mpi_libraries:
        impi2018:
          base: intel-mpi
          version: 2018.4.274
          target: x86_64
      applications:
        gromacs:
          gromacs:
            base: gromacs
            version: 2022.4
            compiler: gcc9
            mpi: impi2018

The above example overrides the generated ``batch_submit`` variable to change
how different experiments are submitted. In this example, we submit the same
experiment to both SLURM and PBS.

Note that each of the two ``batch_submit`` commands submits a different
template. This means the workspace's configs directory should have two files:
``execute_slurm.tpl`` and ``execute_pbs.tpl`` which will be template submission
scripts to each of the batch systems.
