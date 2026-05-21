.. image:: https://raw.githubusercontent.com/giotto-ai/pyflagser/master/doc/images/Giotto_logo_RGB.svg
   :width: 590


=========
flagserpy
=========

``flagserpy`` is a python API for the flagser C++ library by Daniel Lütgehetmann which computes the homology of directed flag complexes, its forked from pyflagser which is no longer being actively maintained. Please check out the original `luetge/flagser <https://github.com/luetge/flagser>`_ GitHub repository for more information.

Project genesis
---------------

``pyflagser`` is the result of a collaborative effort between `L2F SA <https://www.l2f.ch/>`_, the `Laboratory for Topology and Neuroscience <https://www.epfl.ch/labs/hessbellwald-lab/>`_ at EPFL, and the `Institute of Reconfigurable & Embedded Digital Systems (REDS) <https://heig-vd.ch/en/research/reds>`_ of HEIG-VD. ``flagserpy`` is a continuation of this project by [Jason P. Smith (NTU)](https://jasonpsmith.github.io/) and the [Algebraic Topology and Networks in Biology](https://www.mpi-cbg.de/research/researchgroups/currentgroups/daniela-egas-santander/research-focus) of the Max Planck Institute of Molecular Cell Biology and Genetics.

Installation
------------

Dependencies
~~~~~~~~~~~~

``flagserpy`` requires:

- Python (>= 3.8)
- NumPy (>= 1.17.0)
- SciPy (>= 0.17.0)

User installation
~~~~~~~~~~~~~~~~~

If you already have a working installation of numpy and scipy, the easiest way to install flagserpy is using ``pip``   ::

    python -m pip install -U flagserpy

Documentation
-------------

API reference (stable release): https://docs-pyflagser.giotto.ai

Contributing
------------

We welcome new contributors of all experience levels. To learn more about making a contribution to ``flagserpy``, please see the `CONTRIBUTING.rst file.

Developer installation
~~~~~~~~~~~~~~~~~~~~~~

C++ dependencies:
'''''''''''''''''

-  C++14 compatible compiler
-  CMake >= 3.9

Source code
'''''''''''

You can check the latest sources with the command::

    git clone https://github.com/TopNetBio/flagserpy.git


To install:
'''''''''''

From the cloned repository's root directory, run

.. code-block:: bash

   python -m pip install -e ".[tests]"

This way, you can pull the library's latest changes and make them immediately available on your machine.

Testing
'''''''

After installation, you can launch the test suite from outside the source directory::

    pytest flagserpy


Changelog
---------

See the `RELEASE.rst file for a history of notable changes to flagserpy.

Important links
~~~~~~~~~~~~~~~

- Official source code repo: https://github.com/TopNetBio/flagserpy
- Download releases: https://pypi.org/project/flagserpy/
- Issue tracker: https://github.com/TopNetBio/flagserpy/issues


Contacts:
---------

jason.smith@ntu.ac.uk
