amcp:
=====
`int main(int argc, char *argv[])`
`void exit_handler()` -- ref in main using set_exit_subroutine
`void ctrlc_handler(int signum)` -- ref in main signal handler

bbc:
====
`void bbc_init()`
`void *bbc_thread(void *arg)`
`void write_bbc_to_frame(unsigned int outval, int index, int pos, int page)`; used in thread
`void write_to_nios(int bbcfd, unsigned int addr, unsigned int datum)`; used in init and thread

frame.c:
========
