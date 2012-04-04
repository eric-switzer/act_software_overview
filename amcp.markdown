Overview:
=========

Central rates
-------------
* `PULSE_RATE` fastest rate in the frame, 100 Hz for the bbc and 400 Hz for encoders (frame.h)
* `CTRL_CMD_DEF_PERIOD` is the period (in sec) at which external commands are logged (io.h)
* `CTRL_CMD_DEF_PERFRAME` is the derived rate (in Hz) at which external commands are logged (io.h)

TODO: the `CTRL_CMD_DEF_PERIOD` and `CTRL_CMD_DEF_PERFRAME` definitions are funny.

`amcp.c`
======
`int main(int argc, char *argv[])`
`void exit_handler()` -- ref in main using `set_exit_subroutine`
`void ctrlc_handler(int signum)` -- ref in main signal handler

`bbc.c`
=====
`void bbc_init()`
`void *bbc_thread(void *arg)`
`void write_bbc_to_frame(unsigned int outval, int index, int pos, int page)`; used in thread
`void write_to_nios(int bbcfd, unsigned int addr, unsigned int datum)`; used in init and thread

`frame.c`
=========
The central functions here are `build_frame()` ...


`build_frame()`
---------------
This assembles the output data frame.

1. Count up the number of control variables that are relays (`num_ctrl_cmd_param()`) and will be put in bit fields.  Find the rate, taken to be the maximum rate requested for any servo (`top_ctrl_cmd_bit_rate`). The on-off bits are written in bitfields of two bytes each. Finally, make a list (`all_ctrl_cmd_period`) of all the rates as `PULSE_RATE` divided by the rate for that bit e.g. the number per data frame.
2. Allocate the maps using `init_map()`: takes a pointer to a `frame_map_t` it then allocates `num` frame maps which are all set to start at -1 (to later check if the field has not been touched), returning the pointer.
3. Start to assemble the frame. The first 4 bytes are a counter and start-of-frame word. (so `frame_len` = 4)


Notes:
This needs to be called after `kuka_init` (WHY).

`map_to_frame`: given a frame position 

A "map" points to information in the frame


`map_field` takes a pointer to a `frame_map_t` and assigns the data `start` to `frame_length` and the write-check field start to `frame_check_len`. The number `perframe` must be a multiple of the `PULSE_RATE`. Also check if the data size is consistent with the `frame_map_t` size. The `period` is set to the ratio of the `PULSE_RATE` to `perframe`. Copy the data field name to the frame. Push to `frame_len` ahead by the number of data per frame times their size, and push the `frame_check_len` ahead by just the number of datum per frame.

`frame.h`
=========

`struct map_frame_t`
---------------------
* `start`:
* `check_start`:
* `perframe`: data rate in Hz (frame is 1sec)
* `period`: the `PULSE_RATE` divided by the number per frame
* `size`: side of the data entries here
* `field_name`: descriptive name of the field

`io.h`:
=======

`struct datum_t`
----------------
This is a unit of data for all IO structures.
* `field`: descriptive name of the field
* `perframe`: data rate in Hz (number per frame)
* `rw`: char for `r` read, or `w` write

