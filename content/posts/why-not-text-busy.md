---
title: "why not text busy"
date: 2024-07-27T19:14:00-07:00
draft: false
aliases:
- why-not-text-busy
---

There are two types of troubleshootings:

1. Why something doesn't work?
1. Why something worked?

This post is about the second category.

![text-busy](/images/ubuntu-2004-vs-ubuntu-2204.png)

## The Problem

We have been using [kaniko][1] as image builder since a while back, including
building a customzied kaniko image with itself like below:

```Dockerfile
FROM alpine:latest
COPY --from=gcr.io/kaniko-project/executor:v1.23.2 /kaniko /kaniko
```

Despite of its [official documentation][2] clearly suggesting that this is not
supported:

> Running kaniko in any Docker image other than the official kaniko image is not
> supported due to implementation details.
>
>  * This includes copying the kaniko executables from the official image into
>    another image (e.g. a Jenkins CI agent).
>  * In particular, it cannot use chroot or bind-mount because its container
>    must not require privilege, so it unpacks directly into its own container
>    root and may overwrite anything already there.

The team did it anyway, and it worked pretty fine... until it doesn't!

```console
$ docker run -it --entrypoint /bin/sh -v $(pwd):/workspace gcr.io/kaniko-project/executor:v1.23.2-debug
/workspace # executor --no-push
INFO[0000] Retrieving image manifest gcr.io/kaniko-project/executor:v1.23.2
INFO[0000] Retrieving image gcr.io/kaniko-project/executor:v1.23.2 from registry gcr.io
INFO[0000] Storing source image from stage gcr.io/kaniko-project/executor:v1.23.2 at path /kaniko/stages/gcr.io/kaniko-project/executor:v1.23.2
INFO[0010] Retrieving image manifest alpine:latest
INFO[0010] Retrieving image alpine:latest from registry index.docker.io
INFO[0022] Built cross stage deps: map[]
INFO[0022] Retrieving image manifest alpine:latest
INFO[0022] Returning cached image manifest
INFO[0022] Executing 0 build triggers
INFO[0022] Building stage 'alpine:latest' [idx: '0', base-idx: '-1']
INFO[0022] Unpacking rootfs as cmd COPY --from=gcr.io/kaniko-project/executor:v1.23.2 /kaniko /kaniko requires it.
INFO[0022] COPY --from=gcr.io/kaniko-project/executor:v1.23.2 /kaniko /kaniko
error building image: error building stage: failed to execute command: copying dir: creating file: open /kaniko/executor: text file busy
```

It didn't take too long before we figured it out that:

* It only happens on ubuntu 20.04. (Not happenning on 22.04)
* The binary is on overlayfs (containers). (Not happenning on xfs)

## Simplification

As [kaniko][1] has its own complications, we later figured out [an easy way][3]
to reproduce the same behavior. All we need is to create an application which
tries to open its own executable (aka, `/proc/self/exe`) with `O_RDWR`. You may
build this program as a docker image with the following steps:

```console
$ git clone https://github.com/zhouhaibing089/txtbsy.git
$ cd txtbsy
$ docker build -t txtbsy:v0 -f Dockerfile .
```

> kaniko has its binary at path `/kaniko/executor`. When it tries to copy
> `/kaniko` from another image to its own `/kaniko`, it essentially does the
> same thing to override its own executable to be something else.

And then you can run this program on a freshly installed [ubuntu 20.04][4].

```console
$ docker run -it --entrypoint /bin/sh --rm txtbsy:v0
/ # /app
2024/07/29 01:28:45 pid: 7
2024/07/29 01:28:45 path: /app, dev: 53, inode 675400
2024/07/29 01:28:45 path: /app, dev: 53, inode 675400
/ # /app
2024/07/29 01:28:47 pid: 11
2024/07/29 01:28:47 path: /app, dev: 53, inode 675400
2024/07/29 01:28:47 failed to create /app: open /app: text file busy
```

The first run completed successfully (but unexpectedly) while the second run
failed with `text file busy` error which it should.

## Starting with strace

[strace][5] is a trace tool which shows all the system calls a process makes
during its lifetime. As we see `text file busy` error, it makes sense for us to
see which system call (if at all) gives such error.

```console
$ strace -p <pid> -f -e trace="/(open|exec)" -o syscall.txt
```

> pid is the pid of `/bin/sh` from the container. `-f` allows the program to
> further follow forks (child processes like `/app`). I also applied a filter
> to only care about `open` and `exec`-like system calls, but you may skip it
> to see all of them (which could be a bit noisy).

The [output][6] is something like below:

```console
$ cat syscall.txt
2705  openat(AT_FDCWD, "/root/.ash_history", O_WRONLY|O_CREAT|O_APPEND, 0600) = 3
2895  execve("/app", ["/app"], 0x559425259660 /* 6 vars */) = 0
2895  openat(AT_FDCWD, "/sys/kernel/mm/transparent_hugepage/hpage_pmd_size", O_RDONLY) = 3
2895  openat(AT_FDCWD, "/etc/localtime", O_RDONLY) = 3
2895  openat(AT_FDCWD, "/app", O_RDONLY|O_CLOEXEC) = 3
2895  openat(AT_FDCWD, "/app", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 3

2900  execve("/app", ["/app"], 0x559425259660 /* 6 vars */) = 0
2900  openat(AT_FDCWD, "/sys/kernel/mm/transparent_hugepage/hpage_pmd_size", O_RDONLY) = 3
2900  openat(AT_FDCWD, "/etc/localtime", O_RDONLY) = 3
2900  openat(AT_FDCWD, "/app", O_RDONLY|O_CLOEXEC) = 3
2902  openat(AT_FDCWD, "/app", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = -1 ETXTBSY (Text file busy)
```

If we compare the two `openat` system calls that pid 2895 and pid 2902 made, one
was able to return a valid fd while the other gives -1 with errno `ETXTBSY`.

```
2895  openat(AT_FDCWD, "/app", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = 3
2902  openat(AT_FDCWD, "/app", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0666) = -1 ETXTBSY (Text file busy)
```

## Continue with ftrace

[ftrace][7] is able to trace functions and function graphs so we get better
visibility into what is going on in kernel. In the case above, we knew that 
`openat` was giving `ETXTBSY`. We can look further what happened exactly within
that `openat` system call.

To set it up, we need to be clear on how exactly we need to do it:

* The function we are going to trace is `__x64_sys_openat` (I'm running amd64)
* It should trace child processes as well (like what `strace` does).
* We need to do `function_graph` tracer because we care about call stack.

> The reason why it is important to trace child processes is two folds: 1) It is
> more convenient to trace `/bin/sh` because we can find its pid after we launch
> it but before we run `/app`. 2) Go programs typically have multiple os threads
> so even if we can put a `sleep` in my `/app` program to make it possible to
> look up its pid, the actual `openat` may still run on a different os thread.

We can do all of them with steps like below:

```console
$ echo nop > /sys/kernel/tracing/current_tracer
$ echo __x64_sys_openat > /sys/kernel/tracing/set_graph_function
$ echo function-fork >> /sys/kernel/tracing/trace_options
$ echo <pid> > /sys/kernel/tracing/set_ftrace_pid
$ echo function_graph > /sys/kernel/tracing/current_tracer
... (run the app)
$ cat /sys/kernel/tracing/trace > output
```

I have already captured one [example][8]. From our strace earlier, there were
9 `openat` system calls in total, and the one which showed `ETXTBSY` is the last
one:

```
 0)               |  __x64_sys_openat() {
 0)               |    do_sys_open() {
 0)               |      do_filp_open() {
 0)               |        path_openat() {
 0)               |          do_last() {
 0)               |            vfs_open() {
 0)               |              do_dentry_open() {
 0)               |                ovl_open [overlay]() {
 0)               |                  ovl_open_realfile [overlay]() {
 0)               |                    open_with_fake_path() {
 0)               |                      do_dentry_open() {
 0)               |                        path_get() {
 0)   0.201 us    |                          mntget();
 0)   0.471 us    |                        }
 0)               |                        path_put()
...
```

> I made a significant simplication here (for the sake of saving some space).

In short, it calls into `do_dentry_open` -> `ovl_open` (overlay kernel module) and
then failed in the second `do_dentry_open` call.

```c
// fs/open.c
static int do_dentry_open(struct file *f,
			  struct inode *inode,
			  int (*open)(struct inode *, struct file *))
{
    ...
	if (f->f_mode & FMODE_WRITE && !special_file(inode->i_mode)) {
		error = get_write_access(inode);
		if (unlikely(error))
			goto cleanup_file;
		error = __mnt_want_write(f->f_path.mnt);
		if (unlikely(error)) {
			put_write_access(inode);
			goto cleanup_file;
		}
		f->f_mode |= FMODE_WRITER;
	}
```

Since we didn't see `__mnt_want_write` in function graph (while it is there in
the first `do_dentry_open`), we can be certain that `get_write_access` failed.

```c
// include/linux/fs.h
static inline int get_write_access(struct inode *inode)
{
	return atomic_inc_unless_negative(&inode->i_writecount) ? 0 : -ETXTBSY;
}
```

When the `i_writecount` from the given `inode` is negative, it fails with
`ETXTBSY`.

> You might wonder, why there are two `do_dentry_open`s, and do they open with
> the same file? To give a little bit context here, the first `do_dentry_open`
> opens the file as seen from overlayfs, and the second `do_dentry_open` opens
> the real file in the underlying filesystem.

## End with bpftrace

The question now comes to `i_writecount`, if it is negative, then what value is
it, and most importantly, how does this value change? To answer that question,
we need to rely on [kprobe][9]. I have a [simple bpftrace program][10] which can
answer all of these questions.

### do_filp_open

```c
kretprobe:do_filp_open /comm == "sh" || comm == "app"/ {
    $file = (struct file *)retval;
    printf("[ret do_filp_open     ] %s at inode[%p]: %d %d\n", comm, $file->f_inode, $file->f_inode->i_writecount.counter, $file->f_inode->i_ino);
}
```

`do_filp_open` gives back the `file` pointer on return, and we can examine the
inode address, its `i_writecount` and inode number (for correlation).

### do_open_execat

```c
kretprobe:do_open_execat /comm == "sh" || comm == "app"/ {
    $file = (struct file *)retval;
    printf("[ret do_open_execat   ] %s at inode[%p]: %d %d\n", comm, $file->f_inode, $file->f_inode->i_writecount.counter, $file->f_inode->i_ino);
}
```

`do_open_execat` is executed before the binary is loaded into memory. Before it
returns, it decreases `i_writecount`.

```c
// fs/exec.c
static struct file *do_open_execat(int fd, struct filename *name, int flags)
{
    ...
==> err = deny_write_access(file);
	if (err)
		goto exit;
    ...
```

### free_bprm

```c
kprobe:free_bprm /comm == "sh" || comm == "app"/ {
    $bprm = (struct linux_binprm *)arg0;
    printf("[free_bprm            ] %s at inode[%p]: %d %d\n", comm, $bprm->file->f_inode, $bprm->file->f_inode->i_writecount.counter, $bprm->file->f_inode->i_ino);
}
```

`free_bprm` is executed when the binary is completely mapped into memory. It
increases its `i_writecount` before it returns:

```c
// fs/exec.c
static void free_bprm(struct linux_binprm *bprm)
{
    ...
    if (bprm->file) {
==>		allow_write_access(bprm->file);
		fput(bprm->file);
	}
    ...
}
```

### mmap_region

```c
kprobe:mmap_region /comm == "sh" || comm == "app"/ {
    $file = (struct file *)arg0;
    if ($file != 0 && $file->f_inode != 0) {
        printf("[mmap_region          ] %s at inode[%p]: %d %d\n", comm, $file->f_inode, $file->f_inode->i_writecount.counter, $file->f_inode->i_ino);
    }
}
```

`mmap_region` sets up the mmap for one of the specific region. As you may guess,
the program has multiple regions and this function is going to be called
multiple times. This function, when called successfully, is going to decrease
`i_writecount` first, but then increase `i_writecount` when it is completed.

```c
// mm/mmap.c
unsigned long mmap_region(struct file *file, unsigned long addr,
		unsigned long len, vm_flags_t vm_flags, unsigned long pgoff,
		struct list_head *uf)
{
    ...
    if (file) {
		if (vm_flags & VM_DENYWRITE) {
==>			error = deny_write_access(file);
			if (error)
				goto free_vma;
		}
    ...
    vma_link(mm, vma, prev, rb_link, rb_parent);
	/* Once vma denies write, undo our temporary denial count */
	if (file) {
		if (vm_flags & VM_SHARED)
			mapping_unmap_writable(file->f_mapping);
		if (vm_flags & VM_DENYWRITE)
==>			allow_write_access(file);
	}
	file = vma->vm_file;
```

You might end up with thinking that `mmap_region` isn't going to do anything
with the given `file` regarding `i_writecount`, but that's not right, `vma_link`
does decrease `i_writecount` on `vma->vm_file`.

```c
// mm/mmap.c
static void __vma_link_file(struct vm_area_struct *vma)
{
    ...
    file = vma->vm_file;
    if (file) {
		struct address_space *mapping = file->f_mapping;

		if (vma->vm_flags & VM_DENYWRITE)
==>			atomic_dec(&file_inode(file)->i_writecount);
        ...
    }
    ...
}

static void vma_link(struct mm_struct *mm, struct vm_area_struct *vma,
			struct vm_area_struct *prev, struct rb_node **rb_link,
			struct rb_node *rb_parent)
{
    ...
    __vma_link(mm, vma, prev, rb_link, rb_parent);
	__vma_link_file(vma);
    ...
}
```

Essentially, after `mmap_region`, we would expect that `i_writecount` will be
decreased by one.

### ovl_mmap

```c
kprobe:ovl_mmap /comm == "sh" || comm == "app"/ {
    $file = (struct file *)arg0;
    printf("[ovl_mmap(file)       ] %s at inode[%p]: %d %d\n", comm, $file->f_inode, $file->f_inode->i_writecount.counter, $file->f_inode->i_ino);
    printf("[ovl_mmap(realfile)   ] %s at inode[%p]: %d %d\n", comm, ((struct file *)$file->private_data)->f_inode, ((struct file *)$file->private_data)->f_inode->i_writecount.counter, ((struct file *)$file->private_data)->f_inode->i_ino);
}
```

`ovl_mmap` is registered as `mmap` file operation for any files on overlay
file system. As `mmap_region` calls into this function, we can print information
about the `i_writecount` here regarding `file` (in overlay) and `realfile` (in
underlying file system).

```c
// fs/overlayfs/file.c
static int ovl_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct file *realfile = file->private_data;
    ...
    vma->vm_file = get_file(realfile)
    ...
    ret = call_mmap(vma->vm_file, vma);
}
```

It puts the `realfile` into `vma->vm_file` (remember that `vma_link` is going to
decrease its `i_write_count`) and then calls `mmap` of the `realfile`.

### Other functions

```c
kprobe:ovl_open_realfile /comm == "sh" || comm == "app"/ {
    $file = (struct file *)arg0;
    printf("[ovl_open_realfile    ] %s at inode[%p]: %d %d\n", comm, $file->f_inode, $file->f_inode->i_writecount.counter, $file->f_inode->i_ino);
}

kretprobe:ovl_open_realfile /comm == "sh" || comm == "app"/ {
    $file = (struct file *)retval;
    printf("[ret ovl_open_realfile] %s at inode[%p]: %d %d\n", comm, $file->f_inode, $file->f_inode->i_writecount.counter, $file->f_inode->i_ino);
}

kprobe:do_dentry_open /comm == "sh" || comm == "app"/ {
    $inode = (struct inode *)arg1;
    printf("[do_dentry_open       ] %s at inode[%p]: %d %d\n", comm, $inode, $inode->i_writecount.counter, $inode->i_ino);
}

kprobe:ovl_setattr /comm == "sh" || comm == "app"/ {
    $dentry = (struct dentry *)arg0;
    printf("[ovl_setattr          ] %s at inode[%p]: %d %d\n", comm, $dentry->d_inode, $dentry->d_inode->i_writecount.counter, $dentry->d_inode->i_ino);
}
```

These are just used to show the `i_writecount`.

### Put it in action

Once we run this bpftrace script, and then run the `/app` twice as we we did in
the beginning, we'll see something like below:

#### The first exec

```
[do_dentry_open       ] sh at inode[0xffff9719b7ea1040]: 0 675400
[ovl_open_realfile    ] sh at inode[0xffff9719b7ea1040]: 0 675400
[do_dentry_open       ] sh at inode[0xffff9719ba6600e8]: 0 675400
[ret ovl_open_realfile] sh at inode[0xffff9719ba6600e8]: 0 675400
[ret do_filp_open     ] sh at inode[0xffff9719b7ea1040]: 0 675400
[ret do_open_execat   ] sh at inode[0xffff9719b7ea1040]: -1 675400
[mmap_region          ] app at inode[0xffff9719b7ea1040]: -1 675400
[ovl_mmap(file)       ] app at inode[0xffff9719b7ea1040]: -2 675400
[ovl_mmap(realfile)   ] app at inode[0xffff9719ba6600e8]: 0 675400
[mmap_region          ] app at inode[0xffff9719b7ea1040]: -1 675400
[ovl_mmap(file)       ] app at inode[0xffff9719b7ea1040]: -2 675400
[ovl_mmap(realfile)   ] app at inode[0xffff9719ba6600e8]: -1 675400
[mmap_region          ] app at inode[0xffff9719b7ea1040]: -1 675400
[ovl_mmap(file)       ] app at inode[0xffff9719b7ea1040]: -2 675400
[ovl_mmap(realfile)   ] app at inode[0xffff9719ba6600e8]: -2 675400
[free_bprm            ] app at inode[0xffff9719b7ea1040]: -1 675400
```

> Below, I use `1040` to represent the inode at address `0xffff9719b7ea1040` and
> `00e8` to represent the inode at address `0xffff9719ba6600e8`.


![i_writecount_exec](/images/inode_writecount_exec.svg)

There are 2 inodes, namingly `1040` (addr) and `00e8` (addr). Both of them
have the same inode number `675400`. When we initially open `/app`, it is on
overlayfs, so the inode `1040` is the one from overlayfs super block, once it
returns from `ovl_open_realfile`, `00e8` shows up, and that is the inode from
the underlying file system (ext4 for example).

When it returns from `do_open_execat`, we do see that the `i_writecount` of
inode `1040` turns into `-1`.

Now it goes into `mmap_region`, as we talked earlier, it tries to decrease the
`i_writecount` of the incoming inode `1040`, so that's why `ovl_mmap` shows that
inode `1040` now has `i_write_count` value as -2, and it sets `vma->file` to the
real file `00e8`, so that `vma_link` will decrease its `i_writecount` by 1 which
is going to be -1 (as we can confirm in the second iteration of `mmap_region`).
Further, since `mmap_region` increases the `i_writecount` of inode `1040` before
it returns, its `i_writecount` remains at -1.

After `mmap_region` function runs 3 times, eventually, it is going to set
`i_writecount` of `00e8` to be -3 while `i_writecount` of `1040` remains at -1.

In the end, `free_bprm` is going to release the write of inode `1040`, so after
it returns, the `i_writecount` if inode `1040` becomes 0.

#### Open with read only

```
[do_dentry_open       ] app at inode[0xffff9719b7ea1040]: 0 675400
[ovl_open_realfile    ] app at inode[0xffff9719b7ea1040]: 0 675400
[do_dentry_open       ] app at inode[0xffff9719ba6600e8]: -3 675400
[ret ovl_open_realfile] app at inode[0xffff9719ba6600e8]: -3 675400
[ret do_filp_open     ] app at inode[0xffff9719b7ea1040]: 0 675400
```

We can confirm here that inode `1040` has `i_writecount` as 0, and inode `00e8`
has `i_writecount` as -3 (due to the fact that `vma_link` was called 3 times
above).

#### Open with read write

```console
[do_dentry_open       ] app at inode[0xffff9719b7ea1040]: 0 675400
[ovl_open_realfile    ] app at inode[0xffff9719b7ea1040]: 1 675400
[do_dentry_open       ] app at inode[0xffff9719b7e7d688]: 0 675462
[ret ovl_open_realfile] app at inode[0xffff9719b7e7d688]: 1 675462
[ovl_setattr          ] app at inode[0xffff9719b7ea1040]: 2 675400
[ret do_filp_open     ] app at inode[0xffff9719b7ea1040]: 1 675400
```

As inode `1040` has `i_writecount` as 0, it still allows writes. When the write
is permitted, overlayfs is going to create a new file in upper layer, and that's
why inode `d688` shows up: It's the new inode in the underlying file system.

You'll see that the new inode has a different number `675462` because overlayfs
actually creates a new file in underlying file system. However, users will still
see the same inode `1040` with number `675400` 

#### The second exec

```
[do_dentry_open       ] sh at inode[0xffff9719b7ea1040]: 0 675400
[ovl_open_realfile    ] sh at inode[0xffff9719b7ea1040]: 0 675400
[do_dentry_open       ] sh at inode[0xffff9719b7e7d688]: 0 675462
[ret ovl_open_realfile] sh at inode[0xffff9719b7e7d688]: 0 675462
[ret do_filp_open     ] sh at inode[0xffff9719b7ea1040]: 0 675400
[ret do_open_execat   ] sh at inode[0xffff9719b7ea1040]: -1 675400
[mmap_region          ] app at inode[0xffff9719b7ea1040]: -1 675400
[ovl_mmap(file)       ] app at inode[0xffff9719b7ea1040]: -2 675400
[ovl_mmap(realfile)   ] app at inode[0xffff9719b7e7d688]: 0 675462
[mmap_region          ] app at inode[0xffff9719b7ea1040]: -1 675400
[ovl_mmap(file)       ] app at inode[0xffff9719b7ea1040]: -2 675400
[ovl_mmap(realfile)   ] app at inode[0xffff9719b7e7d688]: -1 675462
[mmap_region          ] app at inode[0xffff9719b7ea1040]: -1 675400
[ovl_mmap(file)       ] app at inode[0xffff9719b7ea1040]: -2 675400
[ovl_mmap(realfile)   ] app at inode[0xffff9719b7e7d688]: -2 675462
[free_bprm            ] app at inode[0xffff9719b7ea1040]: -1 675400
```

This is exactly the same as the first run. In the end, inode `1040` has 0
`i_writecount` and `d688` (the new inode in underlying file system) has
`i_writecount` as -3.

#### Open with read only (2nd)

```
[do_dentry_open       ] app at inode[0xffff9719b7ea1040]: 0 675400
[ovl_open_realfile    ] app at inode[0xffff9719b7ea1040]: 0 675400
[do_dentry_open       ] app at inode[0xffff9719b7e7d688]: -3 675462
[ret ovl_open_realfile] app at inode[0xffff9719b7e7d688]: -3 675462
[ret do_filp_open     ] app at inode[0xffff9719b7ea1040]: 0 675400
```

This is a read only open, so the `i_writecount` value doesn't matter here and it
succeeded as usual.

#### Open with read write (2nd)

```
[do_dentry_open       ] app at inode[0xffff9719b7ea1040]: 0 675400
[ovl_open_realfile    ] app at inode[0xffff9719b7ea1040]: 1 675400
[do_dentry_open       ] app at inode[0xffff9719b7e7d688]: -3 675462
[ret ovl_open_realfile] app at inode[(nil)]: 0 0
[ret do_filp_open     ] app at inode[(nil)]: 0 0
```

This is when `text file busy` error happens, because overlayfs sees that the
file is already in upper layer (the inode `d688`, previously copied up). When we
try to open it with `O_RDWR` again with inode `d688`, it fails with ETXTBSY.

## Summary

While the story when being told, seems to be straightforward, there were a lot
of trials and errors. It literally took me several days to figure out all the
functions to trace, however it was also a great relief / joy when everything was
connected in the end. Thanks for reading!

> You can also use the exact same bpftrace script to understand why ubuntu 22.04
> doesn't have this problem. I'll leave it as an exercise to you. I promise it
> is going to be fun. Let me know what you are going to find out in the comments
> below.

[1]: https://github.com/googleContainerTools/kaniko
[2]: https://github.com/GoogleContainerTools/kaniko?tab=readme-ov-file#known-issues
[3]: https://github.com/zhouhaibing089/txtbsy/blob/main/main.go
[4]: https://releases.ubuntu.com/focal/
[5]: https://man7.org/linux/man-pages/man1/strace.1.html
[6]: https://github.com/zhouhaibing089/txtbsy/blob/main/ubuntu-2004.syscall.txt
[7]: https://github.com/torvalds/linux/blob/master/Documentation/trace/ftrace.rst
[8]: https://github.com/zhouhaibing089/txtbsy/blob/main/ubuntu-2004.ftrace.txt
[9]: https://docs.kernel.org/trace/kprobes.html
[10]: https://github.com/zhouhaibing089/txtbsy/blob/main/file_at_open_or_exec.bt
