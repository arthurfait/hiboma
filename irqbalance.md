# irqbalance

## CentOS on VirtualBox + OS X + Macbook Pro 強制的に one shot モードになる

下記のコードが原因

```
	/* On dual core/hyperthreading shared cache systems just do a one shot setup */
	if (cache_domain_count==1)
		one_shot_mode = 1;
```

理由は下記の通り

> The purpose of irqbalance is to distribute interrupts accross cpus in an smp system such that cache-domain affinity is maximized for each irq. In other words, irqbalance tries to assign irqs to cpu cores such that each irq stands a greater chance of having its interrupt handler be in cache when the irq is asserted to the cpu. This raises a few interesting cases in which the behavior of irqbalance may be non-intuitive. Most notably, cases in which a system has only one cache domain. Nominally these systems are only single cpu environments, but can also be found in multi-core environments in which the cores share an L2 cache. In these situations irqbalance will exit immediately, since there is no work that irqbalance can do which will improve interrupt handling performance. This is normal and not cause for concern. For more information regarding irqbalance, please visit http://irqbalance.org/ 


 * **Cache Domain**
   * L2キャッシュを共有しているコア群?

CPU を cache domain に追加するコード

```c
static struct topo_obj* add_cpu_to_cache_domain(struct topo_obj *cpu,
						    cpumask_t cache_mask)
{
	GList *entry;
	struct topo_obj *cache;
	struct topo_obj *lcpu;

	entry = g_list_first(cache_domains);

	while (entry) {
		cache = entry->data;
        /* キャッシュマスクが違う = cache domain が違う */
		if (cpus_equal(cache_mask, cache->mask))
			break;
		entry = g_list_next(entry);
	}

	if (!entry) {
		cache = calloc(sizeof(struct topo_obj), 1);
		if (!cache)
			return NULL;
		cache->obj_type = OBJ_TYPE_CACHE;
		cache->mask = cache_mask;
		cache->number = cache_domain_count;
		cache->obj_type_list = &cache_domains;
		cache_domains = g_list_append(cache_domains, cache);
		cache_domain_count++;
	}
``` 

## see also

 * http://tester7.hatenablog.com/entry/2014/05/22/IvyBridge_%26%26_ESXi_%26%26_CentOS環境でirqbalanceが起動後数秒で終了する問題 