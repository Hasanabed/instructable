### How to compact VirtualBox Linux Guest:

## On Guest:

sudo dd if=/dev/zero of=/bigemptyfile bs=4096k status=progress

sudo rm -f /bigemptyfile

## On Host:

Close VM and then do a "Clone" of it in VirtualBox.

This will create a clone that is compacted.
