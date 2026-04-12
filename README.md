# Lab: File Systems & Storage

> **Platform:** Ubuntu 22.04
>
> **Install:** `sudo apt update && sudo apt install -y gcc build-essential e2fsprogs util-linux strace wget`

---

## Section 0 - Build Your Filesystem

You're going to create a real ext4 filesystem inside a regular file, mount it, and populate it with a real Linux root filesystem (Alpine). Sections after this operate on this image.

```bash
# Set your prompt to your SRN (last 3 digits only)
# Example: if your SRN is PES1UG24CS042, enter 042
export PS1="042\$ "
```

```bash
# Create a 256 MB file full of zeros
dd if=/dev/zero of=orange.img bs=1M count=256

# Format it as ext4
mkfs.ext4 -L ORANGE orange.img

# Mount it - the -o loop flag tells the kernel to treat the file as a block device,
# so the filesystem inside it behaves exactly like one on a real disk.
sudo mkdir -p /mnt/orange
sudo mount -o loop orange.img /mnt/orange

# Download Alpine Linux minirootfs and extract it
wget -q https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-minirootfs-3.21.3-x86_64.tar.gz
sudo tar xzf alpine-minirootfs-3.21.3-x86_64.tar.gz -C /mnt/orange

# See what we've got
df -h /mnt/orange
ls /mnt/orange
```

`dumpe2fs` reads the **superblock** - the master record ext4 writes at format time. It stores the filesystem's fixed parameters: total blocks, block size, and how many inodes were pre-allocated. These numbers never change after `mkfs`, which is why inode exhaustion is permanent.

```bash
# Record the filesystem metadata
sudo dumpe2fs -h orange.img 2>/dev/null | grep -E 'Block count|Block size|Inode count|Free blocks|Free inodes'
```

**Screenshot 0:** Output of `df -h /mnt/orange`, `ls /mnt/orange`, and `dumpe2fs` summary.

> **Q0.1:** From the `dumpe2fs` output: how many total inodes exist, how many are free, and how many are used? How many total blocks, and what's the block size? Show the arithmetic to verify: `used_inodes = total - free`.

---

## Section 1 - Inodes: The Real Identity of a File

Filenames are a convenience for humans. The filesystem doesn't care about names - it identifies everything by _inode number_. An inode holds the file's metadata (size, permissions, timestamps, block pointers) but _not_ the filename. The filename lives in the directory that contains the file.

This means you can have multiple names pointing to the same inode (hard links), or you can have a file with _no_ name at all (deleted but still open).

```
  Directory entry:            Inode:
  ┌──────────────────┐        ┌─────────────────────┐
  │ name: "hello.txt"│───────▶│ inode #41523        │
  │ inode: 41523     │        │ size: 1024          │
  └──────────────────┘        │ uid: 1000           │
  ┌──────────────────┐        │ blocks: [882, 883]  │
  │ name: "alias.txt"│───────▶│ link count: 2       │
  │ inode: 41523     │        └─────────────────────┘
  └──────────────────┘
  Two names, one file. link count = 2.
```

### Hard Links vs Symbolic Links

```bash
# Create a test file on our filesystem
echo "original content" | sudo tee /mnt/orange/test.txt

# Hard link: another directory entry pointing to the SAME inode
sudo ln /mnt/orange/test.txt /mnt/orange/hard.txt

# Symbolic link: a separate file whose content is a path string
sudo ln -s /mnt/orange/test.txt /mnt/orange/soft.txt

# Compare inode numbers
ls -li /mnt/orange/test.txt /mnt/orange/hard.txt /mnt/orange/soft.txt
stat /mnt/orange/test.txt
stat /mnt/orange/soft.txt
```

### Inode Exhaustion

A filesystem can run out of inodes before running out of space. The number of inodes is fixed at format time.

```bash
# Create a tiny filesystem with very few inodes
dd if=/dev/zero of=tiny.img bs=1M count=10
mkfs.ext4 -N 16 tiny.img   # only 16 inodes total!
sudo mkdir -p /mnt/tiny
sudo mount -o loop tiny.img /mnt/tiny

# Some inodes are already used by the FS itself (root dir, journal, etc.)
df -i /mnt/tiny

# Try to create files until we hit the inode limit
for i in $(seq 1 20); do
    sudo touch /mnt/tiny/file_$i 2>&1
done
echo "--- after inode exhaustion ---"
df -i /mnt/tiny
df -h /mnt/tiny   # space is still available!

sudo umount /mnt/tiny
rm -f tiny.img
```

### Code Task: Inode & Link Explorer

`stat()` follows symbolic links and reports the **target file's** inode. `lstat()` does not follow - it reports the **symlink's own** inode. This distinction is what lets the kernel tell you whether a path *is* a symlink or merely *points through* one. The `follow` parameter in `print_stat` below selects which call to use.

The program below creates a file and sets up three paths to it. Complete the four TODOs - each one targets a distinct concept:

```c
// inode_explore.c - complete the four TODOs
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>

static void print_stat(const char *label, const char *path, int follow) {
    struct stat sb;
    int ret = follow ? stat(path, &sb) : lstat(path, &sb);
    if (ret < 0) {
        printf("  %-20s %s: %s\n", label, path, strerror(errno));
        return;
    }
    printf("  %-20s inode=%-8lu  links=%-2lu  size=%-8ld  type=%s\n",
           label, (unsigned long)sb.st_ino, (unsigned long)sb.st_nlink,
           (long)sb.st_size,
           S_ISREG(sb.st_mode) ? "regular" :
           S_ISLNK(sb.st_mode) ? "symlink" :
           S_ISDIR(sb.st_mode) ? "directory" : "other");
}

int main(void) {
    const char *orig = "/mnt/orange/explore_test.txt";
    const char *hard = "/mnt/orange/explore_hard.txt";
    const char *soft = "/mnt/orange/explore_soft.txt";

    // Create original file
    int fd = open(orig, O_CREAT | O_WRONLY | O_TRUNC, 0644);
    write(fd, "inode exploration\n", 18);
    close(fd);

    printf("=== After creating original ===\n");
    print_stat("original (stat):", orig, 1);

    // TODO 1: Create a hard link (link()) from orig → hard.
    //         Create a symbolic link (symlink()) from orig → soft.

    printf("\n=== After creating links ===\n");
    print_stat("original (stat):", orig, 1);
    print_stat("hard link (stat):", hard, 1);
    print_stat("soft via stat:", soft, 1);   // follows symlink
    // TODO 2: Call print_stat on 'soft' with follow=0 (uses lstat internally).
    //         The inode number will differ from the stat() call above. Record why.

    // TODO 3: Delete the original file using unlink().
    printf("\n=== After unlinking original ===\n");
    print_stat("hard link (stat):", hard, 1);   // still works? link count?
    print_stat("soft via stat:", soft, 1);       // broken?

    // TODO 4: Open and read 64 bytes via the hard link path using open()/read().
    //         Then attempt the same via the soft link path.
    //         Print the content on success, or strerror(errno) on failure.
    //         Which one works? Print a one-line explanation as a printf.

    // Clean up
    unlink(hard);
    unlink(soft);
    return 0;
}
```

```bash
gcc -O0 -o inode_explore inode_explore.c
sudo ./inode_explore
```

> **Q1.1:** `test.txt` and `hard.txt` share the same inode number. What is the link count shown by `stat`? Delete `test.txt` with `sudo rm /mnt/orange/test.txt` - does `hard.txt` still work? What happened to the link count? Why doesn't the data get freed?
>
> **Q1.2:** `soft.txt` has a _different_ inode number from `test.txt`. After deleting `test.txt`, does `cat /mnt/orange/soft.txt` still work? Why not? What's the fundamental difference between how the kernel resolves a hard link vs a symbolic link?
>
> **Q1.3:** In the inode exhaustion experiment, at what point did `touch` start failing? How much disk space was still free when that happened? Why can't ext4 just create more inodes on the fly?
>
> **Q1.4:** In your completed `inode_explore.c`, what inode number does `stat()` return for the symlink vs what `lstat()` returns? Why are they different? Which one gives you the symlink's _own_ inode?
>
> **Q1.5:** A process opens a file, then a second process deletes it (the directory entry disappears). The file's data still exists on disk. Which field inside the in-memory inode tracks that the file is still open (the open-file-table reference count)? What syscall by the first process finally drops that count to zero and triggers the free? Now consider: if the first process keeps the file open indefinitely, what happens to disk space?

**Screenshot 1:** `ls -li` showing inode numbers + link counts, `df -i /mnt/tiny` before and after exhaustion, and your completed `./inode_explore` output.

---

## Section 2 - What Happens When You open() a File

When you call `open()`, the kernel doesn't just hand you the file. It walks the directory tree component by component, looks up each name in the parent directory's entries, follows the chain of inodes, and finally creates a _file descriptor_ - an integer that's your process's handle to an open file. The kernel maintains three layers of bookkeeping:

```
  Example matching the code below:

  fd1 ──┐
        ├──▶ open file entry A: flags = O_RDWR,   current offset = 0, refcount = 2 ──▶ inode #41523
  fd3 ──┘

  fd2 ─────▶ open file entry B: flags = O_RDONLY, current offset = 0, refcount = 1 ──▶ inode #41523

  Per-process FD table:
  - `fd1`, `fd2`, and `fd3` are just integer entries in the process's FD table

  System-wide open file table:
  - open file entry A is shared by `fd1` and `fd3` because `fd3 = dup(fd1)`
  - open file entry B is separate because `fd2` came from a different `open()`

  In-memory inode table:
  - both open file entries still point to the same inode, because both refer to the same file
```

```c
// fd_explore.c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main(void) {
    // Create a test file
    int fd1 = open("/mnt/orange/fd_test.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    write(fd1, "ABCDEFGHIJ", 10);

    // Two independent opens of the same file → two separate offsets
    int fd2 = open("/mnt/orange/fd_test.txt", O_RDONLY);

    // dup() shares the SAME open file entry → same offset
    int fd3 = dup(fd1);

    printf("fd1=%d  fd2=%d  fd3=%d\n\n", fd1, fd2, fd3);

    char buf[16] = {0};

    // TODO 1: Seek fd1 to position 0 with lseek(), then read 3 bytes into buf.
    //         Zero buf before each read (memset). Print the result, e.g.:
    //           printf("read(fd1, 3): \"%s\"  → fd1 offset now at 3\n", buf);
    //         Predict first: what 3 characters will you get?

    // TODO 2: Without seeking, read 3 bytes from fd3 into buf (memset first).
    //         Print:  printf("read(fd3, 3): \"%s\"  → shared offset, continues from 3\n", buf);
    //         Predict first: does fd3 = dup(fd1) share the open file entry (and offset)?

    // TODO 3: Without seeking, read 3 bytes from fd2 into buf (memset first).
    //         Print:  printf("read(fd2, 3): \"%s\"  → independent open, starts at 0\n", buf);
    //         Predict first: fd2 came from a separate open() - what is its offset?

    // The kernel exposes each process's open file descriptors as symlinks under
    // /proc/<pid>/fd - each symlink points to the file that fd refers to.
    // This lets you inspect the live FD table of a running process.
    printf("\n--- /proc/self/fd ---\n");
    char cmd[128];
    snprintf(cmd, sizeof(cmd), "ls -l /proc/%d/fd", getpid());
    system(cmd);

    close(fd1);
    close(fd2);
    close(fd3);
    return 0;
}
```

```bash
gcc -O0 -o fd_explore fd_explore.c
sudo ./fd_explore

# strace intercepts every system call a process makes and prints them to stderr.
# It lets you see exactly how a program talks to the kernel - open, read, write, close.
# Here we filter to just the file-related calls to watch what 'cat' actually does.
strace -e openat,read,write,close cat /mnt/orange/etc/hostname 2>&1
```

> **Q2.1:** `fd1` and `fd3` share a file offset (because `fd3 = dup(fd1)`), but `fd2` has its own. After `read(fd1, 3)`, reading from `fd3` returns "DEF" not "ABC". Why? Draw the three-layer diagram for this specific scenario.
>
> **Q2.2:** In the `strace` output for `cat /mnt/orange/etc/hostname`, how many `read()` calls does `cat` make? What size buffer does it use? At what point does it see EOF (return value 0)?
>
> **Q2.3:** A file descriptor is just an integer index. If a process opens 1000 files, what limits prevent it from opening more? Run `ulimit -n` to check. What happens if a server process hits this limit?

**Screenshot 2:** Output of `./fd_explore` showing the three reads, and the `strace` output for `cat`.

---

## Section 3 - Directory Implementation

A directory is just a file whose data contains a list of `(name, inode)` pairs. When you do `ls`, the kernel reads the directory's data blocks and parses these entries. ext4 uses a hash-tree (HTree) structure for large directories - essentially a B-tree keyed on filename hash - so that lookup doesn't degrade to a linear scan.

```
  Simple directory (small, linear):
  ┌────────┬───────┬────────┬───────┐
  │ "."    │ "..  "│ "foo"  │ "bar" │
  │ ino=2  │ ino=2 │ ino=15 │ ino=23│
  └────────┴───────┴────────┴───────┘

  HTree directory (large, hashed):
     ┌──────────────────────┐
     │ Root index block     │
     │ hash < 0x40 → blk 5  │
     │ hash < 0x80 → blk 9  │
     │ hash ≥ 0x80 → blk 12 │
     └──────────────────────┘
       │         │         │
       ▼         ▼         ▼
   ┌───────┐ ┌───────┐ ┌───────┐
   │entries│ │entries│ │entries│   ← leaf blocks with actual (name, inode) pairs
   └───────┘ └───────┘ └───────┘
```

```bash
# On our Alpine filesystem, let's look at directory sizes
sudo ls -lad /mnt/orange/etc /mnt/orange/usr

# Create a large directory and watch it grow
sudo mkdir -p /mnt/orange/bigdir
for i in $(seq 1 500); do
    sudo touch /mnt/orange/bigdir/file_$i
done
sudo ls -lad /mnt/orange/bigdir   # directory size grew!

# Now time a lookup in a small vs large directory
echo "--- small directory (etc, ~40 entries) ---"
time sudo ls /mnt/orange/etc > /dev/null

echo "--- large directory (bigdir, 500 entries) ---"
time sudo ls /mnt/orange/bigdir > /dev/null
```

The standard `readdir()` library function hides the on-disk structure behind a clean API. To see the raw `(name, inode, record-length)` tuples the kernel actually returns, we call the underlying `getdents64` syscall directly. This exposes the `d_reclen` field - the variable-length record size that `readdir()` abstracts away.

```c
// readdir_raw.c - read directory entries using getdents64 syscall
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <string.h>

struct linux_dirent64 {
    unsigned long long d_ino;
    long long          d_off;
    unsigned short     d_reclen;
    unsigned char      d_type;
    char               d_name[];
};

int main(int argc, char *argv[]) {
    const char *dir = argc > 1 ? argv[1] : "/mnt/orange/etc";
    int fd = open(dir, O_RDONLY | O_DIRECTORY);
    if (fd < 0) { perror("open"); return 1; }

    char buf[4096];
    int count = 0;
    int nread;

    while ((nread = syscall(SYS_getdents64, fd, buf, sizeof(buf))) > 0) {
        int pos = 0;
        while (pos < nread) {
            struct linux_dirent64 *d = (struct linux_dirent64 *)(buf + pos);
            if (count < 10)  // print first 10 entries
                printf("  inode=%-8llu type=%d reclen=%-4d name=\"%s\"\n",
                       d->d_ino, d->d_type, d->d_reclen, d->d_name);
            pos += d->d_reclen;
            count++;
        }
    }
    printf("  ... total entries: %d\n", count);
    close(fd);
    return 0;
}
```

```bash
gcc -O0 -o readdir_raw readdir_raw.c
sudo ./readdir_raw /mnt/orange/etc
sudo ./readdir_raw /mnt/orange/bigdir
```

### Code Task: Hard-Link Finder

A directory tree can contain hard links - multiple filenames pointing to the same inode. `find_dupes` walks a directory tree and prints every group of paths that share an inode number. Implement the two TODOs:

```c
// find_dupes.c - find all hard-linked files under a directory tree
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <sys/stat.h>

#define MAX_FILES 4096

typedef struct { unsigned long inode; char path[512]; } FileEntry;

static FileEntry files[MAX_FILES];
static int file_count = 0;

// TODO 1: Implement walk(const char *dir).
//         Use opendir()/readdir() + lstat() on each entry.
//         For regular files: store inode + full path in files[], increment file_count.
//         For directories (skip "." and ".."): recurse.
//         Stop silently if file_count reaches MAX_FILES.
static void walk(const char *dir) { /* your code */ }

// TODO 2: Implement find_dupes().
//         Sort files[] by inode using qsort (write a comparator).
//         Walk the sorted array; when consecutive entries share an inode,
//         print them as a group. Only print groups of size >= 2.
//         Format each group as:
//           inode 12345:
//             /mnt/orange/etc/hostname
//             /mnt/orange/etc/hostname.bak
static void find_dupes(void) { /* your code */ }

int main(int argc, char *argv[]) {
    const char *root = argc > 1 ? argv[1] : "/mnt/orange";
    walk(root);
    printf("Scanned %d files under %s\n\n", file_count, root);
    find_dupes();
    return 0;
}
```

```bash
# Plant a few hard links for find_dupes to discover
sudo ln /mnt/orange/etc/hostname /mnt/orange/etc/hostname.bak
sudo ln /mnt/orange/etc/fstab /mnt/orange/home/fstab_copy 2>/dev/null || true

gcc -O0 -o find_dupes find_dupes.c
sudo ./find_dupes /mnt/orange
```

> **Q3.1:** What is the directory file's size (from `ls -lad`) for `bigdir` vs `etc`? A directory's size on disk grows as you add entries - but does it shrink when you delete them? Try `sudo rm /mnt/orange/bigdir/file_{1..400}` and check again. Why or why not?
>
> **Q3.2:** In the `readdir_raw` output, each entry has a `d_reclen` (record length). Are all entries the same size? What determines the record length? Why would the kernel want variable-length records here instead of fixed-size ones?
>
> **Q3.3:** ext4 switches from linear directory entries to an HTree (hash-based) index when a directory grows beyond one block. Why is linear scan O(n) per lookup while HTree is O(1) amortized? For a directory with 100,000 files, roughly how many disk reads does each approach need to find one file?
>
> **Q3.4:** From your `find_dupes` output, list one hard-link group: the shared inode number and both paths. Now: if you run `chmod 600` on one of those paths, does the permission change on the other path too? Why? What does this tell you about where permissions are stored - in the directory entry or the inode?

**Screenshot 3:** Directory sizes for `etc` vs `bigdir`, first 10 entries from `readdir_raw`, and your completed `./find_dupes /mnt/orange` output.

---

## Section 4 - Allocation Methods

How does a filesystem decide _which blocks on disk_ hold a given file's data? Three classical strategies exist:

```
  Contiguous:  File occupies consecutive blocks.
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ A │ A │ A │   │ B │ B │   │   │
  └───┴───┴───┴───┴───┴───┴───┴───┘
  + Fast sequential read (one seek)
  - External fragmentation, can't grow easily

  Linked:  Each block has a pointer to the next.
  ┌───┐   ┌───┐   ┌───┐
  │ A │──▶│ A │──▶│ A │──▶ NULL
  └───┘   └───┘   └───┘
  + No external fragmentation, easy to grow
  - Random access is O(n): to reach block k, follow k pointers

  Indexed (ext4 extents):  Inode holds an extent tree.
  ┌──────────────────────────────┐
  │ Inode extent tree:           │
  │  [0-99]   → disk blocks 500  │
  │  [100-149]→ disk blocks 800  │
  │  [150-151]→ disk block 1200  │
  └──────────────────────────────┘
  + Fast random access, no external frag
  - More metadata overhead
```

ext4 uses extents (a form of indexed allocation). Let's see it in action, then build a FAT (linked allocation) simulator:

`filefrag` asks the kernel for a file's extent map - the list of contiguous block ranges that make up the file on disk. One extent means the file is stored as a single contiguous run. Multiple extents mean it is fragmented across different locations, requiring separate seeks to read sequentially.

```bash
# Create files of different sizes and check their extent layout
sudo dd if=/dev/urandom of=/mnt/orange/small.bin bs=4K count=1
sudo dd if=/dev/urandom of=/mnt/orange/medium.bin bs=4K count=100
sudo dd if=/dev/urandom of=/mnt/orange/large.bin bs=1M count=50

# filefrag shows the extent map - how many extents (fragments) each file has
sudo filefrag -v /mnt/orange/small.bin
sudo filefrag -v /mnt/orange/medium.bin
sudo filefrag -v /mnt/orange/large.bin
```

In a FAT filesystem, the File Allocation Table is an array where `fat[i]` stores the index of the **next** block belonging to the same file, or a sentinel (`FAT_EOF`) if block `i` is the last one. To find block `k` of a file, you must follow `k` pointers in sequence - there is no shortcut. `fat_alloc` uses **first-fit**: it scans from block 0 and takes the first free run it finds. This is simple but can produce fragmented chains over time.

```c
// fat_sim.c - simulate FAT-style linked allocation and measure random access cost
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define DISK_BLOCKS 4096
#define FAT_FREE   -1
#define FAT_EOF    -2

int fat[DISK_BLOCKS];   // FAT table: fat[i] = next block in chain, or FAT_EOF

// Allocate a file of n blocks using first-fit from FAT
int fat_alloc(int n) {
    int first = -1, prev = -1;
    int allocated = 0;
    for (int i = 0; i < DISK_BLOCKS && allocated < n; i++) {
        if (fat[i] == FAT_FREE) {
            fat[i] = FAT_EOF;  // tentatively mark as end
            if (prev >= 0) fat[prev] = i;
            if (first < 0) first = i;
            prev = i;
            allocated++;
        }
    }
    if (allocated < n) { return -1; }  // not enough space
    return first;
}

// Random access: reach block at logical offset k in the chain.
// TODO: Walk the FAT chain starting at 'start'.
//       Each step follows one fat[] pointer - that is one "disk read" in a real FAT.
//       Stop after k hops or when you hit FAT_EOF, whichever comes first.
//       Return the total number of hops taken.
int fat_seek(int start, int k) {
    /* your code here */
    return 0;
}

int main(void) {
    // Initialize FAT
    for (int i = 0; i < DISK_BLOCKS; i++) fat[i] = FAT_FREE;

    // Fragment the disk: allocate 64 small files, free every other one
    int files[64];
    for (int i = 0; i < 64; i++)
        files[i] = fat_alloc(32);  // 64 files × 32 blocks = 2048 blocks used
    for (int i = 0; i < 64; i += 2)  // free even-numbered files
        for (int b = files[i]; b != FAT_EOF; ) {
            int next = fat[b];
            fat[b] = FAT_FREE;
            b = next;
        }

    // Allocate a large file on the now-fragmented disk
    int big_file = fat_alloc(512);
    if (big_file < 0) { printf("Allocation failed!\n"); return 1; }

    // Measure random access: seek to positions 0, 100, 200, 300, 400, 500
    printf("%-12s  %s\n", "Offset", "Hops (pointer follows)");
    printf("----------------------------\n");
    for (int off = 0; off <= 500; off += 100) {
        int hops = fat_seek(big_file, off);
        printf("%-12d  %d\n", off, hops);
    }

    printf("\nIn a FAT, seeking to block N requires following N pointers.\n");
    printf("ext4 extents: one tree lookup regardless of offset.\n");

    return 0;
}
```

```bash
gcc -O0 -o fat_sim fat_sim.c
./fat_sim
```

> **Q4.1:** Look at the `filefrag` output for `large.bin`. How many extents does it have? If it's just 1 extent, the file is stored contiguously. If it's more than 1, the file is fragmented. Why might a freshly formatted filesystem produce fewer extents than a heavily used one?
>
> **Q4.2:** In the FAT simulator, seeking to offset 500 requires 500 hops. What is the time complexity of random access in FAT? Compare this to ext4 extents where an extent tree of depth 1 can address `4 × 340^1 = 1360` extents - how many block reads does ext4 need for the same random seek?
>
> **Q4.3:** Contiguous allocation has the best read performance but worst flexibility. Linked (FAT) has no external fragmentation but O(n) seeks. Indexed (extents) balances both. Why did early systems like MS-DOS choose FAT despite its O(n) seek problem? (Think about the hardware constraints of the 1980s - small disks, small files, simple CPUs.)

**Screenshot 4:** `filefrag -v` output for the three files, and the FAT simulator output.

---

## Section 5 - RAID Structure

RAID (Redundant Array of Independent Disks) combines multiple disks for performance, redundancy, or both. Three levels matter:

```
  RAID 0 (Striping):          RAID 1 (Mirroring):        RAID 5 (Striping + Distributed Parity):
  ┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐          ┌──────┐ ┌──────┐ ┌──────┐
  │ D0   │ │ D1   │          │ D0   │ │ D0   │          │ D0   │ │ D1   │ │ P0   │
  │ D2   │ │ D3   │          │ D1   │ │ D1   │          │ D3   │ │ P1   │ │ D2   │
  │ D4   │ │ D5   │          │ D2   │ │ D2   │          │ P2   │ │ D4   │ │ D5   │
  └──────┘ └──────┘          └──────┘ └──────┘          └──────┘ └──────┘ └──────┘
  2× throughput               1 disk can fail             1 disk can fail
  0 redundancy                50% capacity wasted         only 1/N capacity wasted
```

RAID 5 parity works by XOR. For each stripe, the parity block `P = D0 ⊕ D1 ⊕ ... ⊕ Dn`. XOR has a key property: it is **self-inverse** - XORing any value with itself cancels it out (`A ⊕ A = 0`). This means if you know `P` and all data blocks except one, you can recover the missing block by XORing everything else: `D1 = P ⊕ D0` (since `P ⊕ D0 = D0 ⊕ D1 ⊕ D0 = D1`). This is why any single disk failure is recoverable.

```c
// raid_sim.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BLOCK_SIZE 8       // bytes per block (small for visibility)
#define NUM_STRIPES 4
#define NUM_DISKS 3        // RAID 5 with 3 disks

unsigned char disks[NUM_DISKS][NUM_STRIPES][BLOCK_SIZE];

static void print_disks(const char *label) {
    printf("\n%s\n", label);
    for (int d = 0; d < NUM_DISKS; d++) {
        printf("  Disk %d: ", d);
        for (int s = 0; s < NUM_STRIPES; s++) {
            char tag;
            // In RAID 5, parity rotates: stripe s has parity on disk (s % NUM_DISKS)
            if (d == (s % NUM_DISKS))
                tag = 'P';
            else
                tag = 'D';
            printf("[%c:", tag);
            for (int b = 0; b < BLOCK_SIZE; b++)
                printf("%02x", disks[d][s][b]);
            printf("] ");
        }
        printf("\n");
    }
}

// TODO: Implement reconstruct(failed_disk).
//       For each stripe s and each byte b, XOR all surviving disks together
//       and write the result into disks[failed_disk][s][b].
//       The failed disk's blocks are already zeroed before this is called.
//       Recall: XOR is self-inverse. If P = A ^ B, then A = P ^ B.
//       The loop structure mirrors the parity computation in main() below.
static void reconstruct(int failed_disk) {
    (void)failed_disk;
    /* your code here */
}

int main(void) {
    srand(42);  // Fixed seed so every run produces the same hex values -
                // this makes the manual byte-by-byte verification in Q5.1 reproducible.

    // Write data blocks (skip parity disk for each stripe)
    for (int s = 0; s < NUM_STRIPES; s++) {
        int parity_disk = s % NUM_DISKS;
        for (int d = 0; d < NUM_DISKS; d++) {
            if (d != parity_disk) {
                for (int b = 0; b < BLOCK_SIZE; b++)
                    disks[d][s][b] = rand() & 0xFF;
            }
        }
        // Compute parity = XOR of all data blocks in this stripe
        memset(disks[parity_disk][s], 0, BLOCK_SIZE);
        for (int d = 0; d < NUM_DISKS; d++) {
            if (d != parity_disk)
                for (int b = 0; b < BLOCK_SIZE; b++)
                    disks[parity_disk][s][b] ^= disks[d][s][b];
        }
    }

    print_disks("=== RAID 5 Array (healthy) ===");

    // Simulate disk 1 failure
    int failed_disk = 1;
    unsigned char backup[NUM_STRIPES][BLOCK_SIZE];
    memcpy(backup, disks[failed_disk], sizeof(backup));  // save for verification
    memset(disks[failed_disk], 0, sizeof(disks[failed_disk]));

    printf("\n*** DISK %d FAILED ***\n", failed_disk);
    reconstruct(failed_disk);

    print_disks("=== After reconstruction ===");

    // Verify
    int correct = (memcmp(backup, disks[failed_disk], sizeof(backup)) == 0);
    printf("\nReconstruction %s\n", correct ? "CORRECT ✓" : "FAILED ✗");

    // Capacity analysis
    int total_blocks = NUM_DISKS * NUM_STRIPES;
    int parity_blocks = NUM_STRIPES;  // one parity per stripe
    printf("\nCapacity: %d data + %d parity = %d total blocks\n",
           total_blocks - parity_blocks, parity_blocks, total_blocks);
    printf("Usable: %.0f%%\n", 100.0 * (total_blocks - parity_blocks) / total_blocks);

    return 0;
}
```

```bash
gcc -O0 -o raid_sim raid_sim.c
./raid_sim
```

> **Q5.1:** Verify the reconstruction manually for stripe 0. Take the hex values from disk 0 and disk 2, XOR them byte by byte. Does the result match the reconstructed disk 1? Show at least the first 4 bytes of your work.
>
> **Q5.2:** RAID 5 with 3 disks gives 66% usable capacity. What would it be with 5 disks? With 10? What's the formula for N disks? Why is RAID 5 more space-efficient than RAID 1 as you add disks?
>
> **Q5.3:** What happens in RAID 5 if _two_ disks fail simultaneously? Can the data be reconstructed? What RAID level handles double disk failures, and what's the tradeoff? (Look up RAID 6.)
>
> **Q5.4:** RAID 5 has a "write hole": updating one data block requires a read-modify-write of the parity block - two separate disk operations. If power fails between them, parity is inconsistent with data. Neither `fsck` nor the RAID controller can detect which version is correct. Describe one hardware mechanism and one software mechanism used by real systems to close this hole.

**Screenshot 5:** Full RAID simulator output showing healthy array, failure, and successful reconstruction.

---

## Section 6 - The Page Cache

Every file read or write in Linux goes through the page cache - a region of physical RAM where the kernel keeps recently-used filesystem data. On the first read, the kernel faults in the data from disk and keeps it in the cache. Subsequent reads of the same file data are served from RAM without any disk I/O. Writes are also buffered: `write()` returns as soon as the data lands in cache (the kernel flushes to disk later), which is why it seems fast even on slow disks.

```
  File read path:
  process → read() → page cache hit? → YES → copy to user buffer (fast)
                                     → NO  → read from disk → fill cache → copy to user buffer (slow)

  File write path:
  process → write() → copy into page cache (dirty page) → return immediately
                                  ↓ (background)
                             pdflush writes dirty pages to disk
```

```c
// cache_bench.c - measure cold vs warm read latency, and O_DIRECT bypass
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <time.h>
#include <string.h>
#include <errno.h>

#define BUF_SIZE (4 * 1024)   // 4 KB reads

static double read_file(const char *path, int use_direct) {
    // TODO: Set the correct open() flags.
    //       When use_direct != 0, add O_DIRECT to bypass the page cache entirely.
    //       O_DIRECT transfers data directly between the disk and your buffer,
    //       skipping the kernel's page cache on both reads and writes.
    int flags = O_RDONLY; /* add O_DIRECT when use_direct != 0 */
    int fd = open(path, flags);
    if (fd < 0) { perror("open"); return -1; }

    // Buffer must be 4096-byte aligned for O_DIRECT (DMA hardware requirement).
    void *buf;
    posix_memalign(&buf, 4096, BUF_SIZE);

    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);

    ssize_t n;
    long total = 0;
    while ((n = read(fd, buf, BUF_SIZE)) > 0)
        total += n;

    clock_gettime(CLOCK_MONOTONIC, &t1);
    close(fd);
    free(buf);

    double elapsed = (t1.tv_sec - t0.tv_sec) + (t1.tv_nsec - t0.tv_nsec) / 1e9;
    printf("  %ld bytes in %.4f s  →  %.1f MB/s\n",
           total, elapsed, (total / (1024.0 * 1024.0)) / elapsed);
    return elapsed;
}

int main(int argc, char *argv[]) {
    if (argc < 2) { fprintf(stderr, "usage: %s <file>\n", argv[0]); return 1; }

    printf("=== Cold read (page cache just dropped) ===\n");
    double cold = read_file(argv[1], 0);

    printf("\n=== Warm read (file still in page cache) ===\n");
    double warm = read_file(argv[1], 0);

    printf("\n=== O_DIRECT read (bypasses page cache entirely) ===\n");
    double direct = read_file(argv[1], 1);

    if (cold > 0 && warm > 0)
        printf("\nWarm speedup over cold:     %.0f×\n", cold / warm);
    if (direct > 0 && warm > 0)
        printf("Cache speedup over O_DIRECT: %.0f×\n", direct / warm);

    return 0;
}
```

```bash
# Create a 32 MB test file
sudo dd if=/dev/urandom of=/mnt/orange/bench.bin bs=1M count=32

gcc -O0 -o cache_bench cache_bench.c

# Check page cache size BEFORE the run
grep -E 'Cached:|Buffers:' /proc/meminfo

# Writing to /proc/sys/vm/drop_caches tells the kernel to evict cached pages from RAM.
# 1 = evict page cache (file data), 2 = evict dentries/inodes, 3 = evict both.
# We use 3 to guarantee the next read hits the disk, not cache - producing a true cold read.
sudo sh -c "echo 3 > /proc/sys/vm/drop_caches"
sudo ./cache_bench /mnt/orange/bench.bin

# Check page cache size AFTER the run - did it grow?
grep -E 'Cached:|Buffers:' /proc/meminfo
```

> **Q6.1:** What is the speedup ratio between cold and warm reads? Run the warm read a third time (without dropping the cache) - is it consistent? What does a near-zero elapsed time on the warm read prove about where the data came from?
>
> **Q6.2:** The O_DIRECT read bypasses the page cache and goes straight to disk every time. PostgreSQL and other databases use O_DIRECT by default. Why would a database deliberately give up the page cache speedup? (Think about who manages buffering and at what granularity - the OS cache vs the database buffer pool.)
>
> **Q6.3:** Two processes open `/mnt/orange/bench.bin` and both read it sequentially. Does the OS load two separate copies of the file into RAM? Connect your answer to what you observed in U3 about COW and shared physical frames - is the page cache using the same mechanism?

**Screenshot 6:** Full `cache_bench` output showing cold, warm, and O_DIRECT timings with speedup ratios, and `grep Cached /proc/meminfo` output from both before and after the run.

---

## Cleanup

```bash
# Unmount the filesystem
sudo umount /mnt/orange 2>/dev/null
sudo rmdir /mnt/orange 2>/dev/null
# Remove tiny.img if still around
sudo umount /mnt/tiny 2>/dev/null
sudo rmdir /mnt/tiny 2>/dev/null

# Clean up all binaries, source files, and images
rm -f orange.img tiny.img alpine-minirootfs-*.tar.gz
rm -f inode_explore fd_explore readdir_raw find_dupes fat_sim raid_sim cache_bench
rm -f inode_explore.c fd_explore.c readdir_raw.c find_dupes.c fat_sim.c raid_sim.c cache_bench.c
rm -f small.bin medium.bin large.bin bench.bin
```

---

## Submission Checklist

### Screenshots (7 total)

| #   | What to capture                                                               |
| --- | ----------------------------------------------------------------------------- |
| 0   | `df`, `ls`, and `dumpe2fs` summary of the Alpine filesystem                   |
| 1   | `ls -li` inode numbers + `df -i` inode exhaustion + `./inode_explore` output  |
| 2   | `./fd_explore` output + `strace` of `cat`                                     |
| 3   | Directory sizes + `readdir_raw` entries + `./find_dupes` output               |
| 4   | `filefrag -v` for three files + FAT simulator output                          |
| 5   | RAID simulator: healthy array, failure, reconstruction                        |
| 6   | `cache_bench` timings (cold / warm / O_DIRECT) + `/proc/meminfo` cache counts |

### Written Answers (24 total)

| Section | Questions                        |
| ------- | -------------------------------- |
| 0       | Q0.1                             |
| 1       | Q1.1, Q1.2, Q1.3, Q1.4, Q1.5    |
| 2       | Q2.1, Q2.2, Q2.3                 |
| 3       | Q3.1, Q3.2, Q3.3, Q3.4           |
| 4       | Q4.1, Q4.2, Q4.3                 |
| 5       | Q5.1, Q5.2, Q5.3, Q5.4           |
| 6       | Q6.1, Q6.2, Q6.3                 |
