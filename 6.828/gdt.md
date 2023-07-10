## 为什么要有保护模式

OS负责整个计算机的硬件和软件管理，可以做任何事。但是用户程序应该收到限制，以免在其运行时有意或无意地破坏了OS或其他的用户程序。实模式下用户程序可以访问或修改任意内存，所以保护模式非常重要。

## 全局描述符表GDT

内存分为不同的段使得程序可以在内存中浮动但准确地执行，指令使用的段内的偏移地址。保护模式下仍然使用段地址和段内地址。但是每个段都会有一个8个字节的段描述符（segment descriptor）记录与一个段有关的信息。段描述符存放在内存的一片连续空间里，叫作段描述符表。

最重要的就是全局描述符表GDT（global descriptor table），为整个系统服务，在进入保护模式之前就建立起来。GDT可位于内存任何地方，但在实模式时只能访问1MB的内存，所以GDT地址一般小于1MB，进入保护模式后可以调整它的地址。

为了跟踪GTD，处理器内部有一个48位全局描述符表寄存器GDTR，有2部分。0-15位是全局描述符表边界，其实就就是描述符表的大小，最大为64KB，所以最多可以有`64KB / 8B = 8096`个描述符；16-47位是全局描述符表基地址，刚好表示4GB。

描述符的格式如下图：

![捕获](http://hayami.ml:8090/upload/2021/09/%E6%8D%95%E8%8E%B7-1a18b11c9db74209979e2db0795e982c.JPG)

```c
// Segment Descriptors
struct Segdesc {
	unsigned sd_lim_15_0 : 16;  // Low bits of segment limit
	unsigned sd_base_15_0 : 16; // Low bits of segment base address
	unsigned sd_base_23_16 : 8; // Middle bits of segment base address
	unsigned sd_type : 4;       // Segment type (see STS_ constants)
	unsigned sd_s : 1;          // 0 = system, 1 = application
	unsigned sd_dpl : 2;        // Descriptor Privilege Level
	unsigned sd_p : 1;          // Present
	unsigned sd_lim_19_16 : 4;  // High bits of segment limit
	unsigned sd_avl : 1;        // Unused (available for software use)
	unsigned sd_rsv1 : 1;       // Reserved
	unsigned sd_db : 1;         // 0 = 16-bit segment, 1 = 32-bit segment
	unsigned sd_g : 1;          // Granularity: limit scaled by 4K when set
	unsigned sd_base_31_24 : 8; // High bits of segment base address
};
```



## 内存访问

在保护模式下，段寄存器里存的段选择子，传到段选择器里的是段描述符在段描述表中的索引号。每个段选择器都有相应的描述符高速缓存器，存放段的基地址，限界和段属性。

段选择子的结构如下图：

![](http://hayami.ml:8090/upload/2021/09/%E6%8D%95%E8%8E%B7-4a306c94ab384228a9b37653869525c3.JPG)

将选择子中的描述符索引 * 8 + GDT的基地址就得到段描述符的地址，然后将这个段描述符加载到段描述符高速缓存寄存器。

![段选择器和段描述符高速缓存器的加载过程](http://hayami.ml:8090/upload/2021/09/%E6%AE%B5%E9%80%89%E6%8B%A9%E5%99%A8%E5%92%8C%E6%AE%B5%E6%8F%8F%E8%BF%B0%E7%AC%A6%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98%E5%99%A8%E7%9A%84%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B-1882e4abc74a4a6c9bb1c0a5d9296901.png)

在内存的数据是，根据DS（数据段寄存器）的高速缓存器里的段基地址+段内偏移地址就得到真正的地址。

![保护模式下的内存访问示意图](http://hayami.ml:8090/upload/2021/09/%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F%E4%B8%8B%E7%9A%84%E5%86%85%E5%AD%98%E8%AE%BF%E9%97%AE%E7%A4%BA%E6%84%8F%E5%9B%BE-738c727f6db644c2a7e71ab9edd0c1c8.png)

在取指令时，根据CS（代码段寄存器）的高速缓存器里的段基地址+EIP就得到指令的地址。

![保护模式下处理器取指令的过程](http://hayami.ml:8090/upload/2021/09/%E4%BF%9D%E6%8A%A4%E6%A8%A1%E5%BC%8F%E4%B8%8B%E5%A4%84%E7%90%86%E5%99%A8%E5%8F%96%E6%8C%87%E4%BB%A4%E7%9A%84%E8%BF%87%E7%A8%8B-06c6cf9cc36346bba1ff7f12ae1249dd.png)