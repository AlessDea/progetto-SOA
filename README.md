# SOAproject
## Block-level data management service
This specification is related to a Linux device driver implementing block-level maintenance of user data, in particular of user messages. A block of the block-device has size 4KB and its layout is organized as follows:
- the lower half (X bytes) keeps user data;
- the upper half (4KB-X bytes) keeps metadata for the management of the device.

Clearly, the lower the value of X the better since larger messages can be actually managed by software.

The device driver is essentially based on system-calls partially supported by the VFS and partially not. The following VFS non-supported system calls are requested to be designed and implemented:

- `int put_data(char * source, size_t size)` used to put into one free block of the block-device size bytes of the user-space data identified by the source pointer, this operation must be executed all or nothing; the system call returns an integer representing the offset of the device (the block index) where data have been put; if there is currently no room available on the device, the service should simply return the ENOMEM error;
- `int get_data(int offset, char * destination, size_t size)` used to read up to size bytes from the block at a given offset, if it currently keeps data; this system call should return the amount of bytes actually loaded into the destination area or zero if no data is currently kept by the device block; this service should return the ENODATA error if no data is currently valid and associated with the offset parameter.
- `int invalidate_data(int offset)` used to invalidate data in a block at a given offset; invalidation means that data should logically disappear from the device; this service should return the ENODATA error if no data is currently valid and associated with the offset parameter.

When putting data, the operation of reporting data on the device can be either executed by the page-cache write back daemon of the Linux kernel or immediately (in a synchronous manner) depending on a compile-time choice.

The device driver should support file system operations allowing the access to the currently saved data:

- `open` for opening the device as a simple stream of bytes
- `release` for closing the file associated with the device
- `read` to access the device file content, according to the order of the delivery of data. A read operation should only return data related to messages not invalidated before the access in read mode to the corresponding block of the device in an I/O session.

The device should be accessible as a file in a file system supporting the above file operations. At the same time the device should be mounted on whichever directory of the file system to enable the operations by threads. For simplicity, it is assumed that the device driver can support a single mount at a time. When the device is not mounted, not only the above file operations should simply return with error, but also the VFS non-supported system calls introduced above should return with error (in particular with the ENODEV error).

The maximum number of manageable blocks is a parameter NBLOCKS that can be configured at compile time. A block-device layout (its partition) can actually keep up to NBLOCKS blocks or less. If it keeps more than NBLOCKS blocks, the mount operation of the device should fail. The user level software to format the device for its usage should also be designed and implemented.