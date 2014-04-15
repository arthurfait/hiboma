# /usr/sbin/purge

```
$ pkgutil --file-info /usr/sbin/purge
volume: /
path: /usr/sbin/purge

pkgid: com.apple.pkg.BSD
pkg-version: 10.9.0.1.1.1306847324
install-time: 1383202560
uid: 0
gid: 0
mode: 755

pkgid: com.apple.pkg.update.os.10.9.2.13C64.delta
pkg-version: 1.0.0.0.1.1306847324
install-time: 1394500291
uid: 0
gid: 0
mode: 755
```

## USAGE

```
purge(8)                  BSD System Manager's Manual                 purge(8)

NAME
     purge -- force disk cache to be purged (flushed and emptied)

SYNOPSIS
     purge

DESCRIPTION
     Purge can be used to approximate initial boot conditions with a cold disk
     buffer cache for performance analysis. It does not affect anonymous mem-
     ory that has been allocated through malloc, vm_allocate, etc.

SEE ALSO
     sync(8), malloc(3)

                              September 20, 2005
```

anon なメモリは対象外。

dtruss 取ると ___vfs_purge___ 呼んでる

```
 3047/0x4204:  vfs_purge(0x7FFF53CDEBA0, 0x7FFF53CDEBB0, 0x7FFF53CDEC60)		 = 0 0
```

## vfs_purge

xnu-2422.1.72/bsd/vfs/vfs_syscalls.c

 * vfs_purge
   * vfs_iterate で ファイルシステムをイテレートするコールバックを呼ぶ
 * -> vfs_purge_callback
   * vnode_iterate で vnode をイテレートするコールバックを呼ぶ
 * -> vnode_purge_callback

Linux で言う所の address_space に map された領域を破棄する仕組みぽい

```c
/*
 * Purge buffer cache for simulating cold starts
 */
static int vnode_purge_callback(struct vnode *vp, __unused void *cargs)
{
	ubc_msync(vp, (off_t)0, ubc_getsize(vp), NULL /* off_t *resid_off */, UBC_PUSHALL | UBC_INVALIDATE);

	return VNODE_RETURNED;
}

static int vfs_purge_callback(mount_t mp, __unused void * arg)
{
	vnode_iterate(mp, VNODE_WAIT | VNODE_ITERATE_ALL, vnode_purge_callback, NULL);

	return VFS_RETURNED;
}

int
vfs_purge(__unused struct proc *p, __unused struct vfs_purge_args *uap, __unused int32_t *retval)
{
	if (!kauth_cred_issuser(kauth_cred_get()))
		return EPERM;

	vfs_iterate(0/* flags */, vfs_purge_callback, NULL);

	return 0;
}
```

ubc_msync

 * ubc = Unified Buffer Cache の略?
   * http://www.jp.netbsd.org/ja/docs/kernel/
 * vnode のメモリオブジェクトを invalidate する
   * invalidate_mapping_pages 相当かな
```c
/*
 * ubc_msync
 *
 * Clean and/or invalidate a range in the memory object that backs this vnode
 *
 * Parameters:	vp			The vnode whose associated ubc_info's
 *					associated memory object is to have a
 *					range invalidated within it
 *		beg_off			The start of the range, as an offset
 *		end_off			The end of the range, as an offset
 *		resid_off		The address of an off_t supplied by the
 *					caller; may be set to NULL to ignore
 *		flags			See ubc_msync_internal()
 *
 * Returns:	0			Success
 *		!0			Failure; an errno is returned
 *
 * Implicit Returns:
 *		*resid_off, modified	If non-NULL, the  contents are ALWAYS
 *					modified; they are initialized to the
 *					beg_off, and in case of an I/O error,
 *					the difference between beg_off and the
 *					current value will reflect what was
 *					able to be written before the error
 *					occurred.  If no error is returned, the
 *					value of the resid_off is undefined; do
 *					NOT use it in place of end_off if you
 *					intend to increment from the end of the
 *					last call and call iteratively.
 *
 * Notes:	see ubc_msync_internal() for more detailed information.
 *
 */
errno_t
ubc_msync(vnode_t vp, off_t beg_off, off_t end_off, off_t *resid_off, int flags)
{
        int retval;
	int io_errno = 0;
	
	if (resid_off)
	        *resid_off = beg_off;

        retval = ubc_msync_internal(vp, beg_off, end_off, resid_off, flags, &io_errno);

	if (retval == 0 && io_errno == 0)
	        return (EINVAL);
	return (io_errno);
}
```

ubc_msync_internal が強い

 * 下記のフラグによって writeback するページの内容が決まる
 * UBC_PUSHALL なので dirty なページと、通常のページキャッシュと全て writeback する様子

```c
#define	UBC_PUSHDIRTY	0x01	/* clean any dirty pages in the specified range to the backing store */
#define	UBC_PUSHALL	0x02	/* push both dirty and precious pages to the backing store */
#define	UBC_INVALIDATE	0x04	/* invalidate pages in the specified range... may be used with UBC_PUSHDIRTY/ALL */
#define	UBC_SYNC	0x08	/* wait for I/Os generated by UBC_PUSHDIRTY to complete */
```

vnode にマップされたページを破棄する感じ

```c
/*
 * Clean and/or invalidate a range in the memory object that backs this vnode
 *
 * Parameters:	vp			The vnode whose associated ubc_info's
 *					associated memory object is to have a
 *					range invalidated within it
 *		beg_off			The start of the range, as an offset
 *		end_off			The end of the range, as an offset
 *		resid_off		The address of an off_t supplied by the
 *					caller; may be set to NULL to ignore
 *		flags			MUST contain at least one of the flags
 *					UBC_INVALIDATE, UBC_PUSHDIRTY, or
 *					UBC_PUSHALL; if UBC_PUSHDIRTY is used,
 *					UBC_SYNC may also be specified to cause
 *					this function to block until the
 *					operation is complete.  The behavior
 *					of UBC_SYNC is otherwise undefined.
 *		io_errno		The address of an int to contain the
 *					errno from a failed I/O operation, if
 *					one occurs; may be set to NULL to
 *					ignore
 *
 * Returns:	1			Success
 *		0			Failure
 *
 * Implicit Returns:
 *		*resid_off, modified	The contents of this offset MAY be
 *					modified; in case of an I/O error, the
 *					difference between beg_off and the
 *					current value will reflect what was
 *					able to be written before the error
 *					occurred.
 *		*io_errno, modified	The contents of this offset are set to
 *					an errno, if an error occurs; if the
 *					caller supplies an io_errno parameter,
 *					they should be careful to initialize it
 *					to 0 before calling this function to
 *					enable them to distinguish an error
 *					with a valid *resid_off from an invalid
 *					one, and to avoid potentially falsely
 *					reporting an error, depending on use.
 *
 * Notes:	If there is no ubc_info associated with the vnode supplied,
 *		this function immediately returns success.
 *
 *		If the value of end_off is less than or equal to beg_off, this
 *		function immediately returns success; that is, end_off is NOT
 *		inclusive.
 *
 *		IMPORTANT: one of the flags UBC_INVALIDATE, UBC_PUSHDIRTY, or
 *		UBC_PUSHALL MUST be specified; that is, it is NOT possible to
 *		attempt to block on in-progress I/O by calling this function
 *		with UBC_PUSHDIRTY, and then later call it with just UBC_SYNC
 *		in order to block pending on the I/O already in progress.
 *
 *		The start offset is truncated to the page boundary and the
 *		size is adjusted to include the last page in the range; that
 *		is, end_off on exactly a page boundary will not change if it
 *		is rounded, and the range of bytes written will be from the
 *		truncate beg_off to the rounded (end_off - 1).
 */
static int
ubc_msync_internal(vnode_t vp, off_t beg_off, off_t end_off, off_t *resid_off, int flags, int *io_errno)
{
	memory_object_size_t	tsize;
	kern_return_t		kret;
	int request_flags = 0;
	int flush_flags   = MEMORY_OBJECT_RETURN_NONE;
	
	if ( !UBCINFOEXISTS(vp))
	        return (0);
	if ((flags & (UBC_INVALIDATE | UBC_PUSHDIRTY | UBC_PUSHALL)) == 0)
	        return (0);
	if (end_off <= beg_off)
	        return (1);

	if (flags & UBC_INVALIDATE)
	        /*
		 * discard the resident pages
		 */
		request_flags = (MEMORY_OBJECT_DATA_FLUSH | MEMORY_OBJECT_DATA_NO_CHANGE);

	if (flags & UBC_SYNC)
	        /*
		 * wait for all the I/O to complete before returning
		 */
	        request_flags |= MEMORY_OBJECT_IO_SYNC;

	if (flags & UBC_PUSHDIRTY)
	        /*
		 * we only return the dirty pages in the range
		 */
	        flush_flags = MEMORY_OBJECT_RETURN_DIRTY;

	if (flags & UBC_PUSHALL)
	        /*
		 * then return all the interesting pages in the range (both
		 * dirty and precious) to the pager
		 */
	        flush_flags = MEMORY_OBJECT_RETURN_ALL;

	beg_off = trunc_page_64(beg_off);
	end_off = round_page_64(end_off);
	tsize   = (memory_object_size_t)end_off - beg_off;

	/* flush and/or invalidate pages in the range requested */
	kret = memory_object_lock_request(vp->v_ubcinfo->ui_control,
					  beg_off, tsize,
					  (memory_object_offset_t *)resid_off,
					  io_errno, flush_flags, request_flags,
					  VM_PROT_NO_CHANGE);
	
	return ((kret == KERN_SUCCESS) ? 1 : 0);
}
```
