Utilities
=========

.. Sort these tool by some subjective combination of their
   typical sequence and expected frequency of use.


Local running tool
------------------

.. argparse::
   :ref: artiq.frontend.artiq_run.get_argparser
   :prog: artiq_run

Static compiler
---------------

This tool compiles an experiment into a ELF file. It is primarily used to prepare binaries for the default experiment loaded in non-volatile storage of the core device.
Experiments compiled with this tool are not allowed to use RPCs, and their ``run`` entry point must be a kernel.

.. argparse::
   :ref: artiq.frontend.artiq_compile.get_argparser
   :prog: artiq_compile

Flash storage image generator
-----------------------------

This tool compiles key/value pairs into a binary image suitable for flashing into the flash storage space of the core device.

.. argparse::
   :ref: artiq.frontend.artiq_mkfs.get_argparser
   :prog: artiq_mkfs

.. _flashing-loading-tool: 

Flashing/Loading tool
---------------------

.. argparse::
   :ref: artiq.frontend.artiq_flash.get_argparser
   :prog: artiq_flash

.. _core-device-management-tool:

Core device management tool
---------------------------

The artiq_coremgmt utility gives remote access to the core device logs, the :ref:`core-device-flash-storage`, and other management functions.

To use this tool, it is necessary to specify the IP address your core device can be contacted at. If no option is used, the utility will assume there is a file named ``device_db.py`` in the current directory containing the device database; otherwise, a device database file can be provided with ``--device-db`` or an address directly with ``--device`` (see also below).

To read core device logs::

    $ artiq_coremgmt log

To set core device log level and UART log level (possible levels are ``TRACE``, ``DEBUG``, ``INFO``, ``WARN`` and ``ERROR``)::

    $ artiq_coremgmt log set_level LEVEL
    $ artiq_coremgmt log set_uart_level LEVEL

Note that enabling the ``TRACE`` log level results in small core device slowdown, and printing large amounts of log messages to the UART results in significant core device slowdown.

To read the record whose key is ``mac``::

    $ artiq_coremgmt config read mac

To write the value ``test_value`` in the key ``my_key``::

    $ artiq_coremgmt config write -s my_key test_value
    $ artiq_coremgmt config read my_key
    b'test_value'

You can also write entire files in a record using the ``-f`` parameter. This is useful for instance to write the startup and idle kernels in the flash storage::

    $ artiq_coremgmt config write -f idle_kernel idle.elf
    $ artiq_coremgmt config read idle_kernel | head -c9
    b'\x7fELF

You can write several records at once::

    $ artiq_coremgmt config write -s key1 value1 -f key2 filename -s key3 value3

To remove the previously written key ``my_key``::

    $ artiq_coremgmt config remove my_key

You can remove several keys at once::

    $ artiq_coremgmt config remove key1 key2

To erase the entire flash storage area::

    $ artiq_coremgmt config erase

You do not need to remove a record in order to change its value, just overwrite it::

    $ artiq_coremgmt config write -s my_key some_value
    $ artiq_coremgmt config write -s my_key some_other_value
    $ artiq_coremgmt config read my_key
    b'some_other_value'

.. argparse::
   :ref: artiq.frontend.artiq_coremgmt.get_argparser
   :prog: artiq_coremgmt

Core device logging controller
------------------------------

.. argparse::
   :ref: artiq.frontend.aqctl_corelog.get_argparser
   :prog: aqctl_corelog

Moninj proxy
------------

.. argparse::
   :ref: artiq.frontend.aqctl_moninj_proxy.get_argparser
   :prog: aqctl_moninj_proxy

.. _rtiomap-tool:

RTIO channel name map tool
--------------------------

.. argparse::
   :ref: artiq.frontend.artiq_rtiomap.get_argparser
   :prog: artiq_rtiomap


.. _core-device-rtio-analyzer-tool:

Core device RTIO analyzer tool
------------------------------

:mod:`~artiq.frontend.artiq_coreanalyzer` is a tool to convert core device RTIO logs to VCD waveform files that are readable by third-party tools such as GtkWave. This tool extracts pre-recorded data from an ARTIQ core device buffer (or from a file with the ``-r`` option), and converts it to a standard VCD file format. See :ref:`rtio-analyzer-example` for an example, or :mod:`artiq.test.coredevice.test_analyzer` for a relevant unit test.

.. argparse::
   :ref: artiq.frontend.artiq_coreanalyzer.get_argparser
   :prog: artiq_coreanalyzer

.. _routing-table-tool:

Core device RTIO analyzer proxy
-------------------------------

:mod:`~artiq.frontend.aqctl_coreanalyzer_proxy` is a tool to distribute the core analyzer dump to several clients such as the dashboard. 

.. argparse::
   :ref: artiq.frontend.aqctl_coreanalyzer_proxy.get_argparser
   :prog: aqctl_coreanalyzer_proxy

DRTIO routing table manipulation tool
-------------------------------------

.. argparse::
   :ref: artiq.frontend.artiq_route.get_argparser
   :prog: artiq_route
