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
`map_to_frame`: given a frame position 

A "map" points to information in the frame

`init_map()` takes a pointer to a `frame_map_t` it then allocates `num` frame maps which are all set to start at -1 (to later check if the field has not been touched), returning the pointer.

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

