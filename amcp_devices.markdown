Overview
========

Synchronous devices are `bbc` and `enc`, while asynchronous devices/processes are `kuka/pointing`, `sensoray`, `servo`, `starcam`, `weather`, `window`. For an example of how asynchronous data is inserted into the frame, see `sensoray_thread`.

`sensoray.c`
============

The data are grouped into a number of channels per port. The maximum read frequency on the device is `max_sensoray_freq = PULSE_RATE/max_sensoray_perframe`

`sensoray_thread()`
-------------------
Wait until the frame position counter is between 1000 and 1001 times the maximum sensoray read period.

