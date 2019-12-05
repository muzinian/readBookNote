## 基于闪存的SSD
在硬盘驱动主宰了数十年后，一种新型持久化存储设备最近在世界范围取得了重要意义。通常称它为 __固态硬盘(solid-state storage)__，这种设备不像硬盘那样有机械或者移动部分；它们是有晶体管(transistors)构建的，很像内存和处理器。然而，不像典型的随机访问内存(例如，DRAM)，这样一个 __固态存储设备(solid-state storage device)__(也即，__SSD__)在掉电时也会保持数据，从而是数据持久化设备的理想候选。

我们要关注的技术是 __闪存(flash)__(更具体的，__NAND闪存(NAND-based flash)__)，由舛冈富士雄20世纪80年代发明。我们将看到闪存有很多独特的性质。例如，为了在它的一个chunk中写入数据(例如，一个 __闪存页(flash page)__)，你先不得不擦除更大的chunk(例如，__闪存block(flash block)__)，这就很昂贵了。另外，对一个页写入太多次会导致 __耗尽(wear out)__。这两个性质就让制造闪存SSD有了新的挑战：
>#### 症结：如何构建闪存SSD
>我们要怎样才能构建闪存SSD？我们如何处理擦除这一昂贵特性？我们要怎样构建一个可用时间很久的设备，即使重复覆写回到是设备耗尽？技术进步的比赛会停止么？或者会停止在不再让人惊喜？
### 44.1 存储单个bit
闪存芯片设计为在单个晶体管可以存储一到多个bits；晶体管束缚的电荷水平 (the level of charge trapped within the transistor is mapped to binary value)被映射到二进制值(the level of charge trapped within the transistor is mapped to binary value)。在 __单层cell(single-level cell SLC)__ 闪存，一个晶体管只存储一个bit(1或者0)；而在 __多级cell(multi-level cell MLC)__ 闪存中，两位被编码为不同的电荷水平，例如，00，01，10和11分别代表，低，稍微低，稍微高和高水平。甚至还有 __三层cell(triple-level cell TLC)__ 闪存，在每个cell中编码了三个位。总之，SLC芯片性能更高但是也更贵。

