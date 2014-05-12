# ץס¿ñµͳÄ´æ½ø­Ît½ÓÚâ£º[Kernel Memory Allocation](http://sourceware.org/systemtap/wiki/WSKmemCacheAlloc) 

### Problem

¸ùoc/slabinfoÖÌ¹©µÄÅ¢£¬ÈºÎªµÀÇ©½ø·èµĳÔµͳÄ´棿ÏÃÕ¸ö¾½«Ì¹©ºܶàÓµÄÅ¢¡£

### Script

 ```
#This script displays the number of given slab allocations and the backtraces leading up to it. 

 global slab = @1
 global stats, stacks
 probe kernel.function("kmem_cache_alloc") {
           if (kernel_string($cachep->name) == slab) {
                             stats[execname()] <<< 1
                                               stacks[execname(),kernel_string($cachep->name),backtrace()] <<< 1
                                                       }   
 }
# Exit after 10 seconds
probe timer.ms(10000) { exit () }
probe end {
          printf("Number of %s slab allocations by process\n", slab)
                    foreach ([exec] in stats) {
                                      printf("%s:\t%d\n",exec,@count(stats[exec]))
                                                }   
                  printf("\nBacktrace of processes when allocating\n")
                            foreach ([proc,cache,bt] in stacks) {
                                              printf("Exec: %s Name: %s  Count: %d\n",proc,cache,@count(stacks[proc,cache,bt]))
                                                                print_stack(bt)
                                                                                printf("\n-------------------------------------------------------\n\n")
                                                                                        }
}
```

###  Output

```
# stap slab.stp size-32
Number of size-32 slab allocations by process
crond:  2
ps:     2
upsd:   1
snmpd:  3
httpd:  1
xfce4-panel:    1
xfce4-cpugraph-:        11
topfew: 4
Backtrace of processes when allocating
Exec: crond Name: size-32  Count: 2
0xc045d717 : kmem_cache_alloc+0x2/0x4f []
0xf890da44 : ext3_readdir+0x9c/0x5f4 [ext3]
0xc04707d4 : vfs_readdir+0x66/0x90 []
0xc0470a2b : sys_getdents+0x5f/0x9c []
0xc0402d9b : syscall_call+0x7/0xb []
-------------------------------------------------------
[...]
Exec: httpd Name: size-32  Count: 1
0xc045d717 : kmem_cache_alloc+0x2/0x4f []
0xc045da31 : cache_alloc_refill+0x2cd/0x409 []
0xc045dbd9 : __kmalloc+0x6c/0x76 []
0xc05a19bb : pskb_expand_head+0x43/0x110 []
0xc05a57a0 : skb_checksum_help+0x46/0xbc []
0xf8a3d339 : ip_nat_fn+0x45/0x192 [iptable_nat]
0xf8a3d6a6 : ip_nat_local_fn+0x3e/0xb0 [iptable_nat]
0xc05baab8 : nf_iterate+0x38/0x6a []
0xc05babf0 : nf_hook_slow+0x4d/0xb5 []
0xc05c3ff1 : ip_queue_xmit+0x389/0x3ce []
0xc05d1783 : tcp_transmit_skb+0x5f8/0x626 []
0xc05d308f : __tcp_push_pending_frames+0x6bc/0x783 []
0xc05d3e7e : tcp_send_fin+0x13b/0x141 []
0xc05dfeac : inet_shutdown+0x80/0xbb []
0x0000000c
[...]
```
