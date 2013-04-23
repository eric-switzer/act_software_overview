`io_struct.c/io.h`
================
Specifies how names map to hardware channels on specific devices. The structures defining this map are in `io.h`. Standard types are for reading devices, writing to devices, and recording internal variables.

Data read from devices are stored to a binary frame file. The structure that maps an entry in the frame must specify a `datum_t` structure that describes the field name, data rate, whether it is read/write, and a single-char type string. In an example device, `bbcio_t`, this frame data spec is linked to a card and channel in the readout: 
```c
struct datum_t {
  char field[FIELD_LEN];        // name
  int perframe;                 // number of samples per frame
  char rw;                      // 'r' or 'w'
  //! 'c' = char, 's' = short, 'u' = unsigned short, 'S' = long,
  //! 'S' = unsigned long, 'f' = float, 'd' = double
  char type;
};
#define DATUM_END_OF_IO     {"", 'x', 'x'}

struct bbcio_t {
  struct datum_t datum;
  char card;
  char chan;
};
#define END_OF_BBCIO        {DATUM_END_OF_IO, -1, -1}
```
The `io.h` file also has specifications like the IP address of readout hardware, number of channels, etc. The `io_struct.c` file defines the mapping of each channel to frame data as
```c
struct bbcio_t bbcio[] = {
  {{"tr0100_ar3_lens_1k",    100, 'r', 'U'}, UT_CARD_DIODE, 0},
...
  END_OF_BBCIO
};
```

Commanding: The mapping structure for writing variables in `io_struct.c` ties controllable variable names (`ctrl_name`) to field names `field` on the hardware driver.

`frame.c/.h`
============
This provides utilities for assembling a data frame (`build_frame()`), pushing data to it (`map`), and saving the frames (`push_frame_to_disk()`).

The device channel gets associated with and entry in the frame using the map type: 
```c
struct frame_map_t {
  int start;                    // block of the frame where this channel starts (bytes from 0)
  int check_start;              // the number of this block in the frame
  int perframe;                 // number of this type of data per frame
  int period;                   // factor by which this data date is slower than the fastest
  int size;                     // size in bytes of each piece of data
  char field_name[FIELD_LEN];   // name of the data
};
```

A readout device can have many channels. Each channel is associated with a position in the frame by `map_field()` using an array of `frame_map_t`. This array is allocated for each device by `init_map()`, which initially flags all map entries with `start=-1` to indicate that they are not mapped to a position in the frame file.

To build the frame, go through and map each field in the readouts to a position of the frame. Some entries in the `io_struct` specification are for writing to devices, and are not written to a frame. In this case, they stay in the map, but have `start=-1` to indicate they should be ignored. As the loop goes through live channel to write to the frame, the position is incremented through `frame_len` and `frame_check_len`. The frame starts with a 4-byte header, so is initialized as `frame_len=4` but steps up as the data size times the number of dataum in that channel per frame. The `frame_check_len` starts from zero and counts the total number of data records (of any size) in the frame. `map_field()` takes a pointer to a `frame_map_t` and assigns the data `start` to `frame_length` and the `check_start` start to `frame_check_len`. `PULSE_RATE` is the rate in Hz of the fastest data in the frame. A given channel rate can be specified as a multiple factor (`period`) slower than the fastest rate. For example, if encoders are read at 400 Hz and the period is 4, a given channel will have a perframe of 100, or 100 Hz. In this case, the `io_struct` specifies the perframe and derives the period. The number `perframe` must be a multiple of the `PULSE_RATE`. Finally, `map_field()` copies the data field name to the frame map. Push `frame_len` ahead by the number of data per frame times their size, and push the `frame_check_len` ahead by just the number of datum per frame. Any inactive hardware (commented out of the frame struct) will not be registered as a map.

The control parameters (`ctrl_cmd_param[]`) are treated specially. On-off controlled variables are put into a 2-byte bitfield whose rate is the maximum rate (`top_ctrl_cmd_bit_rate`) of all the request on-off variables. These variables are not mapped with a `map_field()` call.

Now the frame is assembled, but the spec for the frame needs to be built from it. `num_internal_spec` is the number of channels in the frame. Create a new `internal_spec` array, where each entry for a channel is a `frame_spec_t`. Procedure:

1. copy the version, `ACT_SPEC_VERSION` and set comments to use `#`
2. For each data source, call `make_raw_spec()`. This applies only to input data, and copies over the data type, perframe rate, units and field name. Units are null and the spec type is 'r'. The procedure is more complicatied for the KUKA bit fields, and constructs the bitfield description for each bit.
3. Add control parameters to the output spec. If it is an on-off type count up the number of bitfields and register their spec.

Finally, allocate the frame. First make `NUM_FRAMES` (`frame.h`) pointers to the frame and frame checks frame. For each of those, allocate the `frame_len` and `frame_check_len` (the actual data). `NUM_FRAMES` is the number of frames to keep in memory as a buffer. This means that a piece of data and idle for up to 5 seconds and still be written to disk before that frame is pushed out of memory. The `frame_check[]` array is set to zero and once a channel is written to, set to one. This helps check that no devices are lagging in writing to the frame.

A given channel in a system may have e.g. 10 records in a frame, and it can write into one of `NUM_FRAMES` possible pages. Each system as a function that uses `map_to_frame()` to return a pointer to the correct absolution position in a frame page. It also flags that it has requested (and presumably written to) a given entry in that frame. 

`push_frame_to_disk()`
----------------------
The first four bytes are the `START_OF_FRAME` signature and counter. Call `check_frame_field` on all of the bbc and abob channels to count up and report the number of entries missing. Any other systems that could lag should be added to this. Set a flag to indicate whether the submit interpreter hearbeat is active. Write on page of the total `NUM_FRAMES`. If the number of frames in the file exceeds `MAX_FRAMES_IN_FILE=(15 * 60)` (15 minutes) then close the flat file and start a new output file.

`new_frame_file` generates a pointer to a rolling file output of frame files. In the first call, it makes the directory for `DATA_DIR/RAW_DIR/time` and writes the spec file (interal spec, and then derived fields) and the log file (pointed to by `message_add_logfile`. On all calls it opens the field for a chunk of frames and sets a cur file pointing to the current frame data written to disk.

`check_frame_field` goes through and fills in the last good (received) value for each piece of data missing from the frame. It does this by calling `frame_check` for the entry. The total number of bad entries is returned.

Other functions in `frame.c`: clocking on functions that write data to the frame, manage derived fields.

TODO: rename check_start to start_index?
TODO: where is the ACT_SPEC_VERSION?, `frame_spec_t`?
TODO: remove the control parameters from the frame, or at least the on-off bitframe
TODO: where are the derived fields registered in the spec file?

`io.c`
=====

`io.c` is mainly a set of support functions which provide links between various channel indices and sanity checks for the IO channel specifications. `io.h` specifies more generic properties of the readout systems like the number of cards, channels, IP addresses, etc.

`sanity_checks()` performs several tests to see that the `io_struct.c` and `frame_struct.c` specify valid data fields. This is called after `build_frame()` in amcp main(), but before the readout threads begin. Using helper functions below, this 1) checks field name lengths, 2) looks for duplicates, 3) checks that there are a sane number of bbc channels or card addresses, that each controllable bbc/abob channel on the bbc has a corresponding external control entry, and that all controllable quantities are saved in the output frame, 4) that all derived fields have some source. Helers:

* `check_duplicates` is a service function that check for duplicates in the field names specified in `io_struct.c`.
* `check_for_source` checks that derived fields are based on real underlying fields, either derived or raw
* `check_length` makes sure a field name is not too long

There are several service functions which translate between channels:

* `bbc_to_write_map()` and `write_map_to_bbc` connect each writable bbcio entry to its `bbcio_write_map` entry (which connects it to the controller). There is an analogous `write_map_to_sensoray`. `get_bbcio_index()` finds the BBC index associated with a channel on a given card.
* `get_bbcio_write_index()` compiles indices for a map from all channel indices to write indices. If the channel is not writable, then return -1.
* `get_bbcio_index_from_field()` is a fast way to go through and find the channel number associated with a field name.
* `get_bbcio_write_treatment()` identifies bbc channels that have a special treatment.
* `get_calib_info_from_bbc()` connects the bbc channel index with the calibrated temperature channels that are used for cycling/servo
* `bbc_to_cycle_ctrl_index()` `bbc_to_servo_index()`: if a BBC channel is used for cycling/servo, read it and apply the lookup to get temperature
* `get_ctrl_cmd_perframe_index()`: given a name, find the index in the control command (perframe?)
* `get_type_size()` gives the size in bytes for the various characters that label data types.

Data model
===========

the map
--------

The map specifies the location of each piece of data in the `frame`. The map is built out of the specification for each device in `io_struct.c`.

Access to the data in the frame is provided by `map_XXX_to_frame`. For each system, this calls `map_to_frame` for the map for that system `XXX_map` and a position within a frame and page. Internally, this does `frame[page][map->start + map->size * realpos]`, where realpos is the position to the block in the frame where the data are.

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

`clock_frame_end` finishes the cycle by writing out the control command values using `write_ctrl_cmd_vals` and the kuka values using `write_kuka_vals`. It then unlocks the locking. If the frame position has reached `DISK_PUSH_INDEX=40` (TODO, val?), then call `push_frame_to_disk` to write out. Note that `NUM_FIRST_SKIPS` frames are discarded to allow for initial synchronization.

Notes
=====

* TODO: larger structure, how is the clock defined (both bbc and enc call clock start/end), how is the rate set?
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

