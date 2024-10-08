PREFACE: I do NOT own any of the files included in this repo. All credit goes to the PostgreSQL team, I only changed the following files from THEIR product to implement 2nd-LRU instead of their Clock Sweep replacement policy.
Their work can be found at https://www.postgresql.org/

Brief Summary:
In this I changed several files: buf_internals.h, freelist.c, buf_init.c, and bufmgr.c. 
The goal of this was to implement a 2nd-LRU replacement policy. In order to do this, I had to assign each buffer its own timestamp, as well as
create a global timestamp to initialize that held an integer value. Whenever refcount was incremented, the buffer that was read in gets its value
set to the global timestamp and then the global timestamp is incremented. We also implemented 2nd-LRU in freelist, this policy works by taking the second 
least recently used buffer and evicting it from the system.

Detailed Changes:
1. buf_init.c: /src/backend/storage/buffer/buf_init.c

All I changed here was on line 131, added
buf->timestamp = 0;
This initialises the timestamp in each buffer to 0

2. freelist.c: /src/backend/storage/buffer/freelist.c

Several lines were added for this. Starting with the function that was implemented on line 195. This is the actual 2nd-LRU function. It reads in all of the
buffers provided, and sets their timestamp equal to the global_timestamp. It then iterates through all of those buffers, finds the buffer with the 
second lowest number on its timestamp, and "evicts it" (prints it to the server terminal). It also prints all of the candidates to be evicted to the server
terminal. It then returns the bufID that was evicted.

Also changed was the line on 390
	buf = GetBufferDescriptor(LRU2()); This used to use clock sweep, but the point was to replace clock sweep with LRU2, so we switched out the functions.

3. bufmgr.c: /src/backend/storage/buffer/bufmgr.c

Added the lines

int global_timestamp = 0;
on line 97, which initialises the global_timestamp to 0

	GetBufferDescriptor(buffer) -> timestamp = global_timestamp;
	int global_timestamp = global_timestamp + 1;
on lines 4521 and 4522

This sets the timestamp for each buffer that is incremented when it is pinned by the refcount equal to the global timestamp that we initialized on
line 97. 

I also added the same change (with the exception of using b instead of buffer, since that was the buffer in those two methods) in the pin and unpin methods

4. buf_internals.h: /src/include/storage/buf_internals.h

I added 	int	timestamp; in bufferdesc on line 252. This adds a timestamp variable to every buffers header
