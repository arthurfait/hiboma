
## /proc を open(2) するプロセスをトレース

```c
// [vagrant@vagrant-centos65 ~]$ sudo stap -L 'kernel.function("sys_open")'
// kernel.function("sys_open@fs/open.c:917") $filename:char const* $flags:int $mode:int $ret:long int
probe kernel.function("sys_open").call
{
    try { 
        path = user_string($filename)
        if (path =~ "^/proc") { 
          printf ("[%d] %s -> %s\n", pid(), execname(), path)
      }
    } catch (e) {
          /* just ignore */
          /* https://www.sourceware.org/ml/systemtap/2013-q3/msg00068.html */
    }
}
```

## probe ポイント多過ぎてカーネルごと死ぬ例

```c
probe kernel.function("*").call
{
 if (pid() == target())
     printf ("%s -> %s\n", thread_indent(1), probefunc())
}

probe kernel.function("*").call
{
 if (pid() == target())
     printf ("%s <- %s\n", thread_indent(-1), probefunc())
}
```

次くらいに絞り込むと動いてくれる

```c
/* sudo foo.tap -c /bin/ls */

probe kernel.function("*@fs/*.c").call
{
 if (pid() == target())
     printf ("%s -> %s\n", thread_indent(1), probefunc())
}

probe kernel.function("*@fs/*.c").call
{
 if (pid() == target())
     printf ("%s <- %s\n", thread_indent(-1), probefunc())
}
```

出力例

```
     0 ls(2274): -> sys_getdents
     3 ls(2274):  -> fget
     4 ls(2274):  -> sys_getdents
     6 ls(2274):  -> vfs_readdir
     8 ls(2274):   -> touch_atime
     9 ls(2274):   -> vfs_readdir
    10 ls(2274):  -> sys_getdents
    12 ls(2274):  -> fput
    13 ls(2274):  -> sys_getdents
    14 ls(2274): -> tracesys
     0 ls(2274): -> path_put
     2 ls(2274):  -> dput
     3 ls(2274):  -> path_put
     4 ls(2274): -> __audit_syscall_exit
     0 ls(2274): -> sys_close
     1 ls(2274):  -> filp_close
     3 ls(2274):   -> dnotify_flush
     6 ls(2274):    -> fsnotify_find_mark_entry
     7 ls(2274):    -> dnotify_flush
     8 ls(2274):   -> filp_close
    10 ls(2274):   -> locks_remove_posix
    11 ls(2274):   -> filp_close
    13 ls(2274):   -> fput
    15 ls(2274):    -> __fput
    17 ls(2274):     -> inotify_inode_queue_event
    18 ls(2274):     -> __fput
    20 ls(2274):     -> __fsnotify_parent
    22 ls(2274):     -> __fput
    23 ls(2274):     -> inotify_dentry_parent_queue_event
    25 ls(2274):     -> __fput
    26 ls(2274):     -> fsnotify
    28 ls(2274):     -> __fput
    29 ls(2274):     -> locks_remove_flock
    31 ls(2274):     -> __fput
    33 ls(2274):     -> file_kill
    34 ls(2274):     -> __fput
    36 ls(2274):     -> dput
    37 ls(2274):     -> __fput
    39 ls(2274):     -> mntput_no_expire
    40 ls(2274):     -> __fput
    41 ls(2274):    -> fput
```

## main で hello

```
sudo stap -e 'probe process("/bin/ls").function("main") { printf("hello\n"); }'
```

## プロセスの関数呼び出しをトレースする

```c
probe process("/bin/ls").function("*").call
{
    printf ("%s -> %s\n", thread_indent(1), probefunc())
}

probe process("/bin/ls").function("*").return
{
    thread_indent(-1)
}
```

出力例

```
     0 ls(3495): -> main
    17 ls(3495):  -> set_program_name
    98 ls(3495):  -> human_options
   115 ls(3495):  -> clone_quoting_options
   123 ls(3495):   -> xmemdup
   129 ls(3495):    -> xmalloc
   143 ls(3495):  -> get_quoting_style
   151 ls(3495):  -> clone_quoting_options
   156 ls(3495):   -> xmemdup
   161 ls(3495):    -> xmalloc
   172 ls(3495):  -> set_char_quoting
   180 ls(3495):  -> xmalloc
   190 ls(3495):  -> clear_files
   198 ls(3495):  -> gobble_file
   207 ls(3495):   -> xstrdup
   213 ls(3495):    -> xmemdup
   227 ls(3495):     -> xmalloc
   235 ls(3495):  -> sort_files
   239 ls(3495):   -> xmalloc
   244 ls(3495):   -> mpsort
   248 ls(3495):    -> mpsort_with_tmp
   256 ls(3495):  -> extract_dirs_from_files
   260 ls(3495):   -> queue_directory
   263 ls(3495):    -> xmalloc
   268 ls(3495):    -> xstrdup
   271 ls(3495):     -> xmemdup
   274 ls(3495):      -> xmalloc
   282 ls(3495):   -> free_ent
   307 ls(3495):  -> print_dir
   320 ls(3495):   -> clear_files   
   349 ls(3495):   -> gobble_file   
   354 ls(3495):    -> xstrdup
   359 ls(3495):     -> xmemdup
   364 ls(3495):      -> xmalloc
   376 ls(3495):   -> gobble_file   
   381 ls(3495):    -> xstrdup
   385 ls(3495):     -> xmemdup
   390 ls(3495):      -> xmalloc
```

## unix_listen の sk_max_ack_backlog を出してみる

`-L` オプションで参照できる変数名を確認する

```
[vagrant@vagrant-centos65 ~]$ sudo stap -L 'kernel.function("unix_listen")'
kernel.function("unix_listen@net/unix/af_unix.c:467") $sock:struct socket* $backlog:int $old_pid:struct pid* $old_cred:struct cred const*
```

試行錯誤して下記のような stap スクリプトを作った

```c
probe kernel.function("unix_listen").call
{
  printf ("[%s] %d %ld\n", probefunc(), $backlog, $sock->sk->sk_max_ack_backlog)
}

probe kernel.function("unix_listen").return
{
  printf ("[%s] %d %ld\n", probefunc(), $backlog, $sock->sk->sk_max_ack_backlog)
}
```

UNIXドメインソケットのサーバは nc で簡易に立てる

```
nc -l -U /tmp/socket
```

サーバをたてると sk_max_ack_backlog のサイズが確認できる!

```
[unix_listen] 5 10
[sys_listen] 5 10

# sudo sysctl -w net.unix.max_dgram_qlen=100 して引き上げた場合

[unix_listen] 5 100
[sys_listen] 5 100
```

## 参考リンク
 
 * http://d.hatena.ne.jp/mmitou/20120721/1342879187


## probe XFS function 

`module("xfs")` 
 
```c
probe module("xfs").function("xfs_iget") { 
     printf ("%s %s %s\n",  ctime(gettimeofday_s()), probefunc(), execname())
     print_backtrace();
}
```

## sk_ack_backlog, sk_max_ack_backlog 

```
global syn_qlen_stats
global acc_qlen_stats
global max_syn_qlen
global max_acc_qlen

probe kernel.function("tcp_v4_conn_request") {
    tcphdr = __get_skb_tcphdr($skb);
    dport = __tcp_skb_dport(tcphdr);
    if (dport != $1) next;
    // First time: compute maximum queue lengths
    if (max_syn_qlen == 0) {
      max_qlen_log = @cast($sk,
         "struct inet_connection_sock")->icsk_accept_queue->listen_opt->max_qlen_log;
      max_syn_qlen = (1 << max_qlen_log);
    }
    if (max_acc_qlen == 0) {
      max_acc_qlen = $sk->sk_max_ack_backlog;
    }
    syn_qlen = @cast($sk, "struct inet_connection_sock")->icsk_accept_queue->listen_opt->qlen;
    syn_qlen_stats <<< syn_qlen;
    acc_qlen_stats <<< $sk->sk_ack_backlog;
}

probe timer.ms(1000) {
    if (max_syn_qlen == 0) {
      printf("No new connection on port %d, yet.\n", $1);
      next;
    }
    ansi_clear_screen();
    ansi_set_color2(30, 46);
    printf(" ♦ Syn queue \n");
    ansi_reset_color();
    print(@hist_log(syn_qlen_stats))
    printf(" — min:%d avg:%d max:%d count:%d\n",
                     @min(syn_qlen_stats),
                     @avg(syn_qlen_stats),
                     @max(syn_qlen_stats),
                     @count(syn_qlen_stats));
    printf(" — allowed maximum: %d\n\n", max_syn_qlen);
    ansi_set_color2(30, 46);
    printf(" ♦ Accept queue \n");
    ansi_reset_color();
    print(@hist_log(acc_qlen_stats))
    printf(" — min:%d avg:%d max:%d count:%d\n",
                     @min(acc_qlen_stats),
                     @avg(acc_qlen_stats),
                     @max(acc_qlen_stats),
                     @count(acc_qlen_stats));
    printf(" — allowed maximum: %d\n\n", max_acc_qlen);
}
```

refs https://github.com/vincentbernat/systemtap-cookbook/blob/master/tcp
