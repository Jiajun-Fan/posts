Disable journal on a HFS+ drive
=============================

I have a USB hard drive and I connected it to may MAC as external storage. Recently I want to connect this hard drive to my raspberry PI so my RP4 can works as NSA.

And soon I realized the problem:  
* The hard drive is formatted with HFS+ **JOURNALED**.
* The HFS+ JOURNALED file system is read only on Linux.
* Read-only file system is definitely unacceptable to me.

After a few googling, I understand that:
* HFS+ file system is writable only if journal is turned off.
* One can turn it off using the `disk utility` of MAC.

But I don't have my macbook by myside, it seems a dead end. So I was thinking can I turn it off by myself. Fortunately the answer this **YES**.

I found some perfect document of [HFS+ head](http://dubeyko.com/development/FileSystems/HFSPLUS/hexdumps/hfsplus_volume_header.html#attributes), it seems quite straight-forward.
There is an attribute (offset is 4) in HFS+ volume head, and the 14th bit indicate if journal is turned on.

|Bit|Attribute name|Description
|---|---|---
|0-7|Reserved|
|...|...|...|
|13|kHFSVolumeJournaledBit|If this bit is set, the volume has a journal.|
|...|...|...|

OK, then let's clear this bit.
```c
#include <stdio.h>
#include <assert.h>
#include <stdint.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

static const int kVolumeOffset = 1024;
static const int kHeadSize = 2048;
static const int kAttributeOffset = 4;

// Journal flag is the 14th bits
// Create mask depends on the host endian
static const uint32_t kLittleEndianMask = 0xFFDFFFFF;
static const uint32_t kBigEndianMask = 0xFFFFDFFF;

static int isLittleEndian() {
  union {
    uint32_t u;
    char c[4];
  } t;
  t.u = 0x1;
  return t.c[0];
}

int main(int argc, const char** argv) {
  // open the device specified the 1st argument
  // It's something like /dev/sda1 on linux
  int fd = open(argv[1], O_RDWR);
  assert(fd > 0);

  // mapping it into memory
  unsigned char* buff = (unsigned char*)mmap(NULL, kVolumeOffset+kHeadSize, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
  assert(buff != (unsigned char*)-1);

  // seek to HFS+ head
  unsigned char* data = buff + kVolumeOffset;

  // check head magic
  assert(data[0] == 'H' && data[1] == '+');

  // get attribute
  uint32_t attribute = *(uint32_t*)(data+kAttributeOffset);

  // clear the journal flag
  uint32_t mask = isLittleEndian() ? kLittleEndianMask : kBigEndianMask;
  attribute &= mask;
  *(uint32_t*)(data+kAttributeOffset) = attribute;

  // unmap device
  munmap(buff, kVolumeOffset+kHeadSize);

  // close the file descriptor
  close(fd);

  printf("Success!\n");

  return 0;
}

```

After disabling Journal and remounting the drive, it's writable as expected.
