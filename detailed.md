
Optical tests:
==============

The XY stage is has two EZstepper controllers on addresses 6 and 7 and has a half-duplex RS485 (two lines) connection on a DB15 connector. The +28V power power for the steppers comes from the red and black lines on the opposite side of the DB15. The RS485 to RS232 conveter that we use was supplied by EZStepper and requires 12V (green +/- header) in addition to the RS485 on two input pins labelled A and B.This is supplied by a dual voltage supply. We then connect RS285->RS232->USB->Fiber->USB port. The USB connection should appear in `lsusb` or `dmesg` and a new device under `/dev/ttyUSB#` is started. You can connect with the terminal as `minicom /dev/ttyUSB#` and set the options to 9600 8N1 and echo local typing. The commands are described in PDF "EZ Stepper Command set and Communications protocol, Command Set For Stepper Models: EZ17, EZHR17, EZHR23".

The x-stage has two rails. The second is for mechanical support. The kit comes with additional platforms but these are not needed, and the Y-driver can be connected directly to the X stage platform using four mounting screws at the bottom. If the electrical connectors to the stepper break, they can be replaced with standard Molex pins from a computer power cable. The stage software and scan are based on those used for BLAST and are in `borsnn:~act/code/optical_measurements/` (on SVN soon). `move_stage` takes x, xvel, y, yvel and moves to that absolute position. Running it with no arguments reads the XY encoders. (Positions are in the thousands and 500 is a good velocity). It does not go negative, and the encoder must be re-zeroed to reach the full stage range. `scan_stage` rasters the stage in X, then in Y. It takes arguments xcenter, xsize, ycenter, ysize, xvel, yvel, and step size. The stage controller software captures ctrl-C and terminates motion.

The TOD written here is `x(t) y(t) serial#(t) chopper(t)` and can be synced with the `serial#(t) detector(t)`. An alternative to the raster scan is point and shoot, moving to some XY, parking, taking data, etc. The problem with this is that each grid position to map the beam would get its own flat and dirfiles.

The Vincent Assoc. VS25S2T1 shutter is controlled by a Uniblitz VMM-D1 controller triggered by TTL from the Wavetek ("Pulse" BNC on the controller, not "Trigger" BNC which does every second TTL pulse). The TTL has a T to a Measurement Computing 1608FS USB ADC. We use channel 0 and run the USB->Fiber->USB port. The Uniblitz controller has an RS232 controller and cable in the box, but we do not use it. The 1608FS can read at high rates in blocks, but we need a sync box serial stamp per read, so have a loop that reads the serial stamp through the BBCPCI card and a single voltage read from channel 0 on the ADC. This sequential read is limited to ~200 Hz. The code is in `borsnn:~act/code/optical_measurements/`. FTS measurements only need a chopper/fan TTL voltage and a timestamp. This can be done with `read_chopper`. The serial signal from the syncbox can be tested using `bbcpci/test_data`. Because these test measurements need the BBC stamp, `amcp` can not use the BBC for triggering at the same time. This is fine because the BBCpci<->amcp connection is mainly for synchronizing the telescope encoders, which do not need to be tracked in these test.

To increase the rate above 200 Hz, the 1608FS needs to be read in blocks, and a possibility there to record the sync serial number is to use the onboard counter or trigger. The sync box has ``FTS`` and poarimeter outputs that give a TTL for the start of the data frame on pins 2 and 7. The step is ~25 ns, and the trigger or counter of the 1608FS need > 500 ns. To use these scheme, those pulses would need to trigger a pulse with extended duration that could be read by the counter. The shutter could have been read by the national instruments DAQs to USB, but it would have required significant effort to bring those channels in and acquire at a rate higher than the thermometry. This would still be asynchronous, and the XY stage software would have to have been added to `amcp`. Because XY stage measurements are a one-off test, we opted against this.

Commanding
==========
The master database is `http://cita.utoronto.ca/~eswitzer/master.json`. This will move to a site computer. The redis server is running on bors and the command is `PUBLISH housekeeping "variable_name value"`. The amcp response is `PUBLISH housekeeping_ack variable_name`, and you can then call a `GET variable_name` to see its current value, which is also tracked on redis. Commands can be directly issued to redis using `redis-cli` on bors. The redis port will stay behind the gateway, and we should use some combination of a tunnel/VPN to access site controls.

ACT to ACTPol software:
=======================
* The encoder driver `ik220`, BLAST bus card (BBC) driver `bbcpci` and polling code `act_timing` are updated to recent linux kernels (32 bit).
* The `interface_server` has been replaced with `redis` (a key-value store with publish/subscribe). The master shelve file which previously specified commands has been replaced with a JSON file available on the web. The housekeeping system parses this file into its `control_struct.c/h.`.
* The old Tkinter/Pmw GUI is replaced with wxpython which builds its interface from the JSON specification. It talks to redis, but web commanding interfaces are also planned.
* The BBC/Toronto infrastructure has been removed from `amcp` except for reading the serial number from the syncronization box.
* All of the relevant code from the feynman CVS server is revamped and now on an SVN at UPenn.
* Integration of the thermometry thread is underway.
* National instruments housekeeping readout to 3 USBs->Fiber->USB port in housekeeping computer in the equipment room.
* We have commanded the telescope through amcp; 1 deg motion.

BBC:
====
* The BLAST bus uses half-duplex MAX3088 drivers for RS-485 control (clock, strobe and data). These are independently powered and opto-isolated on the sending and receiving end. For the card to receive serial sync signals from the UBC sync box, this stage must be powered. On the PCI card side, one has the option of powering the IO stage using PCI power taken off the daughter card. For safety, rather than drawing from the bus, we power the +5V IO stage using a SATA connector to the computer supply. A separate RS-232 serial link with null modem sends commands to the sync box from the same computer.
* If the sync box power cycles, `act_timing` needs to be restarted as `sudo /etc/init.d/act_timing restart`.

`amcp` installation (on older OSs):
====================================
* Redis `wget http://redis.googlecode.com/files/redis-2.4.11.tar.gz` (see `http://redis.io/download`) untar it, then `make; sudo make install; cd utils; sudo ./install_server.sh`
* libevent `wget https://github.com/downloads/libevent/libevent/libevent-2.0.18-stable.tar.gz` (see `http://libevent.org/`) untar it, then `./configure; make; sudo make install`
* hiredis `wget https://github.com/antirez/hiredis/tarball/master` (see `https://github.com/antirez/hiredis/downloads`) untar it, then `make; sudo make install`
* run `sudo /sbin/ldconfig`
* if you do not have the housekeeping trunk `svn co http://actpolsvn.info/svn/actpol_software/housekeeping/trunk`
* in `/etc/redis.conf` (or similar) set `timeout 0` so that subscribed amcp/client sessions will not time out (this is not handled well yet)
