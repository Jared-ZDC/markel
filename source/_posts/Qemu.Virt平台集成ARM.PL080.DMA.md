---
title: Qemu Virt平台集成ARM PL080 DMA
date: 2024-06-04 05:34:28
---
# 背景
最近有比较多的新员工入职，新员工入职如何更快的适应工作，需要通过一些虚拟项目进行工作培训；但是如果利用现有的芯片平台进行验证，有几个缺陷：
* 当前芯片略显复杂，新员工接手理解难度较高
* 平台已经solid，并且有参考的代码以及测试用例，对于新员工而言，没有起到端到端cover一个模块的作用
* 难以预埋bug，无法通过挖掘设计bug，衡量验证效果

因此根据实际工作需求，设计了一个可以给到新员工练兵的验证项目，利用qemu平台，集成一个简单的DMA模块，并且在模块中预埋一些bug，让新员工提前感知在后续验证过程中，应该要重点关注的一些验证重点，验证难点，同时也可以观察在整个新员工项目过程中，新员工的表现，识别高潜人才；

# 验证平台开发 
## 集成开发环境
Qemu针对arm类的处理器，提供了一个单纯的开发环境，Virt平台，如下是virt平台的一些简述：
> [‘virt’ Generic Virtual Platform](https://www.qemu.org/docs/master/system/riscv/virt.html#virt-generic-virtual-platform-virt)
>The virt board is a platform which does not correspond to any real hardware; it is designed for use in virtual machines. It is the recommended board type if you simply want to run a guest such as Linux and do not care about reproducing the idiosyncrasies and limitations of a particular bit of real-world hardware.
>[Supported devices](https://www.qemu.org/docs/master/system/riscv/virt.html#supported-devices)
>The virt machine supports the following devices:
> * Up to 512 generic RV32GC/RV64GC cores, with optional extensions
> * Core Local Interruptor (CLINT)
> * Platform-Level Interrupt Controller (PLIC)
> *CFI parallel NOR flash memory
> * 1 NS16550 compatible UART
> * 1 Google Goldfish RTC
> * 1 SiFive Test device
> * 8 virtio-mmio transport devices
> * 1 generic PCIe host bridge
> * The fw_cfg device that allows a guest to obtain data from QEMU
> The hypervisor extension has been enabled for the default CPU, so virtual machines with hypervisor extension can simply be used without explicitly declaring.

所以在Virt平台中，简单集成一下virtio-mmio设备即可；

于此同时，对于验证新手项目而言，DMA设备是一个能够比较小巧又比较全面的设备，因此可以选择一个相对简单的DMA集成到环境中即可；

## DMA选型
在原生qemu 代码中，提供了如下几款DMA设备：
* bcm2835_dma
* omap_dma
* arm pl080 dma
* arm pl330 dma
* xlnx的3款dma
* ...
![qemu dma](https://github.com/Jared-ZDC/markel/assets/17999499/8631ad56-50c5-4a4d-b657-699aaee31024)


从选择的几个dma spec内容看起来，arm的pl080 dma相对简单，并且有完善的spec资料以及开源支持的驱动代码，因此在选型上个人更偏向arm PL080 DMA设备；

## 集成DMA
qemu virt集成的方式比较简单，首先看一下pl080的设备描述：
```c
static Property pl080_properties[] = {
    DEFINE_PROP_LINK("downstream", PL080State, downstream,
                     TYPE_MEMORY_REGION, MemoryRegion *),
    DEFINE_PROP_END_OF_LIST(),
};
```
PL080的属性只有一个downstream， 类型是memory region类型，表示dma 链接系统memory的区域， 因此在qemu virt平台中集成只要对downstream进行处理即可：
```c
//virt.c
static const MemMapEntry base_memmap[] = {
    /* Space up to 0x8000000 is reserved for a boot ROM */
    [VIRT_FLASH] =              {          0, 0x08000000 },
...
    [VIRT_DMA] =                { 0x09011000, 0x00001000 },
...
    [VIRT_MEM] =                { GiB, LEGACY_RAMLIMIT_BYTES },
};

static const int a15irqmap[] = {
    [VIRT_UART] = 1,
    [VIRT_RTC] = 2,
...
    [VIRT_PLATFORM_BUS] = 112, /* ...to 112 + PLATFORM_BUS_NUM_IRQS -1 */
    [VIRT_DMA] = 212,
};
static void create_dma(const VirtMachineState *vms)
{
    int i;
    char *nodename;
    hwaddr base = vms->memmap[VIRT_PDMA].base;
    hwaddr size = vms->memmap[VIRT_PDMA].size;
    int irq = vms->irqmap[VIRT_PDMA];
    const char compat[] = "arm,pl080\0arm,primecell";
    const char irq_names[] = "intr\0interr\0inttc";
    DeviceState *dev;
    MachineState *ms = MACHINE(vms);
    SysBusDevice *busdev;

    dev = qdev_new("pl080");

    object_property_set_link(OBJECT(dev), "downstream", OBJECT(get_system_memory()), &error_fatal);

    busdev = SYS_BUS_DEVICE(dev);
    sysbus_realize_and_unref(busdev, &error_fatal);
    sysbus_mmio_map(busdev, 0, base);

    for (i = 0; i < 3; ++i) {
        sysbus_connect_irq(busdev, i, qdev_get_gpio_in(vms->gic, irq + i));
    }

    nodename = g_strdup_printf("/pl080@%" PRIx64, base);
    qemu_fdt_add_subnode(ms->fdt, nodename);
    qemu_fdt_setprop(ms->fdt, nodename, "compatible", compat, sizeof(compat));
    qemu_fdt_setprop_sized_cells(ms->fdt, nodename, "reg",
                                 2, base, 2, size);
    qemu_fdt_setprop_cells(ms->fdt, nodename, "interrupts",
                    GIC_FDT_IRQ_TYPE_SPI, irq, GIC_FDT_IRQ_FLAGS_LEVEL_HI,
                    GIC_FDT_IRQ_TYPE_SPI, irq + 1, GIC_FDT_IRQ_FLAGS_LEVEL_HI,
                    GIC_FDT_IRQ_TYPE_SPI, irq + 2, GIC_FDT_IRQ_FLAGS_LEVEL_HI);

    qemu_fdt_setprop(ms->fdt, nodename, "interrupt-names", irq_names,
                     sizeof(irq_names));

    qemu_fdt_setprop_cell(ms->fdt, nodename, "clocks", vms->clock_phandle);
    qemu_fdt_setprop_string(ms->fdt, nodename, "clock-names", "apb_pclk");

    g_free(nodename);
}

```

为了验证集成dma的寄存器访问正确性， 单独将0x64寄存器设置为魔术字寄存器，可读，可写
```patch
//dma集成读写测试用
diff --git a/qemu/qemu-8.1.4/hw/dma/pl080.c b/qemu/qemu-8.1.4/hw/dma/pl080.c
index a03dcf428..9ff191a02 100644
--- a/qemu/qemu-8.1.4/hw/dma/pl080.c
+++ b/qemu/qemu-8.1.4/hw/dma/pl080.c
@@ -221,6 +221,8 @@ static uint64_t pl080_read(void *opaque, hwaddr offset,
     uint32_t i;
     uint32_t mask;

+    qemu_log_mask(LOG_GUEST_ERROR,"pl080_read %x, %d,  size %x\n", offset, offset >> 2, size);
+
     if (offset >= 0xfe0 && offset < 0x1000) {
         if (s->nchannels == 8) {
             return pl080_id[(offset - 0xfe0) >> 2];
@@ -275,6 +277,8 @@ static uint64_t pl080_read(void *opaque, hwaddr offset,
         return s->conf;
     case 13: /* Sync */
         return s->sync;
+    case 16: /*magic word*/
+       return s->magic_words;
     default:
     bad_offset:
         qemu_log_mask(LOG_GUEST_ERROR,
@@ -289,6 +293,7 @@ static void pl080_write(void *opaque, hwaddr offset,
     PL080State *s = (PL080State *)opaque;
     int i;

+    qemu_log_mask(LOG_GUEST_ERROR,"pl080_write %x, size %x, value %x\n", offset, size, value);
     if (offset >= 0x100 && offset < 0x200) {
         i = (offset & 0xe0) >> 5;
         if (i >= s->nchannels)
@@ -338,6 +343,8 @@ static void pl080_write(void *opaque, hwaddr offset,
     case 13: /* Sync */
         s->sync = value;
         break;
+    case 16: /*magic word*/
+       s->magic_words = value;
     default:
     bad_offset:
         qemu_log_mask(LOG_GUEST_ERROR,
@@ -366,7 +373,7 @@ static void pl080_reset(DeviceState *dev)
     s->req_single = 0;
     s->req_burst = 0;
     s->running = 0;
-
+    s->magic_words = 0x55aa55aa;
     for (i = 0; i < s->nchannels; i++) {
         s->chan[i].src = 0;
         s->chan[i].dest = 0;
```

## 编译
将qemu编译完成后，使用如下命令启动系统：
```bash
build/qemu-system-aarch64 \
    -nographic \
    -M virt,virtualization=true,gic-version=3 \
    -cpu cortex-a76 \
    -smp 4 \
    -m 8G \
    -kernel ../../linux/linux-6.6/arch/arm64/boot/Image \
    -append "rootfstype=ramfs rdinit=/linuxrc console=ttyAMA0" \
    -initrd ../../busybox/busybox-1.36.1/initrd.cpio.gz
```

## 测试
### dump dma寄存器
通过dump dma的寄存器，简单看一下dma寄存器是否按照预期集成完成
```bash
[root:DMA: ~]# devmem 0x9012040
0x55AA55AA
[root:DMA: ~]# devmem 0x9012040 32 0xaa55aa55
[root:DMA: ~]# devmem 0x9012040
0xAA55AA55
```
寄存器读写功能正常

### dma搬运测试
#### dma测试驱动
简单写一个dma测试代码，搬运两个地址的数据，并进行搬运后数据检查，看看中断是否正常上报
```
