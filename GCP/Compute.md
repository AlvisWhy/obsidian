DIsk expansion no need to reboot, just use resize2fs could resize, but could not shrink the volumn

Standard PD, pd-balanced, pd-ssd, pd-extreme, hyperdisk
local-ssd(good performance but not long existing)
For extreme, IO could be set separately, others IO grows with storage.

Cloud Shell provisions 5 GB of free persistent disk storage mounted as your $HOME directory on the virtual machine instance. This storage is on a per-user basis and is available across projects. Other dir files will be deleted after the session

APP engine
standard: run in sandbox. Can scale to 0
flexible: run in docker, in VM. So can not scale to 0.

Disk image/ Snapshot
Vm template/ backup

sole tenant: Node group, Node affinity

Autoscale policy
memory usage only refer to used in, but in reality the buffered + cached + slab will not be counted as default. So should use used+ buffered + cached + slab.
Target type:
Gauge:  snapshot value,
Dleta: val in the time window,
cumulative

VM manager + OS Config Agent = batch patching
Ops agent: collect logs

Disk could directly create image.
Disk -> snapshot -> image: Decrease the effect on the original disk