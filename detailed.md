
DeviceNet driver rewrite:
=========================

`appio/apdriver/src/adkmail.c`:
------
Kernel driver functions have 1k limits for their stack variables. Sizes over that have to be explicitly allocated using kmalloc and kfree. Previously, the driver used `ST_RAMIO` and `APP_MAILBOX2` structures that brought the total memory used by the function > 1k. I replaced these with pointers to kmalloc memory, and referencing of these structs later. The 1k limit must have been larger on the old bors Fedora 4 kernel. The kernel limit can be increased using frame size in the `kernel hacking` configuration section. It is better to just kmalloc so that a kernel does not need to be compiled specially for devicenet.

`appio/apdriver/src/kernel.c`:
------
The device major number gives the driver entry in `/proc/devices` to send requests to, and the minor is a subhandler within that driver. The kernel does not see the minor number, and it only has to be chosen to not conflict with another handler. In our case, the devicenet driver registers itself as minor 157 under misc, for misc (unbuffered) character devices on major 10. Registration with a major/minor usually takes place through sysfs class device calls. In our case this was `applicomio_start_sysfs()` in `kernel.c`. An alternative approach is to tack onto the kernel misc driver with `misc_register`. The old driver did both of these things, but the newer kernel must be pickier about doing just one or the other. If you dump `appio.a` (binary driver library from the vendor), you find that it accesses `/dev/ac`. The old devicenet driver used sysfs first to register `/dev/applicomio`, though, before a call to `misc_register` could register `/dev/ac`. The call to `misc_register` failed because 10:157 had already been claimed by sysfs. I can not find any software that uses the sysfs or `/dev/applicomio`, so this must have just been provided for convenience. For now, the sysfs code is still in `kernel.c`, but needed to be updated to 2.6 kernels:
* `class_simple` -> `class`
* `class_device` -> `device`
* `class_device_attribute` -> `device_attribute`
* `class_simple_create` -> `class_create`
* `class_simple_device_add(applicomio_class, ...)` -> `device_create(applicomio_class, NULL, ...)`
* `class_device_create_file` -> `device_create_file` 

Note that `misc_register` makes the `/dev` and `/dev/char` entries, so no calls to `mknod` are needed. For future reference, the driver looks for PCI and ISA, but the card is PCI irq=11. Failures in kern.log backtrace are listed in reverse order of call.

`appio/apdriver/src/applicomio.c`:
-----
* unused macros commented out, more diagnostics printed (see /var/log/kern.log after running modprobe). In kernel code, primitive printk should be used in
place of printf.
* added `__init` and `__exit` macros to the module init and exit function prototypes, cleaned up conditionals

`appio/apdriver/src/u_ahal.c`:
-----
* `request_irq` wants explicit cast to `irq_hander_t`, which is a pointer to a function that returns an `irq_request_t`: `request_irq(ui_IRQ, &ac_interrupt, SA_SHIRQ,...)` -> `request_irq(ui_IRQ,
(irq_handler_t)ac_interrupt, IRQF_SHARED,...)`

`appio/apincl/drv/applicom.h`:
-------
* added `#include <linux/version.h>`,
* commented  `//static loff_t ac_llseek(struct file *, loff_t, int);`

`Makefile`
------
* In install, old chkconfig and udev code is from Fedora 4 and needs to be replaced with newer boot-time configuration mgmt for ubuntu.
* rename: appio.a -> libappio.a so that -lappio works

Summary of other drivers updates:
--------------------
* The encoder driver `ik220`, BLAST bus card (BBC) driver `bbcpci` and polling code `act_timing` are updated to recent linux kernels (32 bit).
* The `interface_server` has been replaced with redis. The master shelve file which previously specified commands has been replaced with a JSON file available on the web or from the redis server. The housekeeping system parses this file into its control_struct.c/h.
* The old Tkinter/Pmw GUI is replaced by wxpython which builds its interface from the JSON specification.
* The BBC/Toronto infrastructure has been removed from amcp except for reading the serial number from the syncronization box.
* All of the relevant code from the feynman CVS server is revamped and now on an SVN at UPenn.
* amcp running, old ACT code removed, redis scheduler talks to redis, telescope motion tests through amcp
* bede streaming to adefile down the mountain (note: slow)
* hardware: replacement for bors up and running (borsn), Ubuntu 10.04 32-bit, move cards to borsn and plug in cables as before, update HD and OS in old bors, upgrade the link to Canopy

Optical test setup:
===================
Software: The stage controller and shutter readout are in `borsnn:~act/code/optical_measurements/`.

Shutter readout:
----------------
The Vincent Assoc. VS25S2T1 shutter is controlled by a Uniblitz VMM-D1 driver triggered by TTL from the Wavetek ("Pulse" BNC on the controller, not the "Trigger" BNC, which does every second TTL pulse). The TTL has a T to a Measurement Computing 1608FS USB ADC. We use channel 0 and run the USB->Fiber->USB port. The Uniblitz controller has an RS232 controller and cable in the box, but we do not use it. The 1608FS can read at high rates in blocks, but we need a sync box serial stamp per read, so have a loop that reads the serial stamp through the BBCPCI card and a single voltage read from the ADC. This sequential read is limited to ~200 Hz. FTS measurements only need a chopper/fan TTL voltage and a timestamp. This can be done with `read_chopper`. The serial signal from the syncbox can be tested using `bbcpci/test_data`. Because these test measurements need the BBC stamp, `amcp` can not use the BBC for triggering at the same time. This is fine because the BBCpci<->amcp connection is mainly for synchronizing the telescope encoders, which do not need to be tracked in these tests.

To increase the rate above 200 Hz, the 1608FS needs to be read in blocks, and a possibility there to record the sync serial number is to use the onboard counter or trigger. The sync box has "FTS" and polarimeter outputs that give a TTL for the start of the data frame on pins 2 and 7. The step is ~25 ns, and the trigger or counter of the 1608FS need > 500 ns. To use these scheme, those pulses would need to trigger a pulse with extended duration that could be read by the 1608. The shutter could have been read by the national instruments DAQs to USB, but it would have required significant effort to bring those channels in and acquire at a rate higher than the thermometry. This would still be asynchronous, and the XY stage software would have to have been added to `amcp`. Because optical measurements are a one-off test, we opted against this.

XY stage controller:
--------------------
The XY stage is has two EZstepper controllers on addresses 6 and 7 and has a half-duplex RS485 (two lines) connection on a DB15 connector. The +28V power power for the steppers comes from the red and black lines on the opposite side of the DB15. The RS485 to RS232 converter that we use was supplied by EZStepper and requires 12V (green +/- header) in addition to the RS485 on two input pins labelled A and B.We then connect RS285->RS232->USB->Fiber->USB port. The USB connection should appear in `lsusb` or `dmesg` and a new device under `/dev/ttyUSB#` is started. You can connect with the terminal as `minicom /dev/ttyUSB#` and set the options to 9600 8N1 and echo local typing. The commands are described in PDF "EZ Stepper Command set and Communications protocol, Command Set For Stepper Models: EZ17, EZHR17, EZHR23".

The x-stage has two rails. The second is for support. The kit comes with additional platforms but these are not needed, and the Y-driver can be connected directly to the X stage platform using four mounting screws at the bottom. If the electrical connectors to the stepper break, they can be replaced with standard Molex pins from a computer power cable. `move_stage` takes x, xvel, y, yvel and moves to that absolute position. Running it with no arguments reads the XY encoders. (Positions are in the thousands and 500 is a good velocity). It does not go negative, and the encoder must be re-zeroed to reach the full stage range. `scan_stage` rasters the stage in X, then in Y. It takes arguments xcenter, xsize, ycenter, ysize, xvel, yvel, and step size. The stage controller software captures ctrl-C and terminates motion.

The TOD written here is `x(t) y(t) serial#(t) chopper(t)` and can be synced with the `serial#(t) detector(t)`. The maximum read rate here is 17 Hz because the XY stage, BBC, and USB ADC are read sequentially and the rate is limited by the 9600 baud connection to the XY stage. An alternative to the raster scan is point and shoot, moving to some XY, parking, taking data, etc. The problem with this is that each grid position to map the beam would get its own flat and dirfiles. To go faster than 17 Hz, `scan_stage` would need to be made multi-threaded and read each channel as fast as possible, repeating values last received on the slowest channels.

Commanding
==========
The old commanding software was based on the `interface_server`. This has been replaced by `redis`, a new/widely used/well-maintained open source database that has almost exactly the same behavior. The publish/subscribe model is used to issue commands to `amcp`, which replies with an acknowledgment and sets the key/value on the server for that variable. The old master shelve file which specified the telescope commands has been replaced with a web-accessible `master.json`. Commands can be issued through redis using either a GUI or text interface. The text interface is available through `redis-cli` on the houskeeping computer (or telnet to the redis port). For text-based commanding, it is a good idea to start a terminal with redis-cli and run `monitor` to see all messages on the server. The full set of redis commands is available on `http://redis.io/commands`, but we only use `publish` and `get` for commanding. `info` gives general information and `monitor` tails a command log.

To set a housekeeping variable, `publish housekeeping "variable_name value"`. The amcp response is `PUBLISH housekeeping_ack variable_name`, and you can then call a `GET variable_name` to see its current value. Some variables (like a servo state) may update too fast and not be updated live on the server. The machine running redis is on the local network; an ssh tunnel/VPN through the gateway allows remote access. The `master.json` file is parsed into `control_struct.c/h` in amcp.

The old commanding GUI `excalibur` was based on Pmw and Tkinter and has been replaced with a wxpython interface that builds itself from the JSON specification, `https://github.com/eric-switzer/viscacha`. A web commanding interface is also planned.

The auto-startup sequency is `dn_start_seq`. Set this high using:
`publish housekeeping "dn_start_seq 1"`
publish sends a message to any subscribers of the "housekeeping" channel (this is just amcp now) and the message is "dn_start_seq 1" where 1 is the requested value. amcp should reply by publishing to "housekeeping_ack" with the variable name it received. It also sets that value in the key/value store, and you can check its value by using `get dn_start_seq`. Actions still need a variable value, so e.g. publish housekeeping "point_go_home 1", needs the "1" to set that request high.@


Updates and software organization
=================================
* The encoder driver `ik220`, BLAST bus card (BBC) driver `bbcpci` and polling code `act_timing` are updated to recent linux kernels (32 bit).
* The BBC/Toronto infrastructure has been removed from `amcp` except for reading the serial number from the syncronization box.
* National instruments housekeeping readout to 3 USBs->Fiber->USB port in housekeeping computer in the equipment room.
* All of the relevant code from the feynman CVS server is revamped and now on an SVN at UPenn.
* Remaining: Integrating the thermometry in the frame. Moving from 32 to 64 bit OS and impact on drivers? Newest LTS Ubuntu.

Using the BBC pci card as a fast serial reader:
===============================================
The BLAST bus uses half-duplex MAX3088 drivers for RS-485 control (clock, strobe and data). These are independently powered and opto-isolated on the sending and receiving end. For the card to receive serial sync signals from the UBC sync box, this stage must be powered. On the PCI card side, one has the option of powering the IO stage using PCI power taken off the daughter card. For safety, rather than drawing from the bus, we power the +5V IO stage using a SATA connector to the computer supply. A separate RS-232 serial link with null modem sends commands to the sync box from the same computer.

If the sync box power cycles, `act_timing` needs to be restarted as `sudo /etc/init.d/act_timing restart`.

`amcp` installation (on older OSs):
====================================
* Redis `wget http://redis.googlecode.com/files/redis-2.4.11.tar.gz` (see `http://redis.io/download`) untar it, then `make; sudo make install; cd utils; sudo ./install_server.sh`
* libevent `wget https://github.com/downloads/libevent/libevent/libevent-2.0.18-stable.tar.gz` (see `http://libevent.org/`) untar it, then `./configure; make; sudo make install`
* hiredis `wget https://github.com/antirez/hiredis/tarball/master` (see `https://github.com/antirez/hiredis/downloads`) untar it, then `make; sudo make install`
* run `sudo /sbin/ldconfig`
* if you do not have the housekeeping trunk `svn co http://actpolsvn.info/svn/actpol_software/housekeeping/trunk`
* in `/etc/redis.conf` (or similar) set `timeout 0` so that subscribed amcp/client sessions will not time out (this is not handled well yet)

amcp writes its logs to /data/housekeeping/etc/amcp.log; use `tail -f` or multitail.

