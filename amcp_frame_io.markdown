Overview:
=========

`frame.c`
=========
This provides utilities for assembling a data frame (`build_frame()`), pushing data to it (`map`), and saving the frames `push_frame_to_disc()`.

Maps
----

The map specifies the location of each piece of data in the `frame`. The map is built out of the specification for each device in `io_struct.c`.

Access to the data in the frame is provided by `map_XXX_to_frame`. For each system, this calls `map_to_frame` for the map for that system `XXX_map` and a position within a frame and page. Internally, this does `frame[page][map->start + map->size * realpos]`, where realpos is the position to the block in the frame where the data are.

`build_frame()`
---------------

This assembles the output data frame. Procedure:

1. Count up the number of control variables that are relays (`num_ctrl_cmd_param()`) and will be put in bit fields.  Find the rate, taken to be the maximum rate requested for any servo (`top_ctrl_cmd_bit_rate`). The on-off bits are written in bitfields of two bytes each. Finally, make a list (`all_ctrl_cmd_period`) of all the rates as `PULSE_RATE` divided by the rate for that bit e.g. the number per data frame.
2. Allocate the maps using `init_map()`: takes a pointer to a `frame_map_t` it then allocates `num` frame maps which are all set to start at -1 (to later check if the field has not been touched), returning the pointer.
3. Assemble the data frame. The first 4 bytes are a counter and start-of-frame word. (so `frame_len` = 4). For each data entry in the frame from a given data source, loop through the number of entries using `num_XXX()`, set the `start` to -1, and call `map_field()`. `map_field` takes a pointer to a `frame_map_t` and assigns the data `start` to `frame_length` and the write-check field `check_start` start to `frame_check_len`. The number `perframe` must be a multiple of the `PULSE_RATE`. Also check if the data size is consistent with the `frame_map_t` size. The `period` is set to the ratio of the `PULSE_RATE` to `perframe`. Copy the data field name to the frame. Push to `frame_len` ahead by the number of data per frame times their size, and push the `frame_check_len` ahead by just the number of datum per frame. Only data in the io structs which are read from active hardware are allocated.
4. Assemble the control parameter section of the frame. Rather than use `map_field`, build this directly in the function: 1) calculate the rate and period, 2) copy the name over, 3) copy over the location to point to fir this piece if data in the frame. For relays this is assumed to be 2 bytes. If it is not an on/off type (a relay) (TODO: what is that, then), assume it has size 4 and put -1 in its `ctrl_cmd_bit_map` entry, which is usually the bit number for a bitfield.

Now the frame is assembled, but the "spec" for the frame needs to be built from it. `num_internal_spec` is the number of channels in the frame and `frame_len` is the total size in bytes. Create a new`internal_spec` array, where each entry for a channel is a `frame_spec_t`. Procedure:

1. copy the version, `ACT_SPEC_VERSION`
2. For each data source, call `make_raw_spec`. This applies only to input data, and copies over the data type, rate, units and field name. The procedure is more complicatied for the KUKA bit fields, and constructs the bitfield description for each bit.
3. Add control parameters to the output spec.

Finally, allocate the frame. First make `NUM_FRAMES` (`frame.h`) pointers to the frame and frame checks frame. For each of those, allocate the `frame_len` and `frame_check_len` (the actual data). `NUM_FRAMES` is the number of frames to keep in memory as a buffer. This means that a piece of data and idle for up to 5 seconds and still be written to disk before that frame is pushed out of memory.

derived fields
--------------

Supporting functions: The function `get_derived_field` takes a field index and depending on its type returns the field for each derived data type. `num_derived_field` has a static counter which sums up the number of derived fields. `get_derived_index`?

timing
------

* `PULSE_RATE` fastest rate in the frame, 100 Hz for the bbc and 400 Hz for encoders (frame.h)
* `CTRL_CMD_DEF_PERIOD` is the period (in sec) at which external commands are logged (io.h)
* `CTRL_CMD_DEF_PERFRAME` is the derived rate (in Hz) at which external commands are logged (io.h)

`clock_frame_check` has a static tracker of the previous time and frame page. If the page has not advanced after `CLOCK_FRAME_TIMEOUT`, then give a warning and reset the timer. (Otherwise reset the frame page and time tracker to the present value for next time the function is called.)

`clock_frame_start` locks the thread. If the position in the frame exceeds the `PULSE_RATE`, restart the frame position. If the frame page exceeds `NUM_FRAMES`, reset the frame page. Return the position and the page.

`clock_frame_end` finishes the cycle by writing out the control command values using `write_ctrl_cmd_vals` and the kuka values using `write_kuka_vals`. It then unlocks the locking. If the frame position has reached `DISC_PUSH_INDEX=40` (TODO, val?), then call `push_frame_to_disc` to write out. Note that `NUM_FIRST_SKIPS` frames are discarded to allow for initial synchronization.

output
------
`push_frame_to_disc` writes the frame. The first four bytes are the `START_OF_FRAME` signature and counter. Call `check_frame_field` on all of the bbc and abob channels to count up the number of entries missing. Set the flag for the submit interpreter heartbeat (TODO more). Write 5 frames to a given chunk output file. If the number of frames in the file exceeds `MAX_FRAMES_IN_FILE=(15 * 60)` (15 minutes) then close the flat file and start a new output file.

`new_frame_file` generates a pointer to a rolling file output of frame files. In the first call, it makes the directory for `DATA_DIR/RAW_DIR/time` and writes the spec file (interal spec, and then derived fields) and the log file (pointed to by `message_add_logfile`. On all calls it opens the field for a chunk of frames and sets a "cur" file pointing to the current frame data written to disk.

`check_frame_field` goes through and fills in the last good (received) value for each piece of data missing from the frame. It does this by calling `frame_check` for the entry. The total number of bad entries is returned.

Relevant structures
-------------------

In frame.h, `struct map_frame_t`
* `start`:
* `check_start`:
* `perframe`: data rate in Hz (frame is 1sec)
* `period`: the `PULSE_RATE` divided by the number per frame
* `size`: side of the data entries here
* `field_name`: descriptive name of the field

In io.h, `struct datum_t` (this is a unit of data for all IO structures.)
* `field`: descriptive name of the field
* `perframe`: data rate in Hz (number per frame)
* `rw`: char for `r` read, or `w` write

Notes
------

* TODO: the `CTRL_CMD_DEF_PERIOD` and `CTRL_CMD_DEF_PERFRAME` definitions are funny.
* TODO: fill in workings of derived fields in frame.c
* TODO: where is `clock_frame_check` used?
* TODO: write out more details of the internal spec construction for KUKA and control bitfields.
* TODO: Why does this needs to be called after `kuka_init`.
* TODO: where is `frame_spec_t` defined? `act_util`? write this definition out.
* TODO: should `last_frame_check_len = frame_len;` be `last_frame_check_len = frame_check_len;` in lin 228 of `frame.c`: YES, fixed.


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

