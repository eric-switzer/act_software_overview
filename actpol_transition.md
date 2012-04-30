Summary:
========

* The encoder driver `ik220`, BLAST bus card (BBC) driver `bbcpci` and polling code `act_timing` are updated to recent linux kernels (32 bit).
* The `interface_server` has been replaced with `redis` (a key-value store with publish/subscribe). The master shelve file which previously specified commands has been replaced with a JSON file available on the web. The housekeeping system parses this file into its `control_struct.c/h.`.
* The old Tkinter/Pmw GUI is being replaced with wxpython which builds its interface from the JSON specification. It will talk to redis, but web commanding interfaces are also planned.
* The BBC/Toronto infrastructure has been removed from amcp except for reading the serial number from the syncronization box. (working)
* All of the relevant code from the feynman CVS server is revamped and now on an SVN at UPenn.
* Integration of the thermometry thread is underway.

TODO:
=====
* get `appio` to compile under new kernels (DeviceNet/KUKA comm.)
* finish the wxpython commanding interface
* Mike Nolta to move software to the Penn SVN
* set up Trac on the SVN and confirm that it is backed up
* make a nice cable/power hydra for the sync box -> BBC PCI connection.
* replace the EEPROM or otherwise on the failed sensoray board (for panel temps)
* install the GPS card an discipline the clock
* install the drivers for USB national instruments hardware readout on borsNN
* continue memory issues checks in amcp `sudo valgrind --leak-check=full --tool=memcheck -v ./amcp`
* site work: get encoders + interal clocking + streaming
* site work: commit current version (fixed memory leak)
* write new amcp thread that reads from a buffer written by the thermometry thread
* tunnel the site redis through an ssh gateway

Installation (on older OSs):
============================
* Redis `wget http://redis.googlecode.com/files/redis-2.4.11.tar.gz` (see `http://redis.io/download`) untar it, then `make; sudo make install; cd utils; sudo ./install_server.sh`
* libevent `wget https://github.com/downloads/libevent/libevent/libevent-2.0.18-stable.tar.gz` (see `http://libevent.org/`) untar it, then `./configure; make; sudo make install`
* hiredis `wget https://github.com/antirez/hiredis/tarball/master` (see `https://github.com/antirez/hiredis/downloads`) untar it, then `make; sudo make install`
* run `sudo /sbin/ldconfig`
* if you do not have the housekeeping trunk `svn co http://actpolsvn.info/svn/actpol_software/housekeeping/trunk`
* in `/etc/redis.conf` (or similar) set `timeout 0` so that subscribed amcp/client sessions will not time out (this is not handled well yet)
