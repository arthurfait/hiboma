# mallopt(3)

 * glibc malloc(3) のチューニングや挙動を変える
 * http://man7.org/linux/man-pages/man3/mallopt.3.html

## M_PERTURB オプション

malloc/free する際に指定したバイトでチャンクを埋める

```
       M_PERTURB (since glibc 2.4)
              If this parameter is set to a nonzero value, then bytes of
              allocated memory (other than allocations via calloc(3)) are
              initialized to the complement of the value in the least
              significant byte of value, and when allocated memory is
              released using free(3), the freed bytes are set to the least
              significant byte of value.  This can be useful for detecting
              errors where programs incorrectly rely on allocated memory
              being initialized to zero, or reuse values in memory that has
              already been freed.
```

 * データをセキュアにクリアするための実装とは違う?
   * [jemalloc](http://linux.die.net/man/3/jemalloc) でも `ln -sfv 'junk:true' /etc/malloc.conf` とすることで同様の機能が使える
   * http://gihyo.jp/admin/clip/01/fdt/201204/26
 * mallopt を使わずとも 環境変数 MALLOC_PERTURB_=\<int\> でもセットできる

### 実装 (glibc)

perturb_byte に値がセットされているか否かで挙動が変わる. mallopt(3) に M_PERTURB 指定した際にセットされる

```c
int mALLOPt(param_number, value) int param_number; int value;
#endif
{
//
  case M_PERTURB:
    perturb_byte = value;
    break;
```

下記のような実装になっている

 * memset で埋める
 * malloc/free とで埋める文字列が異なる
   * 同じ文字で埋めたらどっちが malloc で どっちが free か分からんもんね

```c
/* ------------------ Testing support ----------------------------------*/

static int perturb_byte;

#define alloc_perturb(p, n) memset (p, (perturb_byte ^ 0xff) & 0xff, n)
#define free_perturb(p, n) memset (p, perturb_byte & 0xff, n)
```

### 検証コード

```c
#if 0
#!/bin/bash
CFLAGS="-O2 -std=gnu99 -W -Wall -fPIE -D_FORTIFY_SOURCE=2"
o=`basename $0`
o=".${o%.*}"
gcc ${CFLAGS} -o $o $0 && ./$o $*; exit
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <malloc.h>

int main()
{
    /* 小さくすると free した後も参照できないな */
	size_t size = 128;
	char *p = malloc(size+1);
	p[size] = '\0';
	printf("%s\n", p);

	/* fill heap with ABCDEFG ... */
	for (size_t i = 0; i < size; i++)
		p[i] = ( i % 26 ) + 65;

	printf("%s\n", p);
	p[size] = '\0';

	free(p);

	/* free しても参照できる */
	printf("%s\n", p);
	exit(0);
}
```

free したデータを参照できる

```
[vagrant@vagrant-centos65 vagrant]$ ./.heap 

ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWX
ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWX
```

alloc したデータが ? (255-65) と free したデータが AAA ... で上書きされている

```
[vagrant@vagrant-centos65 vagrant]$ MALLOC_PERTURB_=65 ./.heap 
????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????
ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWX
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAq
```

jemalloc でも同様の事ができる

```
$ sudo ln -sfv 'junk:true' /etc/malloc.conf
`/etc/malloc.conf' -> `junk:true'

$ jemalloc.sh /vagrant/.heap 
????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????
ABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOPQRSTUVWXYZABCDEFGHIJKLMNOP
ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
``` 

## M_MMAP_THRESHOLD オプション

 * 閾値を超えたら sbrk(2) ではなく mmap(2) でアロケート
 * ヒープは終端の領域を free しないと縮める事ができないけど、 mmap だと関係無い
 * disadvantages
   * free してチャンクが全部解放された際に munmap(2) になりカーネルが都度 zero 初期化しないといけない
   * free list で再利用されない
   * free したポインタを参照したら当然 SIGSEGV 出す

```
       M_MMAP_THRESHOLD
              For allocations greater than or equal to the limit specified
              (in bytes) by M_MMAP_THRESHOLD that can't be satisfied from
              the free list, the memory-allocation functions employ mmap(2)
              instead of increasing the program break using sbrk(2).

              Allocating memory using mmap(2) has the significant advantage
              that the allocated memory blocks can always be independently
              released back to the system.  (By contrast, the heap can be
              trimmed only if memory is freed at the top end.)  On the other
              hand, there are some disadvantages to the use of mmap(2):
              deallocated space is not placed on the free list for reuse by
              later allocations; memory may be wasted because mmap(2)
              allocations must be page-aligned; and the kernel must perform
              the expensive task of zeroing out memory allocated via
              mmap(2).  Balancing these factors leads to a default setting
              of 128*1024 for the M_MMAP_THRESHOLD parameter.

              The lower limit for this parameter is 0.  The upper limit is
              DEFAULT_MMAP_THRESHOLD_MAX: 512*1024 on 32-bit systems or
              4*1024*1024*sizeof(long) on 64-bit systems.

              Note: Nowadays, glibc uses a dynamic mmap threshold by
              default.  The initial value of the threshold is 128*1024, but
              when blocks larger than the current threshold and less than or
              equal to DEFAULT_MMAP_THRESHOLD_MAX are freed, the threshold
              is adjusted upward to the size of the freed block.  When
              dynamic mmap thresholding is in effect, the threshold for
              trimming the heap is also dynamically adjusted to be twice the
              dynamic mmap threshold.  Dynamic adjustment of the mmap
              threshold is disabled if any of the M_TRIM_THRESHOLD,
              M_TOP_PAD, M_MMAP_THRESHOLD, or M_MMAP_MAX parameters is set.
```

#### 検証コード

 * malloc(128KB) して free したポインタを printf で参照する
   * => SIGSEGV を起こす はず
 * `MALLOC_MMAP_THRESHOLD_=204800 bash mmap.c` にして上限を引き上げるとSIGSEGVしない 

```c
#if 0
#!/bin/bash
CFLAGS="-O2 -std=gnu99 -g -W -Wall -fPIE -D_FORTIFY_SOURCE=2"
o=`basename $0`
o=".${o%.*}"
gcc ${CFLAGS} -o $o $0 && ./$o $*; exit
#endif

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <malloc.h>

int main()
{
	size_t size = 128 * 1024;
	char *p = malloc(size);
	if (!p) {
		perror("malloc failed");
		exit(1);
	}

    /* print heap address */
	printf("@%p\n", p);

	/* fIll a buffer with ABCDEF .... */
	for (int i = 0; i < size; i++)
		p[i] = ( i % 26 ) + 65;

	/* munmap(2) */
	free(p);

	/* SIGSEGV */
	printf("%s\n", p);
	exit(0);
}
```

#### strace して mmap(2), brk(2) の様子を確かめる

 * 128 * 1024 ちょうどのサイズを mmap するのではない
   * 135168 = 33 * 4096 にアラインされている
 * free(3) の際に munmap している

```
mmap(NULL, 135168, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4eaf8c1000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f4eaf8e9000

// ヒープのアドレス
write(1, "@0x7f4eaf8c1010\n", 16@0x7f4eaf8c1010
)       = 16
munmap(0x7f4eaf8c1000, 135168)          = 0
--- SIGSEGV (Segmentation fault) @ 0 (0) ---
+++ killed by SIGSEGV +++
```

#### `MALLOC_MMAP_THRESHOLD_=204800` に引き上げた場合

 * mmap じゃなくて brk
 * サイズが 0x2486000 - 0x2445000 = 0x41000 = 266240 バイト = 4096 * 65 にアラインされている

```
brk(0)                                  = 0x2445000
brk(0x2486000)                          = 0x2486000
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 1), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f2d16482000
write(1, "@0x2445010\n", 11@0x2445010
)            = 11
brk(0x2466000)                          = 0x2466000
```