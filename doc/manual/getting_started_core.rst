Getting started with the core language
======================================

.. _connecting-to-the-core-device:

Connecting to the core device
-----------------------------

As a very first step, we will turn on a LED on the core device. Create a file ``led.py`` containing the following: ::

    from artiq.experiment import *


    class LED(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("led")

        @kernel
        def run(self):
            self.core.reset()
            self.led.on()

The central part of our code is our ``LED`` class, which derives from :class:`artiq.language.environment.EnvExperiment`. Among other features, :class:`~artiq.language.environment.EnvExperiment` calls our :meth:`~artiq.language.environment.Experiment.build` method and provides the :meth:`~artiq.language.environment.HasEnvironment.setattr_device` method that interfaces with the device database to create the appropriate device drivers and make those drivers accessible as ``self.core`` and ``self.led``. The :func:`~artiq.language.core.kernel` decorator (``@kernel``) tells the system that the :meth:`~artiq.language.environment.Experiment.run` method must be compiled for and executed on the core device (instead of being interpreted and executed as regular Python code on the host). The decorator uses ``self.core`` internally, which is why we request the core device using :meth:`~artiq.language.environment.HasEnvironment.setattr_device` like any other.

It is important that you supply the correct device database for your system configuration; it is generated by a Python script typically called ``device_db.py`` (see also :ref:`the device database <device-db>`). If you purchased a system from M-Labs, the ``device_db.py`` for your system will normally already have been provided to you (either on the USB stick, inside ``~/artiq`` on the NUC, or by email). If you have the JSON description file for your system on hand, you can use the ARTIQ front-end tool ``artiq_ddb_template`` to generate a matching device database file. Otherwise, you can also find examples in the ``examples`` folder of ARTIQ (sorted in corresponding subfolders per core device) which you can edit to match your system.  

.. note::
    To access the examples, you can find where the ARTIQ package is installed on your machine with: ::

        python3 -c "import artiq; print(artiq.__path__[0])"

Make sure ``device_db.py`` is in the same directory as ``led.py``. The field ``core_addr``, placed at the top of the file, needs to match the current IP address of your core device in order for your host machine to contact it. If you purchased a pre-assembled system and haven't changed the IP address it is normally already set correctly.

Run your code using ``artiq_run``, which is one of the ARTIQ front-end tools: ::

    $ artiq_run led.py

The process should terminate quietly and the LED of the device should turn on. Congratulations! You have a basic ARTIQ system up and running.

Host/core device interaction (RPC)
----------------------------------

A method or function running on the core device (which we call a "kernel") may communicate with the host by calling non-kernel functions that may accept parameters and may return a value. The "remote procedure call" (RPC) mechanisms automatically handle the communication between the host and the device, conveying between them what function to call, what parameters to call it with, and the resulting value, once returned. 

Modify ``led.py`` as follows: ::

    def input_led_state() -> TBool:
        return input("Enter desired LED state: ") == "1"

    class LED(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("led")

        @kernel
        def run(self):
            self.core.reset()
            s = input_led_state()
            self.core.break_realtime()
            if s:
                self.led.on()
            else:
                self.led.off()


You can then turn the LED off and on by entering 0 or 1 at the prompt that appears: ::

    $ artiq_run led.py
    Enter desired LED state: 1
    $ artiq_run led.py
    Enter desired LED state: 0

What happens is that the ARTIQ compiler notices that the :meth:`input_led_state` function does not have a ``@kernel`` decorator (:func:`~artiq.language.core.kernel`) and thus must be executed on the host. When the function is called on the core device, it sends a request to the host, which executes it. The core device waits until the host returns, and then continues the kernel; in this case, the host displays the prompt, collects user input, and the core device sets the LED state accordingly. 

The return type of all RPC functions must be known in advance. If the return value is not ``None``, the compiler requires a type annotation, like ``-> TBool`` in the example above.  

Without the :meth:`~artiq.coredevice.core.Core.break_realtime` call, the RTIO events emitted by :func:`self.led.on()` or :func:`self.led.off()` would be scheduled at a fixed and very short delay after entering :meth:`~artiq.language.environment.Experiment.run()`. These events would fail because the RPC to :meth:`input_led_state()` can take an arbitrarily long amount of time, and therefore the deadline for the submission of RTIO events would have long passed when :func:`self.led.on()` or :func:`self.led.off()` are called (that is, the ``rtio_counter`` wall clock will have advanced far ahead of the timeline cursor ``now``, and an :exc:`~artiq.coredevice.exceptions.RTIOUnderflow` would result; see :ref:`artiq-real-time-i-o-concepts` for the full explanation of wall clock vs. timeline.) The :meth:`~artiq.coredevice.core.Core.break_realtime` call is necessary to waive the real-time requirements of the LED state change. Rather than delaying by any particular time interval, it reads ``rtio_counter`` and moves up the ``now`` cursor far enough to ensure it's once again safely ahead of the wall clock. 

Real-time Input/Output (RTIO)
-----------------------------

The point of running code on the core device is the ability to meet demanding real-time constraints. In particular, the core device can respond to an incoming stimulus or the result of a measurement with a low and predictable latency. We will see how to use inputs later; first, we must familiarize ourselves with how time is managed in kernels.

Create a new file ``rtio.py`` containing the following: ::

    from artiq.experiment import *


    class Tutorial(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("ttl0")

        @kernel
        def run(self):
            self.core.reset()
            self.ttl0.output()
            for i in range(1000000):
                delay(2*us)
                self.ttl0.pulse(2*us)

In its :meth:`~artiq.language.environment.Experiment.build` method, the experiment obtains the core device and a TTL device called ``ttl0`` as defined in the device database.
In ARTIQ, TTL is used roughly synonymous with "a single generic digital signal" and does not refer to a specific signaling standard or voltage/current levels.

When :meth:`~artiq.language.environment.Experiment.run`, the experiment first ensures that ``ttl0`` is in output mode and actively driving the device it is connected to.
Bidirectional TTL channels (i.e. :class:`~artiq.coredevice.ttl.TTLInOut`) are in input (high impedance) mode by default, output-only TTL channels (:class:`~artiq.coredevice.ttl.TTLOut`) are always in output mode.
There are no input-only TTL channels.

The experiment then drives one million 2 µs long pulses separated by 2 µs each.
Connect an oscilloscope or logic analyzer to TTL0 and run ``artiq_run.py rtio.py``.
Notice that the generated signal's period is precisely 4 µs, and that it has a duty cycle of precisely 50%.
This is not what one would expect if the delay and the pulse were implemented with register-based general purpose input output (GPIO) that is CPU-controlled.
The signal's period would depend on CPU speed, and overhead from the loop, memory management, function calls, etc., all of which are hard to predict and variable.
Any asymmetry in the overhead would manifest itself in a distorted and variable duty cycle.

Instead, inside the core device, output timing is generated by the gateware and the CPU only programs switching commands with certain timestamps that the CPU computes.

This guarantees precise timing as long as the CPU can keep generating timestamps that are increasing fast enough. In the case that it fails to do so (and attempts to program an event with a timestamp smaller than the current RTIO clock timestamp), :exc:`~artiq.coredevice.exceptions.RTIOUnderflow` is raised. The kernel causing it may catch it (using a regular ``try... except...`` construct), or allow it to propagate to the host.

Try reducing the period of the generated waveform until the CPU cannot keep up with the generation of switching events and the underflow exception is raised. Then try catching it: ::

    from artiq.experiment import *


    def print_underflow():
        print("RTIO underflow occured")

    class Tutorial(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("ttl0")

        @kernel
        def run(self):
            self.core.reset()
            try:
                for i in range(1000000):
                    self.ttl0.pulse(...)
                    delay(...)
            except RTIOUnderflow:
                print_underflow()


Parallel and sequential blocks
------------------------------

It is often necessary for several pulses to overlap one another. This can be expressed through the use of ``with parallel`` constructs, in which the events generated by the individual statements are executed at the same time. The duration of the ``parallel`` block is the duration of its longest statement.

Try the following code and observe the generated pulses on a 2-channel oscilloscope or logic analyzer: ::

    from artiq.experiment import *

    class Tutorial(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("ttl0")
            self.setattr_device("ttl1")

        @kernel
        def run(self):
            self.core.reset()
            for i in range(1000000):
                with parallel:
                    self.ttl0.pulse(2*us)
                    self.ttl1.pulse(4*us)
                delay(4*us)

ARTIQ can implement ``with parallel`` blocks without having to resort to any of the typical parallel processing approaches.
It simply remembers the position on the timeline when entering the ``parallel`` block and then seeks back to that position after submitting the events generated by each statement.
In other words, the statements in the ``parallel`` block are actually executed sequentially, only the RTIO events generated by them are scheduled to be executed in parallel.
Note that accordingly if a statement takes a lot of CPU time to execute (which is different from -- and has nothing to do with! -- the events *scheduled* by the statement taking a long time), it may cause a subsequent statement to miss the deadline for timely submission of its events (and raise :exc:`~artiq.coredevice.exceptions.RTIOUnderflow`), while earlier statements in the parallel block would have submitted their events without problems.   

Within a parallel block, some statements can be scheduled sequentially again using a ``with sequential`` block. Observe the pulses generated by this code: ::

    for i in range(1000000):
        with parallel:
            with sequential:
                self.ttl0.pulse(2*us)
                delay(1*us)
                self.ttl0.pulse(1*us)
            self.ttl1.pulse(4*us)
        delay(4*us)

Particular care needs to be taken when working with ``parallel`` blocks which generate large numbers of RTIO events, as it is possible to create sequence errors. A sequence error is caused when the scalable event dispatcher (SED) cannot queue an RTIO event due to its timestamp being the same as or earlier than another event in its queue. By default, the SED has 8 lanes, which suffice in most cases to avoid sequence errors; however, if many (>8) events are queued with interlaced timestamps the problem can still surface. See :ref:`sequence-errors`. 

Note that for performance reasons sequence errors do not halt execution of the kernel. Instead, they are reported in the core log. If the ``aqctl_corelog`` process has been started with ``artiq_ctlmgr``, then these errors will be posted to the master log. If an experiment is executed through ``artiq_run``, the errors will only be visible in the core log. 

Sequence errors can usually be overcome by reordering the generation of the events (again, different from and unrelated to reordering the events themselves). Alternatively, the number of SED lanes can be increased in the gateware.

.. _rtio-analyzer-example:

RTIO analyzer
-------------

The core device records the real-time I/O waveforms into a circular buffer. It is possible to dump any Python object so that it appears alongside the waveforms using the ``rtio_log`` function, which accepts a channel name (i.e. a log target) as the first argument: ::

    from artiq.experiment import *


    class Tutorial(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("ttl0")

        @kernel
        def run(self):
            self.core.reset()
            for i in range(100):
                self.ttl0.pulse(...)
                rtio_log("ttl0", "i", i)
                delay(...)

Afterwards, the recorded data can be extracted and written to a VCD file using ``artiq_coreanalyzer -w rtio.vcd`` (see :ref:`core-device-rtio-analyzer-tool`). VCD files can be viewed using third-party tools such as GtkWave.

.. _getting-started-dma: 

Direct Memory Access (DMA)
--------------------------

DMA allows for storing fixed sequences of RTIO events in system memory and having the DMA core in the FPGA play them back at high speed. Provided that the specifications of a desired event sequence are known far enough in advance, and no other RTIO issues (collisions, sequence errors) are provoked, even extremely fast and detailed event sequences are always possible to generate and execute. However, if they are time-consuming for the CPU to generate, they may require very large amounts of positive slack in order to allow the CPU enough time to complete the generation before the wall clock 'catches up' (that is, without running into RTIO underflows). A better option is to record these sequences to the DMA core. Once recorded, events sequences are fixed and cannot be modified, but can be safely replayed at any position in the timeline, potentially repeatedly. 

Try this: ::

    from artiq.experiment import *


    class DMAPulses(EnvExperiment):
        def build(self):
            self.setattr_device("core")
            self.setattr_device("core_dma")
            self.setattr_device("ttl0")

        @kernel
        def record(self):
            with self.core_dma.record("pulses"):
                # all RTIO operations now go to the "pulses"
                # DMA buffer, instead of being executed immediately.
                for i in range(50):
                    self.ttl0.pulse(100*ns)
                    delay(100*ns)

        @kernel
        def run(self):
            self.core.reset()
            self.record()
            # prefetch the address of the DMA buffer
            # for faster playback trigger
            pulses_handle = self.core_dma.get_handle("pulses")
            self.core.break_realtime()
            while True:
                # execute RTIO operations in the DMA buffer
                # each playback advances the timeline by 50*(100+100) ns
                self.core_dma.playback_handle(pulses_handle)

For more documentation on the methods used, see the :mod:`artiq.coredevice.dma` reference.
