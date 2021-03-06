ReconOS partial reconfiguration demo
====================================

Target: Xilinx ML605

Overview
--------

This ReconOS demo application consists of four hardware threads, each of which
receives a 32-bit message, sets a bit (the position of which depends on the
gemeric C_THREAD_NUM set at synthesis time) and forwards the resulting message
to another mailbox (which gets passed to the next thread). Each thread also
sends the mask of its modified bit in a separate message to main() to indicate
successful execution.

The main() program creates the four threads, passes a message set to 0x0 to
the first thread and waits for all messages (four bit masks and the final
result). It then waits for a few seconds before creating a new set of threads
and repeating the process indefinitely. It also compares the bitmasks and
results for correct computation.

Thus, the flow of messages looks as follows:

    0x00000000
        |
        V
    thread_1 (0x40000000) -------+
        |                        |
        V                        |
    0x40000000                   |
        |                        |
        V                        |
    thread_2 (0x20000000) -----+ |
        |                      | |
        V                      V V
    0x60000000                main()
        |                     ^ ^ ^
        V                     | | |
    thread_3 (0x10000000) ----+ | |
        |                       | |
        V                       | |
    0x70000000                  | |
        |                       | |
        V                       | |
    thread_4 (0x08000000) ------+ |
        |                         |
        V                         |
    0x78000000  (result) ---------+


In the hardware platform, only two slots are included, so that during a single
iteration of the loop multiple reconfiguratios are necessary. In fact, as long
as the reconfiguration logic (i.e., the HW scheduler) is not smart enough to
see what threads are already in the slots, _all_ threads created will cause a
partial reconfiguration.


Manually building the demo
--------------------------

These are the steps necessary to build the demo manually using the regular
ReconOS tool chain and PlanAhead. Steps for automatic builds are shown in 
a later section.

Please read the whole process description before starting.


* Build the static netlists for the base design

    cd $RECONOS/demos/pr_msg_demo/hw
    make netlist-static


* Remove the dummy hardware thread netlists (one per slot) from the
synthsized EDK design, so that PlanAhead treats the hw_task_* modules as black
boxes.

    rm -rf edk-static/implementation/hw_task_*


* Build the hardware thread netlists

    cd $RECONOS/demos/pr_msg_demo/hw/hwthreads
    make


* Create a new PlanAhead project (with the PR feature enabled). Use
$RECONOS/demos/pr_msg_demo/hw/edk-static/synthesis/system.ngc as the toplevel
netlist, and .../edk-static/implementation as the netlist directory. Also use
.../edk-static/data/system.ucf as the constraints file.

* Insert the created thread_x.ngc (in .../hwthreads/) files of the respective hardware
thread as partially reconfigurable modules within PlanAhead (TODO: more
elaborate instructions, refer to Xilinx PR documentation).

* Place the Pblocks for the slots, and *save* the project.

* Create the necessary number of configurations (so that every thread ist
placed once in every slot).

* Run the design run for the first configuration.

* Promote the resulting partitions.

* Repeat the previous two steps for all configurations.

* Verify all configurations.

* Create bitstreams for all configurations / design runs.

As a result, you will have complete bistreams for all configurations, as well
as partial bitstreams for every thread/slot combination. The software build
process assumes you have named the four necessary configurations "config_1",
"config_2", "config_3", and "config_4". The eight resulting partial bitstreams
should thus be:

* config_1_hw_task_0_thread_1_partial.bit
* config_2_hw_task_0_thread_2_partial.bit
* config_3_hw_task_0_thread_3_partial.bit
* config_4_hw_task_0_thread_4_partial.bit
* config_1_hw_task_1_thread_1_partial.bit
* config_2_hw_task_1_thread_2_partial.bit
* config_3_hw_task_1_thread_3_partial.bit
* config_4_hw_task_1_thread_4_partial.bit

They can be found in the ".runs" subdirectory of the PlanAhead project.

To build the software application for the pr_msg_demo, issue the following
commands (you might need to edit the Makefile in the 'sw' directory, first,
to account for the PlanAhead project name):

    cd $RECONOS/demos/pr_msg_demo/sw
    export HW_DESIGN=$RECONOS/demos/pr_msg_demo/hw/edk-static
    make mrproper setup all

This will create a pr_msg_demo.elf executable including the software
application as well as all the partial bitstreams in a single image.


Automatically building the demo
-------------------------------

Using the makefiles supplied by ReconOS, the entire build process can be
automated, assuming that all project description files (.rprj, .lyt) contain
valid information. In this case, the build process for the complete and
partial bitstreams of the pr_msg_demo is as follows:

Change to the hardware directory and invoke 'make':

    cd $RECONOS/demos/pr_msg_demo/hw
    make all
    
The complete bitstreams can then be found in 

    .../hw/project_reconos_1/project_reconos_1.runs/config_x/config_x.bit

where 'x' is the configuration number (1 to 4).

The partial bitstreams are located in the same directories, with filenames:

* config_1_hw_task_0_thread_1_partial.bit
* config_2_hw_task_0_thread_2_partial.bit
* config_3_hw_task_0_thread_3_partial.bit
* config_4_hw_task_0_thread_4_partial.bit
* config_1_hw_task_1_thread_1_partial.bit
* config_2_hw_task_1_thread_2_partial.bit
* config_3_hw_task_1_thread_3_partial.bit
* config_4_hw_task_1_thread_4_partial.bit

Building the software application is exactly the same process as in a manual
build:

    cd $RECONOS/demos/pr_msg_demo/sw
    export HW_DESIGN=$RECONOS/demos/pr_msg_demo/hw/edk-static
    make mrproper setup all

resulting in a 'pr_msg_demo.elf' which can be uploaded to an ML605 which has
been initialized with a complete bistream (see next section).

Running the demo
----------------

To execute the pr_msg_demo, you need the following files:

* config_1.bit (one of the complete configurations created by PlanAhead)
* pr_msg_demo.elf (the software executable including the partial bitstreams)

Just use the following steps:

* Open a serial terminal on the port connected to the ML605, using the
parameters "57600 8N1" (e.g., using minicom).
* Download the full configuration bitstream to the FPGA:

    dow config_1.bit

* Download the application executable (this may take a while):

    dow pr_msg_demo.elf

You should now see the application output on the serial terminal, showing
repeated execution of the main application loop and "PASSED" checks on the
results.

