Housekeeping
============

Timing
-------

A synchronization box in the telescope’s receiver cabin generates a system-wide trigger and an associated serial stamp. This gives a synchronization reference for the science camera, housekeeping system, and telescope motion encoders. The stamp and trigger are passed to each science camera’s acquisition electronics through a fiber optic, and to the housekeeping computer (in the equipment room) through an RS485 channel in the cable wrap. The synchronization box sets the fundamental rate for triggering observations by counting down a 25 MHz clock over 50 cycles per detector row, over 32 rows of detectors plus one row of dark SQUIDs for the array readout, over 38 array reads per sample trigger. This gives a final rate of ∼ 399 Hz, which oversamples the beam at the highest frequency.

The synchronization serial stamp from the RS485 channel is incorporated into the housekeeping data (including the encoders) through the following chain: 1) in the housekeeping computer, the primary housekeeping BLAST Bus PCI card receives a 5 MHz biphase signal which encodes serial data stamps at 399 Hz over RS485 from the synchronization box; 2) these trigger CPU interrupts at 399 Hz; 3) the `act_timing` driver handles these interrupts by polling the encoders (the `ik220` driver) at 399 Hz; 4) the housekeeping PCI card clocks down the serial stamps to 99.7 Hz, which it uses to poll the housekeeping acquisition electronics; 5) the housekeeping software then assembles the housekeeping and encoder time streams, matching serial stamps; 6) these are written to disk in a flat file format, where for every data frame there are, for example, 4 times as many 399 Hz encoder values as there are 99.7 Hz housekeeping data frames.

A Meinberg GPS-169 PCI card disciplines the system clock of the housekeeping computer, with a precision of < 1 ms to GPS time. The absolute GPS time is attached to the data from each encoder read request, and the GPS position is averaged down to give a geodetic latitude and longitude for astrometry. The housekeeping computer provides a stratum 0 server for the other computers (data acquisition, merging) on the network to synchronize file time stamps that are used to organize and track files.

Detector readout
-----------------

Each of the three bands has an independent set of Multi-channel electronics (MCE) which is mounted directly onto the camera cryostat and accesses the cold multiplexers through 5 × 100 pin MDM connector ports (per MCE). The MCEs, their power supplies, and MBAC all reside in the telescope receiver cabin. The three MCEs are connected to three storage and control computers in the equipment room through 6 (3 TX/RX pairs, 250 Mbps) Tyco PRO BEAM Jr. rugged multimode fiber optic carriers. The signals from each array’s MCE are decoded by a PCI card from Astronomical Research Cameras, Inc. in each of the three acquisition computers.

The base clock rate of the MCE is 50 MHz. This is divided down to 100 clock cycles per detector row by 32 detector rows plus one row of dark SQUIDs. The native read-rate of the array is then 15.15 kHz as it leaves the cryostat. Nyquist inductors at 0.3 K in the LR-circuit of the detector loop band-limit the response so the array can be multiplexed with minimal aliased noise while maintaining stability of the loop. For science data, the only bandwidth constraint is to at least Nyquist sample the optical beam. To downsample the 15.15 kHz multiplexing rate to 399 Hz, the MCE applies a 4-pole Butterworth filter with a rolloff f3dB = 122 Hz to the feedback stream from each detector. This filter is efficient to implement digitally and has a flat passband. The rolloff then defines our conservative downsampling to 399 Hz, which can be obtained by pulling every 38th sample (at 15.15 kHz) from the filter stack, synchronized by similar clock counting in the synchronization box.

Each time an MCE receives a synchronization serial stamp from the synchronization box, it packages the output of the 32 × 32 + 1 × 32 (32 dark SQUIDs) array and sends it over a fiber optic to a PCI card on its acquisition computer, where it is buffered and written to disk. Additional information that fully specifies the MCE state in each 15-minute acquisition interval is written to a text "run file".
Each of the three data acquisition computers in the equipment room is responsible for coordinating the operations of its MCE.

Housekeeping readout
--------------------

Housekeeping comprises all electronics and systems other than the camera and its electronics and the telescope motion control. The primary systems within housekeeping are the cryogenic thermal readouts and controllers, the telescope health readouts, the telescope motion encoders, and auxiliary monitors (e.g. a magnetometer). The telescope’s azimuth and elevation pointing are read by two Heidenhain encoders (27 bit, 0.0097" accuracy) through a Heidenhain (model IK220) PCI card in the housekeeping computer over RS485 from the cable wrap.

Housekeeping includes data from a variety of auxiliary sources such as a 3-axis magnetometer, dewar pressure monitor, and other system status monitors. A Sensoray 2600 DAQ in the receiver cabin reads auxiliary channels that do not need to be synchronized with the science camera data (such as the primary mirror panel temperatures). It sends these data through the site network at sampling rates from 1 − 20 Hz. Weather data are available both from an on-site WeatherHawk and from the APEX collaboration (`http://www.apex-telescope.org/weather/Historical_weather/`). A CCD star camera can also be mounted to the top of the primary mirror structure as needed to develop a pointing solution.

The KUKA robot cabinet contains motor drivers, an uninterruptible power supply, an embedded computer with a solid-state drive, and a portable "pendant" console. The robot embedded computer has a set of templates in the KUKA-native robot language for executing observations, repointing, slewing to home/stow/maintenance positions, and warming up the mechanical systems. Motion parameters for the templates (such as the scan center and speed) are transferred on a DeviceNet network between the housekeeping computer and the KUKA robot controller. Once the program and its parameters are loaded, the housekeeping computer sets a run bit and the scan starts. The robot controller can be commanded to (a smooth) stop by setting a stop bit on DeviceNet. KUKA’s robot also produces a stream of telescope health information (its internal encoder and resolver readouts, motor currents and temperature) which are broadcast via udp and recorded by the housekeeping computer at 50 Hz. The motion control servo loop uses an identical set of 27-bit Heidenhain encoders along the same axes as the ACT housekeeping pointing encoders, but the KUKA data stream is asynchronous to the housekeeping system.

Data handling after acquisition
-------------------------------

Each on-site data acquisition computer has a server which can stream data (either in real-time, or in a seek mode) to multiple clients. These clients can be either merger processes or operators that display the data in real-time using kst. Each camera observation is keyed to an entry in the mySQL file database. A python front-end of the merger queries the database to determine which files still need to be merged and calls the main merger code (in C). For those data that need to be merged, the merger client connects to the camera and housekeeping data servers and requests the first ∼ 1 second of data for that time and seeks to find time stamps in common. It then proceeds through the camera data, associating each record with a record in the housekeeping data with the same stamp. The output merged data products are stored in a dirfile format where each directory contains one file for each channel and a format file which specifies the calibrations for the channels. To manage the large data volume, J. Fowler developed a lossless delta compression scheme for each channel to reduce the bit length of each record (assuming each channel’s signal exercises only a fraction of the dynamic range of the data type). The algorithm achieves a compression rate ∼ 3 times faster than gzip, and an uncompression rate that is similar to gzip. For the camera data, a savings of ∼ 60% is typical for the algorithm, compared to ∼ 10% for gzip. Housekeeping data typically achieve 80% and 70% for our algorithm and gzip, respectively.

Every computer runs a daemon process responsible for automatically transporting data from the telescope to the ground station and then to North America. Data are automatically relocated from the site computers to RAID storage in the ground station, and then removed from the site computers when the database has identified that they are older than three days and that they have been merged. Data in the ground station are also moved automatically to transfer disks, and then deleted after they have been received in Princeton. Throughout, md5 checksums confirm that the data were compressed, transferred, and transported properly.

amcp:
-----

* `amcp` (c/h) main
* `bbc` (c/h) BLAST bus controller
* `cal.h` calibrations for various housekeeping readouts
* `control` (c/h) Interface to the outside world
* `control_struct` (c/h) Specification to the interface
* `enc` (c/h) Encoder readout
* `extern.h` is the header for things in the `_struct.c` files
* `frame` (c/h) Build and write data frames
* `frame_struct.c` Specification to build the data frames
* `io` (c/h) handle IO
* `io_struct.c` Specifcation for the IO
* `isc_protocol.h` star camera protocol
* `kuka` (c/h) Interface to the KUKA telescope robot controller
* `kuka_struct.h` Specification for the KUKA bits etc.
* `lut` (c/h) Handle look-up tables
* `pointing` (c/h) Determine the motions of the telescope
* `sensoray` (c/h) talk to Sensoray hardware like the ABOB
* `servo` (c/h) thermal control and cycling
* `servo_struct.h` specification for the servos
* `starcam` (c/h) star camera interface
* `weather` (c/h) interface to the WeatherHawk
* `window` (c/h) control the telescope cover window

The core pieces of the software are control, frame, and io. The rest plus into these and deal with various systems.
