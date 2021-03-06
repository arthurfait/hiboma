# 5-2 プロセスからのカーネル呼び出し

 * int 0x80 or sysenter
   * レジスタの退避など
     * システムコール呼び出し
   * do_syscall_trace
   * do_notify_resume
   * do_signal
     * EINTR の実装部分が興味深い
 * iret or sysexit

システムコール呼び出しのパスが thread_info_flags ( _TIF_FOO_BAR なフラグ) によっていろいろ挙動が変わるのがポイント

```c
/*
 * thread information flags
 * - these are process state flags that various assembly files
 *   may need to access
 * - pending work-to-be-done flags are in LSW
 * - other flags in MSW
 * Warning: layout of LSW is hardcoded in entry.S
 */
#define TIF_SYSCALL_TRACE	0	/* syscall trace active */
#define TIF_NOTIFY_RESUME	1	/* callback before returning to user */
#define TIF_SIGPENDING		2	/* signal pending */
#define TIF_NEED_RESCHED	3	/* rescheduling necessary */
#define TIF_SINGLESTEP		4	/* reenable singlestep on user return*/
#define TIF_IRET		5	/* force IRET */
#define TIF_SYSCALL_EMU		6	/* syscall emulation active */
#define TIF_SYSCALL_AUDIT	7	/* syscall auditing active */
#define TIF_SECCOMP		8	/* secure computing */
#define TIF_MCE_NOTIFY		10	/* notify userspace of an MCE */
#define TIF_USER_RETURN_NOTIFY	11	/* notify kernel of userspace return */
#define TIF_NOTSC		16	/* TSC is not accessible in userland */
#define TIF_IA32		17	/* 32bit process */
#define TIF_FORK		18	/* ret_from_fork */
#define TIF_MEMDIE		20
#define TIF_DEBUG		21	/* uses debug registers */
#define TIF_IO_BITMAP		22	/* uses I/O bitmap */
#define TIF_FREEZE		23	/* is freezing for suspend */
#define TIF_FORCED_TF		24	/* true if TF in eflags artificially */
#define TIF_BLOCKSTEP		25	/* set when we want DEBUGCTLMSR_BTF */
#define TIF_LAZY_MMU_UPDATES	27	/* task is updating the mmu lazily */
#define TIF_SYSCALL_TRACEPOINT	28	/* syscall tracepoint instrumentation */
```
 
## 5.2.1 システムコール呼び出し規約

 * int      => iret
 * sysenter => sysexit

exit(2) を アセンブラ(リ) で書くと次のようなコードになる

```asm
;; http://www.tutorialspoint.com/assembly_programming/assembly_system_calls.htm
;; exit(1) と同じ
mov	eax,1		; system call number (sys_exit)
int	0x80		; call kernel
```

write(2) を書いた場合は次のようなコードになる

```asm
;; http://www.tutorialspoint.com/assembly_programming/assembly_system_calls.htm
;; write(1, msg, 4) と同じ
mov	edx,4		; message length
mov	ecx,msg		; message to write
mov	ebx,1		; file descriptor (stdout)
mov	eax,4		; system call number (sys_write)
int	0x80		; call kernel
```

### syscall で getpid(2)

glibc の syscall(3) からシステムコールを呼び出すす場合

```c
#if 0
#!/bin/bash
CFLAGS="-O2 -std=gnu99 -W -Wall -fPIE -D_FORTIFY_SOURCE=2"
o=`basename $0`
o=".${o%.*}"
gcc ${CFLAGS} -o $o $0 && ./$o $*; exit
#endif

#define _GNU_SOURCE         /* See feature_test_macros(7) */
#include <unistd.h>
#include <sys/syscall.h>   /* For SYS_xxx definitions */
#include <stdio.h>

int main()
{
	long retval = syscall(39);
	printf("getpid: %ld\n", retval);
	return 0;
}
```

syscall の glibc の実装は次の通り (sysdeps/unix/sysv/linux/x86_64/syscall.S)

```c
/* Copyright (C) 2001, 2003 Free Software Foundation, Inc.
   This file is part of the GNU C Library.

   The GNU C Library is free software; you can redistribute it and/or
   modify it under the terms of the GNU Lesser General Public
   License as published by the Free Software Foundation; either
   version 2.1 of the License, or (at your option) any later version.

   The GNU C Library is distributed in the hope that it will be useful,
   but WITHOUT ANY WARRANTY; without even the implied warranty of
   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
   Lesser General Public License for more details.

   You should have received a copy of the GNU Lesser General Public
   License along with the GNU C Library; if not, write to the Free
   Software Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA
   02111-1307 USA.  */

#include <sysdep.h>

/* Please consult the file sysdeps/unix/sysv/linux/x86-64/sysdep.h for
   more information about the value -4095 used below.  */

/* Usage: long syscall (syscall_number, arg1, arg2, arg3, arg4, arg5, arg6)
   We need to do some arg shifting, the syscall_number will be in
   rax.  */


        .text
ENTRY (syscall)
        movq %rdi, %rax         /* Syscall number -> rax.  */
        movq %rsi, %rdi         /* shift arg1 - arg5.  */
        movq %rdx, %rsi
        movq %rcx, %rdx
        movq %r8, %r10
        movq %r9, %r8 
        movq 8(%rsp),%r9        /* arg6 is on the stack.  */
        syscall                 /* Do the system call.  */
        cmpq $-4095, %rax       /* Check %rax for error.  */
        jae SYSCALL_ERROR_LABEL /* Jump to error handler if error.  */
L(pseudo_end):
        ret                     /* Return to caller.  */

PSEUDO_END (syscall)
```

#### glibc の getpid の実装を追う

INTERNAL_SYSCALL でカーネルモードに移行する

```c
/* Copyright (C) 2003, 2004 Free Software Foundation, Inc.
   This file is part of the GNU C Library.
   Contributed by Ulrich Drepper <drepper@redhat.com>, 2003.

   The GNU C Library is free software; you can redistribute it and/or
   modify it under the terms of the GNU Lesser General Public
   License as published by the Free Software Foundation; either
   version 2.1 of the License, or (at your option) any later version.

   The GNU C Library is distributed in the hope that it will be useful,
   but WITHOUT ANY WARRANTY; without even the implied warranty of
   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
   Lesser General Public License for more details.

   You should have received a copy of the GNU Lesser General Public
   License along with the GNU C Library; if not, write to the Free
   Software Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA
   02111-1307 USA.  */

#include <unistd.h>
#include <tls.h>
#include <sysdep.h>


#ifndef NOT_IN_libc
static inline __attribute__((always_inline)) pid_t really_getpid (pid_t oldval);

static inline __attribute__((always_inline)) pid_t
really_getpid (pid_t oldval)
{
  if (__builtin_expect (oldval == 0, 1))
    {
      pid_t selftid = THREAD_GETMEM (THREAD_SELF, tid);
      if (__builtin_expect (selftid != 0, 1))
	return selftid;
    }

  INTERNAL_SYSCALL_DECL (err);
  // ->
  pid_t result = INTERNAL_SYSCALL (getpid, err, 0);

  /* We do not set the PID field in the TID here since we might be
     called from a signal handler while the thread executes fork.  */
  if (oldval == 0)
    THREAD_SETMEM (THREAD_SELF, tid, result);
  return result;
}
#endif

pid_t
__getpid (void)
{
#ifdef NOT_IN_libc
  INTERNAL_SYSCALL_DECL (err);
  pid_t result = INTERNAL_SYSCALL (getpid, err, 0);
#else
  pid_t result = THREAD_GETMEM (THREAD_SELF, pid);
  if (__builtin_expect (result <= 0, 0))
    result = really_getpid (result);
#endif
  return result;
}

libc_hidden_def (__getpid)
weak_alias (__getpid, getpid)
libc_hidden_def (getpid)
```

### INTERNAL_SYSCALL

int $0x80 を直で呼び出す場合の実装は次の通り

```c
# define INTERNAL_SYSCALL(name, err, nr, args...) \
  ({									      \
    register unsigned int resultvar;					      \
    EXTRAVAR_##nr							      \
    asm volatile (							      \
    LOADARGS_##nr							      \
    "movl %1, %%eax\n\t"						      \
    "int $0x80\n\t"							      \
    RESTOREARGS_##nr							      \
    : "=a" (resultvar)							      \
    : "i" (__NR_##name) ASMFMT_##nr(args) : "memory", "cc");		      \
    (int) resultvar; })
```

_dl_sysinfo_int80 でカーネルモードに移行。これ なんだっけ?

```c
#  define INTERNAL_SYSCALL(name, err, nr, args...) \
  ({									      \
    register unsigned int resultvar;					      \
    EXTRAVAR_##nr							      \
    asm volatile (							      \
    LOADARGS_##nr							      \
    "movl %1, %%eax\n\t"						      \
    "call *_dl_sysinfo\n\t"						      \
    RESTOREARGS_##nr							      \
    : "=a" (resultvar)							      \
    : "i" (__NR_##name) ASMFMT_##nr(args) : "memory", "cc");		      \
    (int) resultvar; })
```

### インラインアセンブラで int 0x80 で getpid(2)

```c
#if 0
#!/bin/bash
CFLAGS="-O0 -std=gnu99 -W -Wall"
o=`basename $0`
o=".${o%.*}"
gcc ${CFLAGS} -o $o $0 && ./$o $*; exit
#endif

#include <stdio.h>

int main()
{
	long rax;

	/* /usr/include/asm/unistd_32.h */
	__asm__(
		"mov $20, %%rax;"
		"int $0x80;"
		"mov %%rax, %0;"
		:"=r"(rax)
		);

	printf("getpid: %ld\n", rax);
	return 0;
}
```

## 5.2.2 int 0x80/iret によるシステムコール

2.6.32 ではコードが変わりまくりなので、書籍に従って 2.6.15 を読む

```asm
# int の時点で
# - SS
# - ESP
# - EFLAGS
# - CS
# - EIP
# が既にレジスタに積まれている
ENTRY(system_call)
    # 2. eax を退避
	pushl %eax			# save orig_eax

    # 3. その他汎用レジスタを pushl で退避
	SAVE_ALL

    # 4. 現在実行中のタスクのスタックを %ebp にセット   
	GET_THREAD_INFO(%ebp)
					# system call tracing in operation / emulation
	/* Note, _TIF_SECCOMP is bit number 8, and so it needs testw and not testb */
	testw $(_TIF_SYSCALL_EMU|_TIF_SYSCALL_TRACE|_TIF_SECCOMP|_TIF_SYSCALL_AUDIT),TI_flags(%ebp)
    # 5. ↑の testw の結果が 0 でないなら ptrace されているのでジャンプ
	jnz syscall_trace_entry

    # 6. eax がシステムコールの最大の番号を超えてたら ENOSYS かえす
	cmpl $(nr_syscalls), %eax
	jae syscall_badsys

    # 7 システムコールテーブルの %eax 番目を call する
syscall_call:
	call *sys_call_table(,%eax,4)

    # 8. システムコールの戻り値を %eax に入れとく
	movl %eax,EAX(%esp)		# store the return value
syscall_exit:
    # 9. cli で IF = 0 ハードウェア割り込みを無視
	cli				# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	movl TI_flags(%ebp), %ecx
	testw $_TIF_ALLWORK_MASK, %cx	# current->work
	jne syscall_exit_work

restore_all:
    # 10. レジスタの復帰
	movl EFLAGS(%esp), %eax		# mix EFLAGS, SS and CS
	# Warning: OLDSS(%esp) contains the wrong/random values if we
	# are returning to the kernel.
	# See comments in process.c:copy_thread() for details.
	movb OLDSS(%esp), %ah
	movb CS(%esp), %al
	andl $(VM_MASK | (4 << 8) | 3), %eax
	cmpl $((4 << 8) | 3), %eax
	je ldt_ss			# returning to user-space with LDT SS
restore_nocheck:
	RESTORE_REGS
	addl $4, %esp
    # 13 ユーザモードに復帰
1:	iret
.section .fixup,"ax"
iret_exc:
	sti
	pushl $0			# no error code
	pushl $do_iret_error
	jmp error_code
.previous
.section __ex_table,"a"
	.align 4
	.long 1b,iret_exc
.previous
```

#### syscall_table.S

 * システムコールのテーブル
 * 物理メモリ上に連続して並ぶ関数ポインタ群

```asm
.data
ENTRY(sys_call_table)
	.long sys_restart_syscall	/* 0 - old "setup()" system call, used for restarting */
	.long sys_exit
	.long sys_fork
	.long sys_read
	.long sys_write
	.long sys_open		/* 5 */
	.long sys_close
	.long sys_waitpid
	.long sys_creat
	.long sys_link
	.long sys_unlink	/* 10 */
	.long sys_execve
	.long sys_chdir
	.long sys_time
	.long sys_mknod
// ...    
```

#### syscall_exit_work

```asm
syscall_exit_work:
	testb $(_TIF_SYSCALL_TRACE|_TIF_SYSCALL_AUDIT|_TIF_SINGLESTEP), %cl
	jz work_pending
    # IF = 0 ハードウェア割り込み許可
	sti				# could let do_syscall_trace() call
					# schedule() instead
	movl %esp, %eax
	movl $1, %edx
    # ptrace とかあれこれ
	call do_syscall_trace
	jmp resume_userspace
```

#### syscall_badsys

 * eax (= ERRNO) に ENOSYS 入れてユーザ空間に返す
 * resume_userspace で復帰

```asm
syscall_badsys:
	movl $-ENOSYS,EAX(%esp)
	jmp resume_userspace
```

#### syscall_trace_entry

do_syscall_trace に移ってデバッギにあれこて通知だすぽい

```asm
syscall_trace_entry:
	movl $-ENOSYS,EAX(%esp)
	movl %esp, %eax
	xorl %edx,%edx
	call do_syscall_trace
	cmpl $0, %eax
	jne resume_userspace		# ret != 0 -> running under PTRACE_SYSEMU,
					# so must skip actual syscall
	movl ORIG_EAX(%esp), %eax
	cmpl $(nr_syscalls), %eax
	jnae syscall_call
	jmp syscall_exit

	# perform syscall exit tracing
	ALIGN
```

do_syscall_trace の ptrace_notify がおもしろそう

#### resume_userspace

```asm
ENTRY(resume_userspace)
 	cli				# make sure we don't miss an interrupt
					# setting need_resched or sigpending
					# between sampling and the iret
	movl TI_flags(%ebp), %ecx
	andl $_TIF_WORK_MASK, %ecx	# is there any work to be done on
					# int/exception return?
	jne work_pending
	jmp restore_all
```

## do_signal

do_notify_resume から呼ばれる

システムコール復帰時にシグナルを受信していないかを見る

```c
/*
 * Note that 'init' is a special process: it doesn't get signals it doesn't
 * want to handle. Thus you cannot kill init even with a SIGKILL even by
 * mistake.
 */
static void do_signal(struct pt_regs *regs)
{
	struct k_sigaction ka;
	siginfo_t info;
	int signr;
	sigset_t *oldset;

	/*
	 * We want the common case to go fast, which is why we may in certain
	 * cases get here from kernel mode. Just return without doing anything
	 * if so.
	 * X86_32: vm86 regs switched out by assembly code before reaching
	 * here, so testing against kernel CS suffices.
	 */
	if (!user_mode(regs))
		return;

	if (current_thread_info()->status & TS_RESTORE_SIGMASK)
		oldset = &current->saved_sigmask;
	else
		oldset = &current->blocked;

    /* プロセスに届けられたシグナルを一個ずつ取り出して確認 */
	signr = get_signal_to_deliver(&info, &ka, regs, NULL);
	if (signr > 0) {
		/*
		 * Re-enable any watchpoints before delivering the
		 * signal to user space. The processor register will
		 * have been cleared if the watchpoint triggered
		 * inside the kernel.
		 */
		if (current->thread.debugreg7)
			set_debugreg(current->thread.debugreg7, 7);

		/* Whee! Actually deliver the signal.  */
		if (handle_signal(signr, &info, &ka, oldset, regs) == 0) {
			/*
			 * A signal was successfully delivered; the saved
			 * sigmask will have been stored in the signal frame,
			 * and will be restored by sigreturn, so we can simply
			 * clear the TS_RESTORE_SIGMASK flag.
			 */
			current_thread_info()->status &= ~TS_RESTORE_SIGMASK;
		}
		return;
	}

	/* Did we come from a system call? */
    /* resgs->orig_ax との比較 */
	if (syscall_get_nr(current, regs) >= 0) {
		/* Restart the system call - no handlers present */
        /* regs->ax から errno を取る */
		switch (syscall_get_error(current, regs)) {
		case -ERESTARTNOHAND:
		case -ERESTARTSYS:
        /* ハンドラが無い場合。？ */
		case -ERESTARTNOINTR:
			regs->ax = regs->orig_ax;
            /* インスクラションポインタを -2 するとどこ??? */
			regs->ip -= 2;
			break;

        /* sys_restart_syscall で再開する */
		case -ERESTART_RESTARTBLOCK:
            /* rax(システムコールの番号)を NR_restart_syscall にセット */
            /* iret した際に restart_syscall を再実行するはず */
			regs->ax = NR_restart_syscall;
			regs->ip -= 2;
			break;
		}
	}

	/*
	 * If there's no signal to deliver, we just put the saved sigmask
	 * back.
	 */
	if (current_thread_info()->status & TS_RESTORE_SIGMASK) {
		current_thread_info()->status &= ~TS_RESTORE_SIGMASK;
		sigprocmask(SIG_SETMASK, &current->saved_sigmask, NULL);
	}
}
```

_-ERESTART_RESTARTBLOCK でシステムコールを再開する

```c
/*
 * System call entry points.
 */

SYSCALL_DEFINE0(restart_syscall)
{
	struct restart_block *restart = &current_thread_info()->restart_block;
	return restart->fn(restart);
}
```

ip を -2 するのは下記の通り?

```
int 0x80 / sysenter            // -2
iret     / sysexit             // -1
システムコール呼び出し後の命令 // 0
```