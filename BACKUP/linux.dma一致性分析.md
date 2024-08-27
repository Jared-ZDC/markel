---
title: linux dma一致性分析
date: 2024-08-27 06:37:18
tags: 总结思考
categories: 总结思考
toc: true
---
# 背景

在驱动的开发过程中，一般来说会涉及两部分的内存：

* 描述符内存，数据量小，但是会比较频繁的跟CPU进行交互 
* 数据内存，数据量大，跟CPU交互的一次较少；

在一般芯片架构设计以及软件设计上，对两者经常会有些特别不一样的处理，比如对于描述符的内存，是否需要IO去识别，并且主动推送到CPU cache中？ 又或者，对于数据内存，是否需要IO标记不冲刷系统SLC？等等，这里根据业务场景，行为模式的不同，会有一些区分；

同时，对于两块内存的申请，内核也会有特别的内存接口申请方式，一般来说，在内核驱动中，针对描述符内存，一般使用dma_alloc_coherent/dma_alloc_noncoherent接口，并且实际上，目前linux内核驱动，默认使用coherent接口，没有对描述符内存允许软件可配使用noncoherent接口；在处理数据内存的时候，一般使用dma_map_single/dma_pool的方式申请，并且可以选择由硬件维护一致性或者软件维护一致性；

<!-- more -->

## 一致性

这里针对一致性稍微做一下解释，这里说的一致性指代的是外设跟CPU看到的内存数据是否是一致的，跟cache无关； 

比如说，假设映射的内存是Cacheable的，只要CPU跟IO 在Cache中看到的数据是一致的，无论这个数据在硬件架构设计中，是如何保证看到的一致都可以；同样的，假设映射的内存是NonCacheable的，CPU跟IO外设看到的数据在DDR中都是一致的即可；

当然，这里说的更多的是硬件去维护了数据的一致性，也可以由软件去维护，比如大部分情况下，描述符内存支持软件去维护一致性；



> 最近团队看到一款芯片，似乎存在一致性的问题，这个问题比较有意思，我会单独以另外一篇文档去分析该问题，在有些架构设计的时候，需要特别注意；



---



#  代码分析

## 描述符内存

```c
//dma_alloc_coherent -> dma_alloc_attrs -> dma_direct_alloc 

void *dma_direct_alloc(struct device *dev, size_t size,
                dma_addr_t *dma_handle, gfp_t gfp, unsigned long attrs)
{
        bool remap = false, set_uncached = false;
        struct page *page;
        void *ret;
		
    	... ...

        /* we always manually zero the memory once we are done */  --->  (1)
        page = __dma_direct_alloc_pages(dev, size, gfp & ~__GFP_ZERO, true);
        if (!page)
                return NULL;

        ... ...
            

        if (remap) {
                pgprot_t prot = dma_pgprot(dev, PAGE_KERNEL, attrs);   ---> (2)

                if (force_dma_unencrypted(dev))
                        prot = pgprot_decrypted(prot);

                /* remove any dirty cache lines on the kernel alias */
                arch_dma_prep_coherent(page, size);   ---> (3)

                /* create a coherent mapping */   ---> (4)
                ret = dma_common_contiguous_remap(page, size, prot,   
                                __builtin_return_address(0));
                if (!ret)
                        goto out_free_pages;
        } else {
                ret = page_address(page);
                if (dma_set_decrypted(dev, ret, size))
                        goto out_free_pages;
        }

        memset(ret, 0, size);
		
    	... ...

        *dma_handle = phys_to_dma_direct(dev, page_to_phys(page));
        return ret;
    
    	
    	
```

这部分代码主要的过程描述如下：

* （1）通过__dma_direct_alloc_pages申请内存，并且将这部分page的内容清0，为了加快速度，内核默认情况下，都是申请的cacheable的内存，在内核启动过程中，默认将大部分的内存都通过线性映射的方式建立了映射关系；所以在该接口中，只是从软件管理的page内存池中获取对应大小的page，并不会建立页表真实的entry，这部分entry早就建立完成；

* （2）这里是获取后续需要重新remap的时候，建立的页表属性；

```c
  
  pgprot_t dma_pgprot(struct device *dev, pgprot_t prot, unsigned long attrs)
  { 
          if (dev_is_dma_coherent(dev))   ---> 如果dts传入dma-coherent 标志，则使用cacheable的属性（由外部传入）
                  return prot;
  #ifdef CONFIG_ARCH_HAS_DMA_WRITE_COMBINE
          if (attrs & DMA_ATTR_WRITE_COMBINE)
                  return pgprot_writecombine(prot);
  #endif
          return pgprot_dmacoherent(prot);  ---> 否则这里就是用noncache的属性
  }
  
  
  #define pgprot_dmacoherent(prot) \
          __pgprot_modify(prot, PTE_ATTRINDX_MASK, \
                          PTE_ATTRINDX(MT_NORMAL_NC) | PTE_PXN | PTE_UXN)
```
  
* （3）这里特别注意alias， 因为内核在初始的时候有初始化过系统的页表，也有可能当前这块空间之前有人使用过，假设是cacheable属性的，需要保证这块内存的数据已经完全刷入到PoC点，所以这里在申请到的时候，先对这块page进行flush动作

```c
void arch_dma_prep_coherent(struct page *page, size_t size)
{
        unsigned long start = (unsigned long)page_address(page);

        dcache_clean_poc(start, start + size);
}
```

* （4）dma_common_contiguous_remap开始设置具体的页表项PTE等



可以看到，大部分的驱动实际上在描述符内存的应用上，都是使用的coherent的方式，要么都是cacheable，由硬件维护一致性，要么都是noncacheable，数据直接到DDR中，这样也能保持一致性（这个应该是底线要求，也是驱动默认的使用dma_alloc_cohorent接口的原因）；

当然可能有些系统，处于某种原因，需要使用dma_alloc_noncoherent接口，这个时候就需要主动去维护一致性，在使用描述符内存的时候，在软件中，主动进行刷的动作，具体操作可以跟数据内存方式一致；



## 数据内存

一般来说数据内存一般使用dma_map_single/dma_unmap_single的方式申请/释放，并且该接口支持一致性的维护：

* 设备在map page的时候，需要将CPU这段空间的数据刷到PoC点（point of coherent），保证最新的数据已经到一致性节点处；

```c
//dma_map_single -> dma_map_page_attrs -> dma_direct_map_page

static inline dma_addr_t dma_direct_map_page(struct device *dev,
                struct page *page, unsigned long offset, size_t size,
                enum dma_data_direction dir, unsigned long attrs)
{
    
        ... ...
        
        if (!dev_is_dma_coherent(dev) && !(attrs & DMA_ATTR_SKIP_CPU_SYNC))
                arch_sync_dma_for_device(phys, size, dir);
        return dma_addr;
}

```

当然如果设备加了dma-coherent标签，表示硬件支持一致性维护，这里就不需要主动进行刷的动作

* 设备在unmap page的时候，需要CPU invalid cache，保证从DDR中获取设备更新的数据；

```c
static inline void dma_direct_unmap_page(struct device *dev, dma_addr_t addr,
                size_t size, enum dma_data_direction dir, unsigned long attrs)
{
        phys_addr_t phys = dma_to_phys(dev, addr);

        if (!(attrs & DMA_ATTR_SKIP_CPU_SYNC))
                dma_direct_sync_single_for_cpu(dev, addr, size, dir); ---> invalid cache

    	... ...
}
```



# 其它

在有些CPU架构里面，会对描述符内存或者数据内存做一些额外的处理，用来提升整体IO的性能，一般常见的几种特别的做法是：

* 支持标记描述符内存的地址，主动push到对应核的cache中，提升CPU与外设交互的性能
* 支持标记数据内存的范围，比如像在网口中，前面的64字节内存push到cache中，后面的数据不存放cache，避免冲刷其它业务cache，导致性能下降，前面64字节是网络报文头经常处理的，需要与cpu频繁交互，在cache中可以提升CPU交互的性能
* 支持数据内存的filter功能，比如特定长度的数据内存可以支持乱序，特定长度的内存需要保序，这个在PCIe 的网卡中比较常见
* ......



