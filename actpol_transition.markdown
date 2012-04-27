Summary:
========

* The encoder driver `ik220`, BLAST bus card (BBC) driver `bbcpci` and polling code `act_timing` needed to be updated to recent linux kernels. These are still 32-bit.
* The `interface_server` has been replaced with `redis` (a key-value store with publish/subscribe). The master shelve file which previously specified commands has been replaced with a JSON file available on the web. The housekeeping system parses this file into its `control_struct.c/h.`.
* The old Tkinter/Pmw GUI is being replaced with wxpython which builds its interface from the JSON specification. It will talk to redis, but web commanding interfaces are also planned.
* The BBC infrastructure has been removed from amcp except for reading the serial number from the syncronization box. Note that the isolated IO stage of the PCI card needs to be powered independently.
* All of the relevant code from the feynman CVS server is revamped and now on and SVN.
