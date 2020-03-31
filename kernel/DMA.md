## DMA_ZONE的大小是16M?
在32位的x86计算机的条件下是成立的，但是其它的绝大多数情况下不成立.
DMA_ZONE产生的历史原因？
DMA可以直接在内存和外设之间进行数据搬移，对于内存的存取来说，它和CPU一样，是一个访问的master，可以直接访问。
DMA_ZONE产生的本质原因是：不一定所有的DMA都可以访问到所有的内存，本质由硬件的设计限制。

CPU <-------> CACHE <----------> MEMORY
			 						 <-----------||
									DMA		 DMA
									device		device

在32位X86计算机的条件下，ISA实际上只可以访问16M以下的内存。那么ISA上面假设有个网卡，要用DMA，超过16M的内存，它根本就访问不到。所以LINUX内核直接把16M以下的内存单独管理。如果ISA的驱动要申请DMA buffer，GFP_DMA标志来表示你想从这个区域申请内存，可以保证申请到的内存是可以访问的。

DMA_ZONE的大小以及是否存在，都取决于实际的硬件是什么。比如有如下芯片，里面有五个DMA，A，B，C都可以访问所有的内存，D只能访问32M，而E只能访问64M，Linux设计者会直接把DMA_ZONE设为32M。保证申请到的内存都可以被访问。
现在绝大多数的Soc都可以保证DMA访问所有内存。DMA_ZONE可以不必存在。

## DMA_ZONE的内存只能DMA使用吗？
DMA_ZONE的内存做什么都可以，DMA_ZONE的作用是让有缺陷的DMA对应的外设驱动可以申请DMA buffer的时候从这个区域申请而已，但非专属。

## dma_alloc_coherent()申请的内存来自DMA_ZONE?
dma_alloc_coherent()申请的内存本质上取决于对应的DMA硬件。

~~~c
static void *__dma_alloc(struct device *dev, size_t size, dma_addr_t *handle, gfp_t gfp, pgprot_t prot, bool is_coherent, const void *caller)
{
	u64 mask = get_coherent_dma_mask(dev);
	...
	if(mask < 0xffffffffULL)
		gfp |= GFP_DMA;
		...
}
~~~
对于PrimaII而言，绝大多数的外设的dma_coherent_mask都设置为0xfffffffffULL(4G内存全覆盖)，但是SD那个则设置为256M-1对应的数字。这样SD驱动调用*dma_alloc_coherent()*的时候，GFP_DMA标志被设置。但是，其它的外设，mask覆盖了整个4G，调用*dma_alloc_coherent()*获得的内存就不一定需要来自DMA_ZONE.

## dma_alloc_coherent()申请的内存是非cache的吗？
dma_alloc_coherent() is a wrapper around a device-specific allocator, based on the dma_map_ops implementation. The default allocator from arm_dma_ops gives you uncached, buffered memory. It's expected that the driver uses a barrier (which is implied by readl/writel but not __raw_readl/__raw_writel or readl_relaxed/writel_relaxed) to ensure the write buffers are flushed.

If the machine sets arm_coherent_dma_ops rather than arm_dma_ops, the memory will be cacheable, as it's assumed that the hardware is set up for cache-coherent DMA.

## dma_alloc_coherent()申请的内存一定是物理连续的吗？
绝大多数SoC目前都支持和使用CMA技术，并且多数情况下，DMA cohernet APIs以CMA区域为申请的后端，这个时候，dma_alloc_coherent()本质上用__alloc\_from\_contiguous()从CMA区域获取内存，申请来的内存显然是物理连续的。
但是，如果IOMMU存在的话，DMA完全可以访问非连续的内存，并且把物理上不连续的内存，用IOMMU进行重新映射为I/O virtual address。

## 可以直接在进程的虚拟地址空间进行DMA操作吗？
在支持SVA(shared virtual addressing)的场景下，外设可以和CPU共享相同的虚拟地址，这样外设就可以直接共享进程的地址空见。
Shared virtual addressing for the IOMMU https://lwn.net/Articles/747230
