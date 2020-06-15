# 虚拟化
虚拟化技术允许在一台物理机上创建多台虚拟机器。这些虚拟机器本身并不拥有物理资源，但它们却可以像普通的机器一样完成任务，每台虚拟器都不知道其它虚拟机的存在。**Hypervisor**是位于底层物理资源和虚拟环境之间的一个管理层，它创建彼此相互隔离的虚拟环境并为虚拟环境分配物理资源。Hypervisor又称**虚拟机监视程序(Virtual Machine Monitor, VMM)**。安装有hypervisor的物理机被称为**宿主机(host machine)**，而被hypervisor创建并管理的虚拟资源就是**虚拟机(virtual machine)**或**客户机(guest machine)**。

## 虚拟化的类型
根据虚拟机隔离情况的不同，有三种虚拟化方式：
1. **全虚拟化(full virtualization)**。
2. **半虚拟化(paravirtualization)**。
3. **操作系统虚拟化(operating system virtualization)**。

### 全虚拟化
在全虚拟化的虚拟平台中，Guest OS并不知道自己是一台虚拟机，它会认为自己是运行在物理硬件设备上的Host OS。因为hypervisor将操作系统管理的所有硬件设备逻辑抽象为虚拟设备，然后交给Guest OS使用。实际上是Hypervisor为Guest OS制造了一种假象，让Guest OS认为底层硬件平台归自己所有。

全虚拟化为每台虚拟机提供了完全隔离的虚拟环境，大多数操作系统都直接在虚拟机中安装，而不需要任何修改。

### 半虚拟化
半虚拟化需要在对Guest OS的内核代码进行一定的修改之后，才能使Guest OS在半虚拟化的hypervisor中运行。半虚拟化不需要虚拟机陷入特权指令，因此对系统的侵入更小。Guest OS直到hypervisor的存在，并通过**hypercall**直接与hypervisor交流。

### 操作系统虚拟化
操作系统级别的虚拟化是操作系统的一个特性，内核知道多个用户空间实例的存在。我们称这种虚拟化为**容器化(containerzation)**，而将这些用户空间的实例称为**容器(container)**。程序可以在容器内运行，但它们只能访问容器内的内容和使用分配给容器的设备。容器认为自己拥有所有可用资源，实际上它们只能访问分配给容器的那些资源。

## Hypervisors
虚拟机需要表现得与真实的机器一样，比如：我们既能像启动真机一样启动它，也能在它上面按照任意的操作系统……

hypervisor的任务就是高效实现这种假象，它需要在以下三个方面表现良好：
1. **安全性(safety)**：hypervisor应该具有虚拟化资源的完全控制权。
2. **保真性(fidelity)**：虚拟机上的程序应该同其在物理机上运行的表现一致。
3. **高效性(efficiency)**：虚拟机中运行的绝大部分代码都不应该受到VMM的干涉。

毫无疑问，在**解释器(interpreter)**中逐条考查并准确执行每一条指令是安全的。有些指令可以直接执行，而其它不能完全执行的指令则需要由解释器进行模拟(使VMM上运行的操作系统认为自己正确的执行了指令)。

在早期的x86体系结构上，虚拟化一直都是一个问题。每个包含内核态和用户态的CPU都有一个特殊的指令集，其中的指令在内核态与用户态下的执行行为不同，这些指令包括I/O操作和修改MMU设置的指令，称为**敏感指令(sensitive instruction)**。还有一个指令集，其中的指令在用户态执行时会导致陷入，称为**特权指令(privileged instruction)**。Popek和Goldberg指出机器可虚拟化的一个必要条件是敏感指令为特权指令的子集，即：若要在用户态做不应该在用户态做的事，硬件必须陷入。但Intel 386不具备这一特性：很多Intel 386敏感指令在用户态执行时要么被忽略，要么具有不同的行为。因此，那时的386及其后继不能被虚拟化，也不能直接支持hypervisor。除了敏感指令在用户态不能陷入以外，386还存在可以在用户态读取敏感状态而不造成陷入的指令。例如：在2005年前的x86处理器上，程序可以通过读取代码段选择符判断自身时运行在内核态还是用户态，若虚拟机中的操作系统这么做并发现自己运行在用户态，就会做出错误的决策。

从2005年起，Intel和AMD开始在CPU中引入虚拟化支持，使这个问题最终得到解决。这项技术在Intel CPU中被称作**VT(Virtualization Technology)**，而在AMD CPU中称作**SVN(Secure Virtual Machine)**。它们的基本思想使创建可运行虚拟机的容器，客户操作系统在容器中启动并持续运行，直到发生异常并陷入hypervisor。这就使得在x86平台实现经典的**陷入模拟(trap-and-emulate)**成为可能。

Hypervisor创建并管理虚拟环境，分为两种：
1. Type 1 hypervisors (native/type metal hypervisors).
2. Type 2 hypervisors (hosted hypervisors).

[虚拟机](./VirtualMachines.md)一章详细介绍了两类hypervisor。

## 参考资料
1. ANDREW S. TANENBAUM, HERBERT BOS. <i>Modern Operating Systems, 4th Edition</i>. Pearson, 2015.