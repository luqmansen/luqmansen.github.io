by Edward Sciore

My personal note during reading this book.

# Chapter 3: Disk and File Management
- Abstracts 2 OS interfaces: block-level and file-level

> Platter capacities of over 40 GB are now common

This book was published in 2008 that HDD of this size was uncommon 

>access time for 1000 bytes is essentially the same as for 1 byte. In other words, it makes no sense to access a few bytes from disk

Still relevant even for modern day SSD

>The real value of a disk cache is its ability to pre-fetch sectors. Instead of reading
just a requested sector, the disk drive can read the entire track containing that sector
into the cache, in the hope that other sectors of the track will be requested later

Even modern OS's also do the same. So, multiple level of pre-fetching, will it be redundant or make read even more faster? 

>The set of tracks having the same track number is called a cylinder, because if you
look at those tracks from the top of the disk, they describe the outside of a cylinder.

it's nice to know the origin of a given term

>The term disk striping comes from the following imagery: If you imagine that each small disk is painted in a different color, then the virtual disk looks like it has stripes, with its sectors painted in alternating colors
 
also this <img src="https://emoticons.assets.hzchu.top/emoticons/Blob/blobcatphoto.png" alt="Blob_blobcatphoto" title="Blob_blobcatphoto" class="emoji-image m-0" width=25px style="margin: 0px;">

## The Block-Level Interface to the Disk

Sectors -> block  XXX - Cannot directly access it
Sectors -> memory page -> block

> To modify the contents of a block, a client must read the block into a page, modify the bytes in the page, and then write the page back to the block on disk

That's why we want to maximize each operation

Tracking available block
1. Disk Map
	Basically bit map
	{0, 0, 0, 0, 1 , 1, 1, 1}
	0 = block is free
	1 = occupied
	
2. Free List
	Not going to go to the technical details, but i just hate this kind data structure <img src="https://emoticons.assets.hzchu.top/emoticons/Blob/blobcatnotlikethis.png" alt="Blob_blobcatnotlikethis" title="Blob_blobcatnotlikethis" class="emoji-image m-0" width=25px style="margin: 0px;"> 
	
	The idea that, to allocate some N blocks, you have to keep traversing until you find enough space... 
	![[free-list.png]]
	It is space efficient, but the user pays somewhere the cost somewhere else. 
	To me, diskmap seems to be an easier and more convenient choice (for the programmer... <img src="https://emoticons.assets.hzchu.top/emoticons/Blob/blobcatpeekaboo.png" alt="Blob_blobcatpeekaboo" title="Blob_blobcatpeekaboo" class="emoji-image m-0" width=25px style="margin: 0px;"> )

## The File-Level Interface to the Disk

File abstraction 
> a client can read (or write) any number of bytes starting at any position in the file.

>The Java class RandomAccessFile provides a typical API to the le system. Each RandomAccessFile object holds a file pointer that indicates the byte at which the next read or write operation will occur.
>This file pointer can be set explicitly by a call to `seek`. A call to the method readInt (or writeInt) will also move the le pointer past the integer it read (or wrote).

IMO, `RandomAccessFile` is a bad/expensive abstraction i.e.  It gives you the idea that you can read/write byte at a time, when actually it will perform single syscall.

Ideally, wrap it in a buffer that will pull one page at a time.

```rust
use std::fs::File;
use std::io::BufReader;
use byteorder::{BigEndian, ReadBytesExt};

fn main() -> std::io::Result<()> {
    let file = File::open("data.bin")?;
    // Wraps file in an 8KB RAM buffer
    let mut reader = BufReader::new(file);
    // This first call pulls a full 8KB page into RAM from disk
    let first = reader.read_i32::<BigEndian>()?;
    // Subsequent reads hit the RAM buffer instantly—zero syscalls!
    let second = reader.read_i32::<BigEndian>()?;

    Ok(())
}
```

For write, there's also a write buffer, but I think we'll take that into account on buffer management chapter.

> Note that the calls to readInt and writeInt act as if the disk were being
accessed directly, hiding the fact that disk blocks must be accessed through pages.
An OS typically reserves several pages of memory for its own use; these pages are
called I/O buffers. When a file is opened, the OS assigns an I/O buffer to the file,
unbeknownst to the client.

Oh nice, the book also calling it out 

File `seek`'s task

1. convert byte to logical block ref (eg: byte pos 6000, is on block 1, i.e., 6000// mod 4k block size, is 1). This is easy
2. Covert logical block (eg: 1) into an actual physical reference (hard)

3 methods of file allocation
1. Continous allocation
	 - simplest
	 - suffer from internal fragmentation, (i.e. can only work with contiguous location. Cannot be extended once it's created, so it allocates large segment that might not be used) 
	 - and external frags (i.e. because of prev issue, you cannot create bigger file because empty space are scattered anywhere, despite being a lot of them)
 2. Extend based alloc
	 in some sense, this is almost works similarly with free list above?
 3. Indexed allocation
	 this is like a lookup index that give map of block. Neat in the idea, but that means a single file can be scattered across blocks? I wonder how OS can optimize this on a spinning disk.
	 ![[indexed-alloc.png]]


## The Database System and the OS

<img src="https://emoticons.assets.hzchu.top/emoticons/Blob/blobcatread.png" alt="Blob_blobcatread" title="Blob_blobcatread" class="emoji-image m-0" width=25px style="margin: 0px;"> 
