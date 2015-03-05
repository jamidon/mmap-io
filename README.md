# mmap for io.js / Node.js
mmap/madvise/msync for io.js / node.js revisited.

I needed shared memory mapping and came across @bnoordhuis module [node-mmap](https://github.com/bnoordhuis/node-mmap), only to find that it didn't work with later versions of io.js (and compatibles), so out of need I threw this together along with the functionality I found was missing in the node-mmap: advice and sync.

Strong temptations to re-order arguments to something more sane, was kept at bay, and I kept it as mmap(2) and node-mmap for compatibility. Notable difference is the additional optional argument to pass a usage advise in the mapping stage. I've given advise and sync more practical arguments, out of an io.js perspective, than their C/C++ counterparts.

The flag constants have the same crooked names as in C/C++ to make it straight forward for the user to google the net and relate to man-pages.

This is my first io.js addon and after hours wasted reading up on V8 API I luckily stumbled upon [Native Abstractions for Node](https://github.com/rvagg/nan). Makes life so much easier. Hot tip!

It's written in C++11 and [LiveScript](https://github.com/gkz/LiveScript).

It should be noted that mem-mapping is by nature potentially blocking, and _should not be used in concurrent serving/processing applications_, but rather has it's niche where multiple processes are working on the same giant sets of data (thereby sparing physical memory, and load times if the kernel does it's job for read ahead), preferably multiple readers and single or none concurrent writer, to not spoil the gains by shitloads of mutexes. And your noble specific use case ofcourse.


# Log

### 2015-03-04: version 0.1.0
This is the first commit, and the code has one day of development put into it as of now. More tests are needed so don't count on it being production ready just yet (but soon).


# Install
```
npm install mmap-io
```

or as git clone:
```
git clone https://github.com/ozra/mmap-io.git
cd mmap-io
make
```

For dev'ing, there are some shell-scripts for just building individual parts, the Makefile is more of convenience and does the whole hoola baloo including configuring.



# Usage

**Note: All code in examples are in LiveScript**

```livescript

# Following code is plastic fruit; not t[ae]sted...

mmap = require "mmap-io"
fs = require "fs"

some-file = "./foo.bar"

fd = fs.open-sync some-file, "r"
fd-w = fs.open-sync some-file, "r+"

# In the following comments, `[blah]` denotes optional argument,
# `foo = x` denotes default values for arguments

# map( size, protection, privacy, fd [, offset = 0 [, advise = 0]] ) -> Buffer

size = fs.stat-sync(fd).size
rx-prot = mmap.PROT_READ .|. mmap.PROT_EXECUTE
priv = mmap.MAP_SHARED

buffer = mmap.map size, rx-prot, priv, fd
buffer2 = mmap.map size, mmap.PROT_READ, priv, fd, 0, mmap.MADV_SEQUENTIAL
w-buffer = mmap.map size, mmap.PROT_WRITE, priv, fd-w

# advise( buffer, advise ) -> void
# advise( buffer, offset, length, advise ) -> void

mmap.advise w-buffer, mmap.MADV_RANDOM

# sync( buffer ) -> void
# sync( buffer, offset, length ) -> void
# sync( buffer, is-blocking-sync[, do-page-invalidation = false] ) -> void
# sync( buffer, offset = 0, length = buffer.length [, is-blocking-sync = false [, do-page-invalidation = false]] ) -> void

w-buffer[47] = 42
mmap.sync w-buffer
mmap.sync w-buffer, true
mmap.sync w-buffer, 0, size
mmap.sync w-buffer, 0, size, true
mmap.sync w-buffer, 0, size, true, false

# Yeah, you will do _one_ of the variants ofcourse..

```

### Good to Know (TM)

- Checkout man pages mmap(2), madvise(2) and msync(2) for more detailed intell.
- The mappings is automatically unmapped when the buffer is garbage collected.
- Write-mappings need the fd to be opened with "r+", or you'll get a permission error (13).
- If you make a read-only mapping and then ignorantly set a value in the buffer, all hell previously unknown to a JS'er breaks loose (segmentation fault). It is possible to write some devilous code to intercept the SIGSEGV and throw an exception, but let's not do that!
- `Offset`, and in some cases `length` needs to be a multiple of mmap-io.PAGESIZE (which commonly is 4096)


# Todo, Not Todo and Stuff
- More tests
- Huge pages are only supported for anonymous / private mappings (well, in Linux), so I didn't throw in flags for that since I found no use.
- As Ben Noordhuis previously has stated: Supplying hint for a fixed virtual memory adress is kinda moot point in JS, so not supported.
- If you miss a feature - contribute! Or request it in an issue. I might add it. I intend to maintain this module since it will be part of a great deal of code I'm working on.
- If documentation isn't clear, make an issue.
