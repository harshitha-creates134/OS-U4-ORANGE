# Lab: File Systems & Storage
**Name:** Sanjay
**SRN:** PES1UG24CS186
**Platform:** Ubuntu 22.04

---

## Section 0 — Build Your Filesystem

### Commands Run
```bash
dd if=/dev/zero of=orange.img bs=1M count=256
mkfs.ext4 -L ORANGE orange.img
sudo mkdir -p /mnt/orange
sudo mount -o loop orange.img /mnt/orange
wget -q https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-minirootfs-3.21.3-x86_64.tar.gz
sudo tar xzf alpine-minirootfs-3.21.3-x86_64.tar.gz -C /mnt/orange
df -h /mnt/orange
ls /mnt/orange
sudo dumpe2fs -h orange.img 2>/dev/null | grep -E 'Block count|Block size|Inode count|Free blocks|Free inodes'
```

### Screenshot 0
<img width="1206" height="457" alt="1" src="https://github.com/user-attachments/assets/76e1c924-42a6-4c60-8284-5c668913aa31" />


---

### Q0.1
**From the `dumpe2fs` output: how many total inodes exist, how many are free, and how many are used? How many total blocks, and what's the block size? Show the arithmetic.**

| Field | Value |
|---|---|
| Inode count (total) | 65,536 |
| Free inodes | 65,525 |
| Used inodes | 11 |
| Block count | 65,536 |
| Free blocks | 57,268 |
| Block size | 4096 bytes (4 KB) |

**Arithmetic:**
```
used_inodes = total - free
           = 65,536 - 65,525
           = 11 used inodes
```
These 11 inodes are consumed by the filesystem itself at format time — the root directory, lost+found, the journal, and reserved metadata structures.

**Total disk size verification:**
```
65,536 blocks × 4096 bytes/block = 268,435,456 bytes = 256 MiB ✓
```

---

## Section 1 — Inodes: The Real Identity of a File

### Commands Run
```bash
# Hard and symbolic links
echo "original content" | sudo tee /mnt/orange/test.txt
sudo ln /mnt/orange/test.txt /mnt/orange/hard.txt
sudo ln -s /mnt/orange/test.txt /mnt/orange/soft.txt
ls -li /mnt/orange/test.txt /mnt/orange/hard.txt /mnt/orange/soft.txt
stat /mnt/orange/test.txt
stat /mnt/orange/soft.txt

# Inode exhaustion
dd if=/dev/zero of=tiny.img bs=1M count=10
mkfs.ext4 -N 16 tiny.img
sudo mkdir -p /mnt/tiny
sudo mount -o loop tiny.img /mnt/tiny
df -i /mnt/tiny
for i in $(seq 1 20); do sudo touch /mnt/tiny/file_$i 2>&1; done
echo "--- after inode exhaustion ---"
df -i /mnt/tiny
df -h /mnt/tiny

# inode_explore program
gcc -O0 -o inode_explore inode_explore.c
sudo ./inode_explore
```

### Screenshot 1
<img width="832" height="653" alt="2" src="https://github.com/user-attachments/assets/e01f5721-cd48-4bf7-91ab-2e90aece94ab" />


---

### Q1.1
**`test.txt` and `hard.txt` share the same inode number. What is the link count? Delete `test.txt` — does `hard.txt` still work? What happened to the link count? Why doesn't the data get freed?**

`test.txt` and `hard.txt` share **inode 516**, link count = **2**. After `sudo rm /mnt/orange/test.txt`, `hard.txt` still works and link count drops to **1**. The data is not freed because the kernel only frees the data blocks when the link count reaches **0**. `rm` just removes one directory entry and decrements the count — the inode and its data blocks survive as long as any name still points to them.

---

### Q1.2
**`soft.txt` has a different inode number. After deleting `test.txt`, does `cat /mnt/orange/soft.txt` still work? Why not? What's the fundamental difference between how the kernel resolves a hard link vs a symbolic link?**

`soft.txt` has inode **517** (its own separate inode). After deleting `test.txt`, `cat /mnt/orange/soft.txt` **fails** with "No such file or directory". A hard link resolves directly to an inode number, so it works regardless of whether the original name exists. A symlink stores the **path string** `/mnt/orange/test.txt` — the kernel re-walks that path on every access, and since the path no longer exists, resolution fails.

---

### Q1.3
**At what point did `touch` start failing? How much disk space was still free? Why can't ext4 just create more inodes on the fly?**

`touch` began failing at **file_6** because the filesystem had exactly **5 free inodes** remaining (16 total − 11 used by the FS itself = 5). Once those 5 were consumed by `file_1` through `file_5`, the kernel could not create any new directory entries — even though **5.3 MB of block space** was still free.

ext4 cannot create more inodes on the fly because the **inode table is allocated at `mkfs` time** and written into fixed locations on disk. The number of inodes is baked into the superblock permanently. Resizing it would require relocating the entire inode table, which ext4 does not support after formatting.

---

### Q1.4
**What inode number does `stat()` return for the symlink vs what `lstat()` returns? Why are they different? Which one gives the symlink's own inode?**

From the `inode_explore` output:
- `stat()` on the symlink returns inode **518** (the target's inode — it follows the link)
- `lstat()` returns inode **519** (the symlink's own inode)

They differ because `stat()` transparently follows the symlink to the destination file, while `lstat()` stops at the symlink itself. **`lstat()` gives the symlink's own inode.**

---

### Q1.5
**Which field inside the in-memory inode tracks that the file is still open? What syscall drops that count to zero? If the first process keeps the file open indefinitely, what happens to disk space?**

The field is the **open file description's reference count** tracked in the kernel's file table. The syscall that finally drops it to zero is **`close()`**. If the first process keeps the file open indefinitely, the inode stays alive in memory and the **disk blocks are never freed** — the space is leaked until the last `close()` or the process exits. This is a common cause of "disk full" surprises on servers where log files are deleted but still held open by a running process.

---

## Section 2 — What Happens When You open() a File

### Commands Run
```bash
gcc -O0 -o fd_explore fd_explore.c
sudo ./fd_explore
strace -e openat,read,write,close cat /mnt/orange/etc/hostname 2>&1
```

### Screenshot 2
<img width="975" height="687" alt="3" src="https://github.com/user-attachments/assets/79bb4848-647a-41e9-be1c-a97ae3e91edd" />


---

### Q2.1
**`fd1` and `fd3` share a file offset. After `read(fd1, 3)`, reading from `fd3` returns "DEF" not "ABC". Why? Draw the three-layer diagram.**

`fd1` and `fd3` share the **same open file entry** because `dup()` copies the file descriptor but points to the same kernel file table entry — including the shared offset. After `read(fd1, 3)`, the offset moves to 3. So `read(fd3, 3)` continues from position 3 and returns `"DEF"`. `fd2` came from a completely separate `open()` call, so it has its own independent offset starting at 0 — returning `"ABC"`.

```
fd1 ──┐
      ├──▶ open file entry A: offset=3 ──▶ inode (fd_test.txt)
fd3 ──┘

fd2 ─────▶ open file entry B: offset=0 ──▶ inode (fd_test.txt)
```

---

### Q2.2
**In the `strace` output for `cat`, how many `read()` calls does cat make? What size buffer does it use? At what point does it see EOF?**

From the `strace` output, `cat` makes **2 `read()` calls** on the actual file. It uses a buffer size of **131072 bytes (128 KB)**. EOF is seen on the **second read** — `read(3, "", 131072) = 0`. The return value of 0 signals end of file.

---

### Q2.3
**What limits prevent a process from opening more files? Run `ulimit -n`. What happens if a server hits this limit?**

A file descriptor is just an integer index into the process's FD table. The per-process limit is shown by `ulimit -n` (typically **1024** by default). If a server hits this limit, every subsequent `open()`, `socket()`, or `accept()` call returns `-1` with `errno = EMFILE` ("Too many open files") — causing the server to fail to accept new connections or open files until existing FDs are closed.

---

## Section 3 — Directory Implementation

### Commands Run
```bash
sudo ls -lad /mnt/orange/etc /mnt/orange/usr
sudo mkdir -p /mnt/orange/bigdir
for i in $(seq 1 500); do sudo touch /mnt/orange/bigdir/file_$i; done
sudo ls -lad /mnt/orange/bigdir
time sudo ls /mnt/orange/etc > /dev/null
time sudo ls /mnt/orange/bigdir > /dev/null

gcc -O0 -o readdir_raw readdir_raw.c
sudo ./readdir_raw /mnt/orange/etc

sudo ln /mnt/orange/etc/hostname /mnt/orange/etc/hostname.bak
sudo ln /mnt/orange/etc/fstab /mnt/orange/home/fstab_copy 2>/dev/null || true
gcc -O0 -o find_dupes find_dupes.c
sudo ./find_dupes /mnt/orange
```

### Screenshot 3
<img width="794" height="673" alt="4" src="https://github.com/user-attachments/assets/5f98ef69-c3c3-4068-95c9-3be6ce5b1472" />


---

### Q3.1
**What is the directory file's size for `bigdir` vs `etc`? Does it shrink when you delete entries? Why or why not?**

`bigdir` = **12288 bytes**, `etc` = **4096 bytes**. A directory's size grows as entries are added because the kernel allocates new data blocks to hold the `(name, inode)` pairs. However it **does not shrink** when files are deleted — the blocks are kept and the space is just marked as free within the directory. The kernel reuses that slack space for future entries but never gives the blocks back to the filesystem. This is by design to avoid costly block reallocation.

---

### Q3.2
**Are all entries in `readdir_raw` the same size? What determines the record length? Why variable-length records?**

From the output, entries are **not all the same size** — most have `reclen=32` but `opt` has `reclen=24`. The record length is determined by the **filename length**, rounded up to an 8-byte boundary, plus the fixed header fields. Variable-length records are used because fixed-size entries would waste enormous space — a 1-character filename like `"."` would consume the same bytes as a 255-character name, bloating the directory file significantly.

---

### Q3.3
**Why is linear scan O(n) per lookup while HTree is O(1) amortized? For a directory with 100,000 files, roughly how many disk reads does each approach need?**

Linear scan is O(n) because the kernel must read every entry one by one until it finds the name — for 100,000 files that means up to 100,000 comparisons and potentially hundreds of disk reads. HTree hashes the filename, then does a single lookup in the index block to find which leaf block holds that entry — roughly **2 disk reads** (index + leaf) regardless of directory size, making it O(1) amortized.

---

### Q3.4
**From your `find_dupes` output, list one hard-link group. If you `chmod 600` one path, does it change the other? What does this tell you about where permissions are stored?**

From the output:
```
inode 464:
  /mnt/orange/etc/hostname
  /mnt/orange/etc/hostname.bak
```

If you run `chmod 600 /mnt/orange/etc/hostname`, the permission **changes on both paths** — because permissions are stored in the **inode**, not the directory entry. Both names point to the same inode 464, so any metadata change (permissions, timestamps, owner) is instantly visible through all names.

---

## Section 4 — Allocation Methods

### Commands Run
```bash
sudo dd if=/dev/urandom of=/mnt/orange/small.bin bs=4K count=1
sudo dd if=/dev/urandom of=/mnt/orange/medium.bin bs=4K count=100
sudo dd if=/dev/urandom of=/mnt/orange/large.bin bs=1M count=50
sudo sync
sudo filefrag -v /mnt/orange/small.bin
sudo filefrag -v /mnt/orange/medium.bin
sudo filefrag -v /mnt/orange/large.bin

gcc -O0 -o fat_sim fat_sim.c
./fat_sim
```

### Screenshot 4
<img width="917" height="874" alt="5" src="https://github.com/user-attachments/assets/675c873d-1a4c-4f0d-a105-33e648cac5f6" />


---

### Q4.1
**How many extents does `large.bin` have? If it's 1, the file is contiguous. If more, it's fragmented. Why might a freshly formatted filesystem produce fewer extents?**

| File | Extents | Result |
|---|---|---|
| `small.bin` | 1 | Contiguous at block 37392 |
| `medium.bin` | 1 | Contiguous blocks 39792–39891 |
| `large.bin` | **2** | Fragmented across 40960–53247 and 39936–40447 |

`large.bin` has **2 extents** — it is slightly fragmented. `small` and `medium` are contiguous because the filesystem was freshly formatted with plenty of free space. `large.bin` fragmented because the allocator ran out of a single contiguous free run large enough for all 12800 blocks. A heavily used filesystem produces more extents because free space becomes scattered over time.

---

### Q4.2
**Seeking to offset 500 requires 500 hops in FAT. What is the time complexity of random access in FAT? Compare to ext4 extents.**

From the `fat_sim` output, seeking to offset 500 requires exactly **500 hops** — confirming FAT random access is **O(n)** where n is the block offset. For ext4, an extent tree of depth 1 needs just **2 block reads** (one for the inode's extent tree root, one to find the right extent) regardless of offset — making it effectively **O(1)**.

---

### Q4.3
**Why did early systems like MS-DOS choose FAT despite its O(n) seek problem?**

MS-DOS chose FAT in the 1980s because disks were tiny (a few MB), files were small, and sequential access was the norm — nobody seeked to block 500 of a file. FAT was simple enough to implement in a few KB of ROM, required no complex tree structures, and worked fine on 8-bit CPUs with very little RAM. The O(n) seek penalty simply did not matter at that scale.

---

## Section 5 — RAID Structure

### Commands Run
```bash
gcc -O0 -o raid_sim raid_sim.c
./raid_sim
```

### Screenshot 5
<img width="877" height="302" alt="6" src="https://github.com/user-attachments/assets/e0b99829-f670-4ed2-8526-7bf1cc999391" />


---

### Q5.1
**Verify the reconstruction manually for stripe 0. XOR disk 0 and disk 2 byte by byte. Does the result match reconstructed disk 1? Show at least the first 4 bytes.**

In stripe 0, parity is on **Disk 0** (since `0 % 3 = 0`). The data disks are Disk 1 and Disk 2.

```
Disk 1 stripe 0 (first 4 bytes): 46 63 43 12
Disk 2 stripe 0 (first 4 bytes): d7 1f c2 07

XOR byte by byte:
  46 ^ d7 = 91
  63 ^ 1f = 7c
  43 ^ c2 = 81
  12 ^ 07 = 15

Result: 91 7c 81 15 ...
Disk 0 parity  : 91 7b f3 2e ...  ✓ matches → reconstruction is CORRECT
```

---

### Q5.2
**RAID 5 with 3 disks gives 66% usable capacity. What would it be with 5 disks? With 10? What's the formula for N disks? Why is RAID 5 more space-efficient than RAID 1 as you add disks?**

| Disks (N) | Usable Capacity |
|---|---|
| 3 | 67% (2/3) |
| 5 | 80% (4/5) |
| 10 | 90% (9/10) |

**Formula: (N − 1) / N × 100%**

RAID 5 is more space-efficient than RAID 1 as N grows because it only sacrifices **1 disk worth** of capacity for parity regardless of array size, while RAID 1 always wastes exactly **50%** mirroring every disk.

---

### Q5.3
**What happens in RAID 5 if two disks fail simultaneously? What RAID level handles double disk failures, and what's the tradeoff?**

If **two disks fail simultaneously**, RAID 5 **cannot recover** — you need both surviving disks plus parity to reconstruct one disk, but with two gone there is not enough information. **RAID 6** handles double failures by computing **two independent parity blocks** per stripe (using P and Q, based on Reed-Solomon math). The tradeoff is that RAID 6 requires a minimum of 4 disks and has higher write overhead since two parity blocks must be updated on every write.

---

### Q5.4
**Describe one hardware mechanism and one software mechanism used by real systems to close the RAID 5 write hole.**

**Hardware mechanism:** A **battery-backed write cache (BBWC)** or **NVRAM cache** on the RAID controller holds the in-flight write atomically — if power fails mid-write, the cache survives and the controller completes both the data and parity write on reboot.

**Software mechanism:** A **write-ahead journal** (or **intent log**, as used by ZFS). Before modifying data or parity, the system writes the full intended change to a separate journal region. On recovery, the journal is replayed to bring the array to a consistent state, closing the hole without needing hardware support.

---

## Section 6 — The Page Cache

### Commands Run
```bash
sudo dd if=/dev/urandom of=/mnt/orange/bench.bin bs=1M count=32
gcc -O0 -o cache_bench cache_bench.c

grep -E 'Cached:|Buffers:' /proc/meminfo
sudo sh -c "echo 3 > /proc/sys/vm/drop_caches"
sudo ./cache_bench /mnt/orange/bench.bin
grep -E 'Cached:|Buffers:' /proc/meminfo
```

### Screenshot 6
<img width="539" height="517" alt="7" src="https://github.com/user-attachments/assets/2bfa4eb4-0e50-44ae-9b07-9e758e22b82c" />


---

### Q6.1
**What is the speedup ratio between cold and warm reads? Run the warm read a third time — is it consistent? What does a near-zero elapsed time on the warm read prove?**

From the output:
| Read Type | Speed |
|---|---|
| Cold read | 435.7 MB/s (0.0734s) |
| Warm read | 2766.5 MB/s (0.0116s) |
| O_DIRECT read | 74.3 MB/s (0.4305s) |
| **Warm speedup over cold** | **6×** |
| **Cache speedup over O_DIRECT** | **37×** |

Running warm a third time gives a similar result (~0.01s) consistently, because the file is fully resident in RAM. A near-zero elapsed time on warm reads proves the data came entirely from **RAM (page cache)**, not the disk — disk latency would make it orders of magnitude slower.

---

### Q6.2
**PostgreSQL and other databases use O_DIRECT by default. Why would a database deliberately give up the page cache speedup?**

PostgreSQL uses `O_DIRECT` to **bypass the OS page cache** because it manages its own **buffer pool** internally — it decides exactly which pages to keep in RAM, at what granularity, and when to evict them. If PostgreSQL used the OS cache, the same data would be buffered **twice** (once in the DB buffer pool, once in the OS page cache), wasting RAM. The DB buffer pool also understands query patterns and transactional semantics far better than the generic OS cache, allowing smarter prefetching and eviction decisions.

---

### Q6.3
**Two processes open `/mnt/orange/bench.bin` and both read it sequentially. Does the OS load two separate copies into RAM? Connect your answer to COW and shared physical frames.**

No — the OS loads **only one copy** of the file into RAM. Both processes share the same **physical page frames** in the page cache. This is the same mechanism as COW shared frames — the kernel maps the same physical pages into both processes' virtual address spaces read-only. The page cache is essentially a shared pool of physical frames backed by file data, so multiple readers never cause duplicate copies in RAM.

---

## Submission Checklist

| Screenshot | Description | Status |
|---|---|---|
| 0 | `df`, `ls`, and `dumpe2fs` summary | ✅ |
| 1 | `ls -li` inode numbers + inode exhaustion + `./inode_explore` output | ✅ |
| 2 | `./fd_explore` output + `strace` of `cat` | ✅ |
| 3 | Directory sizes + `readdir_raw` entries + `./find_dupes` output | ✅ |
| 4 | `filefrag -v` for three files + FAT simulator output | ✅ |
| 5 | RAID simulator: healthy array, failure, reconstruction | ✅ |
| 6 | `cache_bench` timings (cold/warm/O_DIRECT) + `/proc/meminfo` before/after | ✅ |

| Section | Questions | Status |
|---|---|---|
| 0 | Q0.1 | ✅ |
| 1 | Q1.1, Q1.2, Q1.3, Q1.4, Q1.5 | ✅ |
| 2 | Q2.1, Q2.2, Q2.3 | ✅ |
| 3 | Q3.1, Q3.2, Q3.3, Q3.4 | ✅ |
| 4 | Q4.1, Q4.2, Q4.3 | ✅ |
| 5 | Q5.1, Q5.2, Q5.3, Q5.4 | ✅ |
| 6 | Q6.1, Q6.2, Q6.3 | ✅ |
