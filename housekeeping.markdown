Housekeeping
============

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
