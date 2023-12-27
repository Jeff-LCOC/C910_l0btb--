# 一、介绍l0btb整体架构

## （一）指令提取单元IFU:Instruction Fetch Unit

指令高速缓存的访问位于指令提取单元（IFU），为了更好地理解L0BTB的作用，我们将从IFU介绍起。

### 1.  IFU的作用：

IFU主要功能是从指定地址获取指令数据，并自行预测下一次读取指令的地址。有可能会预测错误，所以会产生一定的代价，IFU的设计目标之一是减少预测的准确率，并同时降低错误预测代价。
     

### 2. IFU的流水级：

取指单元一共分为三个流水级：Instruction Fetch(下文简称为IF)、Instruction Pack(下文简称为IP)、Instruction Buffer(下文简称为IB)。

   IF：访问L1指令缓存获取指令，一周期最多可以取出128bit指令。

   IP：预处理指令(Pack)，最多可以预处理8条指令，在发生缓存miss时进行回填（refill）。

   IB：将指令缓存到指令Buffer中，然后向后续流水线发送指令。

   源码中的pcgen模块主要功能是产生PC，产生PC后即向指令高速缓存发出访问请求。


### 3.  IFU的结构：

-  L0_BTB从pcgen中读入信号：在IP级也存在一个更大的预译码模块，在从IF级传递到IP级的拆解指令包后的预译码基础上，除了条件分支和绝对分支指令外IP级的译码模块可以译码出间接分支指令、call指令、return指令、auipc指令、load/store指令、跳转指令的偏移量(offset)和一些向量指令等等。译码出的间接分支指令可用于间接分支预测，call/return指令可以用于堆栈操作以及L0 BTB的更新，跳转指令的偏移量可以用于判断分支指令的跳转目标是否预测正确从而纠正结果。

- ![img](https://uipougzhas1.feishu.cn/space/api/box/stream/download/asynccode/?code=ZDA5OTMxOTlhNGY1NWVjNGY1ZDQwYWI5ZDdiNjBiMzZfOFBTWnRXYTcxQjlhREduMkt1bzNNdE8wOHhmQWI3SDBfVG9rZW46T25OcGIxZTlqb3FRa1V4NnJxUWNvNDBGbjJiXzE2ODU2MjcxNjU6MTY4NTYzMDc2NV9WNA)

-  这张图是整个ifu的结构图，在此过程中，对l0btb的更新信号在addrgen和ibdp中给出。

- 具体来说，可以分成两个过程，首先，给出更新队列信息。

- ```verilog
  assign l0_btb_update_entry[15:0] =  {16{addrgen_l0_btb_update_vld}}          & addrgen_l0_btb_update_entry[15:0] | {16{l0_btb_update_vld_for_gateclk}}      & ibdp_l0_btb_update_entry[15:0];
  ```

l0_btb_update_entry是更新队列信息，它由两部分组成，一是来自addrgen的更新信息，二是来自ibdp的更新信息，这两者按位或形成更新队列信息。

l0_btb_update_entry的每一位对应了后续实例化中的每一个entry的update信号，如下图所示。

L0 BTB有两种hit，一种是命中非RAS表项，一种是命中RAS表项，但是，若命中的是非RAS表项，除非表项的CNT为1，否则即使命中L0 BTB仍不能产生重定向。统计L0 BTB的命中率我认为应该统计命中且对前端产生重定向行为的次数除以有分支的指令包访问L0 BTB的次数。有一种方法可供参考，可以在IB级写一个信号A = l0_btb_ras_hit（该信号在ibdp中） && ipctrl_ibctrl_l0_btb_hit && !ibctrl_ibdp_chgflw，通过流水线寄存器打一拍流到addrgen级，将寄存器的输出信号记为B，assign C= B && !addrgen_pcgen_pcload，将信号C使能的次数作为分子，分母可以用类似的方法，找到某一些信号可以表征存在分支指令的指令包的数量，将其流水到addrgen级付给新的信号作为分母。流水到addrgen级是为了确保他们真正执行了而不是被刷掉了。l0_btb_mispred信号表示指令包中没有分支指令而命中L0 BTB并且在IF级对前端产生了重定向，或者命中L0 BTB没有命中L1 BTB并且在IF级对前端产生了重定向（由于两者是inclusive的关系，要求L0 BTB中分支一定在L1 BTB中），或者指令包中没有return指令却命中L0 BTB中RAS表项（即IB级的ras_mistaken情况），都算L0 BTB的mispredict

PC的维护是通过pcgen模块，在此模块中处理来自处理器不同流水级的重定向(change flow)以及维护IFU中不同流水级的PC，将最终判断出的PC发送到icache取出指令。icache中的predecode array中存有预译码信息，该信息将和指令信息在IF级输出。

在icache中的predecode模块主要译码出指令包中的绝对跳转指令和条件跳转指令在指令包中的位置信息：当从Icache的precode array中读出指令的预译码结果，若发现为跳转指令，会进行跳转路预测，或者当发生异常或者分支预测失败需要change flow时，或其他需要reissue的情况时，也会进行跳转路预测，通过PC访问BTB获取路预测信息。这些信息可用于在IP级选择L0BTB和BTB中的跳转目标等信号的处理，这个小的预译码模块的作用除了译码出简单的分支指令用于分支预测器，而更关键的操作在于其能够得到将指令包拆解后的信息，此时不同的32位指令和16位压缩指令的位置和清晰地得到了。

## （二）介绍l0btb整体架构

​            

### 1.l0_BTB部分在分支预测器中的作用：

分支预测器只预测分支指令是否需要跳转，但跳转地址的预测也很重要。为了降低分支代价，需要知道尚未译码的指令是不是分支，且要知道这个分支指令的跳转目标是什么。如果即知道这条这条分支指令的跳转方向，也知道这条分支指令的跳转目标，那么我们就可以将分支代价降为零。 

分支目标缓冲器，即BTB(branch target table)，会将分支指令标志(tag)以及其对应的跳转目标(target)缓存在缓冲区中，可以是绝对跳转指令，也可以是条件跳转指令，当指令PC命中缓冲区中的某一项时，会将从BTB中读出的跳转目标，将其作为下一次取指的PC。如果预测正确的话，此时分支代价就为零。即使在IP级检测到了跳转并进行PC的重定向，流水线中仍然会出现气泡。当跳转发生过于频繁时，IFU的带宽无法满足后续执行单元的需求。 为了消除流水线中的气泡，C910中的IFU采用**级联分支目标缓冲区**。L1 BTB是主要的BTB。L0 BTB一共有16个表项(entry)，且为全相联的结构。在IF阶段，如果分支指 令命中L0 BTB，则将存储的预测地址作为跳转地址，立即执行跳转，消除了流水线中的气泡。 

1. 命中过程：L0 BTB和BTB的索引信号和标志信号，即读BTB所需要的信号都在IF级给出，而L0 BTB 若命中则在IF级就可以得到跳转目标，发送给pcgen模块进行PC的重定向。BHT在IP级得到预测跳 转方向的结果后，若命中BTB，则在IP级ipctrl模块根据该结果选择BTB中对应的跳转目标，进行PC 的重定向。
1. 更新维护：L0 BTB是在IP级进行维护的，在IP级比较L0 BTB和BTB中的跳转目标是否一致，若不一 致则发生预测错误，此时将刷掉IF级的有效数据，以BTB中的跳转目标作为重定向的PC，同时产生l0_btb_mispred 信号传递到IB级用于更新L0 BTB。同样的，对L0 BTB中的return指令的维护也在 IP级，和ras模块传递到IP级的栈顶的返回地址比较，不一致则发生预测错误，而在IB级访问ras模 块才能得到正确的返回地址，同样的将IF和IP级的有效数据刷掉。

### 2.L0 BTB的结构

 即使在IP级检测到了跳转并进行PC的重定向，流水线中仍然会出现气泡。 当跳转发生过于频繁时，IFU的带宽无法满足后续执行单元的需求。 为了消除流水线中的气泡，C910中的IFU采用级联分支目标缓冲区。 L1 BTB是主要的BTB。L0 BTB一共有16个表项(entry)，且为全相联的结构。在IF阶段，如果分支指令命中L0 BTB，则将存储的预测地址作为跳转地址，立即执行跳转，消除了流水线中的气泡。

 以IF级的指令PC的[14:0]位作为访问L0 BTB的标志(tag)，由于是全相联结构，将比较L0 BTB中每一路的标志，若命中，则将从L0 BTB中读出的跳转目标(target)作为PC的重定向，从icache中取指令。

```verilog
assign entry0_rd_hit  = (l0_btb_rd_tag[14:0] == entry0_tag[14:0])  && entry0_vld && 												!pcgen_l0_btb_chgflw_mask;
assign entry1_rd_hit  = (l0_btb_rd_tag[14:0] == entry1_tag[14:0])  && entry1_vld && 												!pcgen_l0_btb_chgflw_mask;
assign entry2_rd_hit  = (l0_btb_rd_tag[14:0] == entry2_tag[14:0])  && entry2_vld && 												!pcgen_l0_btb_chgflw_mask;...
```

### 3.对分支指令跳转目标的预测

 如果当前指令PC没有命中L0 BTB，由于当前PC是同时访问BTB，此时相当于直接访问BTB获得分支跳转目标。值得注意的是，由于L0 BTB容量非常小，只有16个表项，为了使其空间利用最大化，存在于L0 BTB里的指令PC也是需要筛选的，应当筛选出那些最有可能发生跳转的指令放在其中，这样使所有L0 BTB中存放的指令PC都是有可能命中并且预测正确的，不会造成空间的浪费。根据源码可以看到，筛选出来的指令PC对应的指令类型有三种，分别为strongly taken的条件跳转指令（预测状态位为11）、绝对跳转指令、以及return指令。

```verilog
assign l0_btb_miss = (!ipdp_ipctrl_l0_btb_vld || ipdp_ipctrl_l0_btb_ras && ipdp_ipctrl_l0_btb_vld)&& ( 						branch_taken && (bht_data[1:0] == 2'b11) ||branch_ntake)   
    			&& l0_btb_ipctrl_st_wait&& ip_vld&& !ip_expt_vld;
```

 L0 BTB与BTB之间对于条件跳转和绝对跳转的缓存是包含的关系，即缓存在L0 BTB中的条件跳转指令和绝对跳转指令也一定在BTB中。发生未命中L0 BTB时，会判断该条件跳转指令或绝对跳转指令是否存在于BTB内，如果存在则发生L0 BTB缺失，若不存在会首先更新BTB，下一次该指令被取出时发生BTB命中而L0 BTB未命中时，才判断L0 BTB缺失，更新L0 BTB。

```verilog
assign l0_btb_br_miss = l0_btb_br_miss_pre&& !ibdp_btb_miss&& !l0_btb_ras_update;
```

 通过比较同一指令PC下L0 BTB的预测跳转目标和BTB的预测跳转目标是否相同，在IFU的IP级判断L0 BTB是否预测错误。若发生预测错误，则刷新IP级之前的流水级，以BTB中的跳转目标作为新的重定向。可以看到，由于L0 BTB中并没有对跳转方向进行预测而是默认命中的指令就发生跳转，因为BHT的结果是在IP级才给出的，所以在IP级对L0 BTB的判断预测是否正确不仅仅是对跳转目标进行判断，同样的也对跳转方向进行判断。

```verilog
assign l0_btb_hit_l1_btb = (bht_result)? (icache_way0_hit)? l0_btb_hit_taken_way0: l0_btb_hit_taken_way1: (icache_way0_hit)? l0_btb_hit_ntake_way0: l0_btb_hit_ntake_way1;
```

 L0 BTB发生未命中或预测错误时都需要进行更新，更新L0 BTB发生在IB级。发生未命中的情况L0 BTB采用fifo的原则更新，新的指令PC对应的预测信息（比如标志、跳转目标、ras位信息等）代替掉之前储存在第0项的指令PC对应的预测信息。发生预测错误时，则将addrgen模块中计算出的有效地址更新到指令PC对应的表项中。

```verilog
always @(posedge l0_btb_create_clk or negedge cpurst_b)
begin
    if(!cpurst_b)entry_fifo[15:0] <= 16'b1;
    else if(l0_btb_create_en)entry_fifo[15:0] <= {entry_fifo[14:0],entry_fifo[15]};
    else entry_fifo[15:0] <= entry_fifo[15:0];
end
```

## （三）各个模块简单介绍以及相互关系



- Invalidating Status Register

## 总体介绍：

L0BTB总共由：Entry，Write_prep模块，invalid模块，read_enable模块，write_state模块，FIFO模块和命中逻辑模块构成，总共7大总成部分

## 1.**entry：**

### 简单介绍：

保存信息的表项

#### 执行内容：

根据write_prep模块和invalid模块提供的信号对entry_vld  entry_cnt entry_ras tag    way_pred  entry_target信号进行不同的操作，分别是更新，保持和清零。

 

## 2.**Write_prep模块：**

![img](https://uipougzhas1.feishu.cn/space/api/box/stream/download/asynccode/?code=ZmZhN2MwYTNmZTJjMjMwMjE5ZDc0ZTJhZDZkMDAxOWFfbmxkQ1pISXM5TU12V1REc2NQYmI1RHZtbHpvMTUydVZfVG9rZW46TkE5R2JrTHZpb1QybHR4cGtFdWM4MEFTbnBnXzE2ODU2MjcxNjU6MTY4NTYzMDc2NV9WNA)

![img](https://uipougzhas1.feishu.cn/space/api/box/stream/download/asynccode/?code=NzEzMmMzYTQ2MmYyZDAwY2I0ZDEyODU5ZjcwODJkMzFfZUo2czd6TUpuUlBETkdhbFU1UVp3Z1VSNjk5N2s4aWZfVG9rZW46RjI2cGJmcXNMb2NzTVJ4elQ1VGNHalBTbjNmXzE2ODU2MjcxNjU6MTY4NTYzMDc2NV9WNA)

### 简单介绍：

为更新操作完成两项事件：选择需更新的表项和选择更新的数据

#### 选择需更新的表项：

根据ibdp和addr的更新有效信号，选择接受来自哪一者的需更新表项，并将其中信号其连接至16个表项的entry_update端口；

#### 选择更新的数据：

根据addr和ibdp的更新的vld信号来选择即将更新进entry的数据来自哪里，若两者皆无效，则全部置零

 

## 3.**invalid模块：**

### 简单介绍：

对表项的无效化信号进行控制

#### 更新条件：

控制entry无效化的信号由一个寄存器产生，寄存器的时钟来的局部使能信号是其自身输出和ifctrl_l0_btb_inv的或，也即当entry在此周期将被无效化或者此周期得知其需要被无效化时，在下一个时钟上升沿来到后才可能更新。。

#### 更新内容：

1，若cpu整体重置，则其无效化信号位1，也即将在下个周期无效化所有表；

2，若其无效化信号为0，而外部控制信号将其无效化，则其无效化信号也置1.

3，若都不是，则无效化信号位保持不变。

对entry的影响：

此信号讲连接至表项的entry_inv，也即其无效化会在一到两个周期后起作用。

 

## 4.**read_enable模块**

### 简单介绍：

确定是否需要进行读取数据的操作，并生成两个信号entry_hit_flop和ras_pc

#### 确定是否需要进行读取数据的操作：

l0_btb_rd为l0btb的读使能信号，而l0_btb_rd只有在btb为使能有效状态且l0btb为使能有效状态且不遮掩 l0_btb的chgflw且（l0_btb的chgflw有效或 l0_btb不暂停），下有效。

 

#### entry_hit_flop信号：

​       意义：判断传递过来的tag与l0btb的表项中哪一项相匹配

​       前置信号：entry_hit 为 16个表项信号比较结果 和 更新的表项信号比较结果 的汇总。

更新条件：在l0btb处于读使能激活状态时或l0btb上一个时钟处于读使能激活状态且l0btb没有被暂停时

更新内容：entry_hit信号。

flop信号的原因：存在更新的表项信号，更新后的正确值，而此时需要过一拍才能发生更新。

 

#### Ras_pc信号：

​       意义：对于retrun指令的地址判断结果

​       赋值原则为：先判断ras到l0btb的push信号是否有效，有效则采用ras直接push的值（ras中最新的ruturn地址），否则看ipdp给l0btb的push信号，有效则采用ipdp级push的值（ipdp中最新的ruturn地址）。最后才选用的ras_l0_btb_pc的值（此时嵌套应该开始返回）。

 

## 5.**write_state模块：**

### 简单介绍：

对l0btb的工作状态进行判断，并输出两种状态WAIT（“等待写入”/工作模式）和相反的IDLE状态

#### 工作状态进行判断：

​    在IDLE状态下，如果pcgen_l0_btb_chgflw_vld有效，便进入WAIT，否则保持IDLE。

​       在WAIT状态下，要求pcgen_l0_btb_chgflw_mask无效且ipctrl_l0_btb_chgflw_vld，ipctrl_l0_btb_wait_next，！ipctrl_l0_btb_ip_vld任意一个有效，则保持WAIT，否则进入IDLE。

pcgen_l0_btb_chgflw_vld：表示有效的流水变化情况。

pcgen_l0_btb_chgflw_mask：表示无效的流水变化情况。

ipctrl_l0_btb_chgflw_vld，ipctrl_l0_btb_wait_next，！ipctrl_l0_btb_ip_vld三个为ipctrl给与l0btb的信号。

 

## 6.**FIFO模块**

### **简单介绍：**

**表项的更新原则**

#### entry_fifo[15:0] 信号：

每一位分别为对应的l0_btb表项的更新使能信号，决定了l0_btb应该更新哪一个表项。

#### 更新情况：

L0 BTB发生未命中或预测错误时都需要进行更新

发生未命中时，代替掉之前储存在第0项的指令PC对应的预测信息。

发生预测错误时，则将addrgen模块中计算出的有效地址更新到指令PC对应的表项中。

 

## 7.**命中逻辑**

### 概括：

根据entry_hit_flop，ras_pc和表项内容，进行命中逻辑判断，并给出结果，包括是否命中entry_chgflw_vld，命中地址entry_hit_target，地址形式（是否为ras），命中路预测信息entry_hit_way_pred，以及相关信号。最后将信号传递给ifctrl

### 对`Entry Hit Logic`部分进行解读。

 

```Plain
//Only when Counter == 1,L0 BTB can be hit
assign entry_hit_counter = entry_hit_flop[0]&entry0_cnt |entry_hit_flop[1]&entry1_cnt|...
```

  `entry_hit_counter`为16个与项的或，每个与项形式为`entry_hit_flop[i]&entry$i_cnt`  其中`entry_hit_flop[i]`上面已经解释，`entry$i_cnt`是每个表项的输出信号，表示是否重定向的一种加强条件  `entry_hit_counter`为1表示至少有一个表项同时满足两个条件，命中

~~~Plain
```verilog
//Indicate whether entry record is ras
assign entry_hit_ras = entry_hit_flop[0]&entry0_ras | entry_hit_flop[1]&entry1_ras |...   
~~~

 

```
entry_hit_ras`为16个与项的或，每个与项形式为`entry_hit_flop[i]&entry$i_ras
```

 

`entry$i_ras`是每个表项的输出信号，表示是否是ras预测类型

```Plain
//Indicate entry record for way prediction
assign entry_hit_way_pred[1:0] = ({2{entry_hit_flop[0]}}  & entry0_way_pred[1:0])
                               | ({2{entry_hit_flop[1]}}  & entry1_way_pred[1:0])
                               | ...
```

 

```
entry_hit_way_pred`为16个与项的或，每个与项形式为`{2{entry_hit_flop[i]}}  & entry$i_way_pred[1:0]
```

 

和上面基本同理，根据命中情况和表项路预测情况处理得到最终的命中路

```Plain
//L0 btb prediction target,it excludes ret result                               
assign entry_hit_pc[19:0] = ({20{entry_hit_flop[0]}} & entry0_target[19:0])
                            | ({20{entry_hit_flop[1]}} & entry1_target[19:0]) 
                            | ...
```

 

```
entry_hit_pc`为16个与项的或，每个与项形式为`{20{entry_hit_flop[i]}}  & entry$i_target[19:0]
```

 

和上面基本同理，把命中情况与pctarget进行按位与，确定最终的命中pc

```Plain
assign entry_if_hit           = |entry_hit_flop[15:0];
assign entry_chgflw_vld       = entry_if_hit && entry_hit_counter;
assign entry_hit_target[PC_WIDTH-2:0] = (entry_hit_ras)
                                        ? ras_pc[PC_WIDTH-2:0]
                                        : {pcgen_l0_btb_if_pc[PC_WIDTH-2:20],entry_hit_pc[19:0]};
```

 

`entry_if_hit`:至少命中一个表项

 

`entry_chgflw_vld`:绝对命中，需要重定向更新

 

`entry_hit_target`如果命中RAS，目的PC去RAS中找，否则使用之前得出的`entry_hit_pc`,高位用模块传入的PC补齐

 

 对ifctrl的输出：

~~~Plain
//Interface with IF at the end
assign l0_btb_ifctrl_chglfw_vld              = entry_chgflw_vld;
assign l0_btb_ifctrl_chgflw_pc[PC_WIDTH-2:0] = entry_hit_target[PC_WIDTH-2:0];
assign l0_btb_ifctrl_chgflw_way_pred[1:0]    = entry_hit_way_pred[1:0];
//output to ifdp
assign l0_btb_ifdp_chgflw_pc[PC_WIDTH-2:0] = entry_hit_target[PC_WIDTH-2:0];
assign l0_btb_ifdp_chgflw_way_pred[1:0]    = entry_hit_way_pred[1:0];
assign l0_btb_ifdp_entry_hit[15:0]         = entry_hit_flop[15:0];
assign l0_btb_ifdp_hit                     = entry_if_hit;
assign l0_btb_ifdp_counter                 = entry_hit_counter;
assign l0_btb_ifdp_ras                     = entry_hit_ras;
//Debug infor
assign l0_btb_debug_cur_state[1:0] = l0_btb_cur_state[1:0];
```
~~~

## 相互关系：

[![pCpw2zd.png](https://s1.ax1x.com/2023/06/03/pCpw2zd.png)](https://imgse.com/i/pCpw2zd)

# 需要更新l0 btb的情况 

## 返回地址预测（ras)

需要进行返回地址更新的情况：

* ras miss: 遇到return指令，但l0 btb中不存在该表项或者命中的表项的ras标志位为0，即没有命中L0 BTB。此时更新L0 BTB的对应的表项。这样当下一次该return指令到达时，就可以直接使用L0 BTB中的预测结果进行PC的重定向。

* ras mistaken: 不是return指令，但命中的表项ras标志位为1。即指令包中没有return指令，却发生了L0 BTB中return的命中的情况，`l0_btb_ras_mistaken` 信号为1。此时就将这个表项命中失效，当下一个相同PC的指令包到达L0 BTB时不命中。

* ras mispred: 是return指令，命中的表项ras标志位也为1，但返回地址计算错误

  从L0 BTB读出的跳转目标会传递到IP级，在IP级和从ras模块中栈顶读出的值比较是否相等判断是否预测正确，如果发生预测错误则根据ras模块堆栈中的信息更新L0 BTB。

```verilog
assign l0_btb_ras_update     = (l0_btb_ras_miss 
                               || l0_btb_ras_mispred 
                               || l0_btb_ras_mistaken);
//ras的miss，mispred, mistaken都归于同一类更新  
assign l0_btb_ras_mistaken = ibctrl_ibdp_if_chgflw_vld
                         && ipdp_ibdp_l0_btb_ras
                         && !ibctrl_ibdp_ip_chgflw_vld
                         && ib_data_vld
                         && !(|ibdp_hn_preturn[7:0]);
assign l0_btb_ras_miss    = (ibctrl_ibdp_if_chgflw_vld
                            && !ipdp_ibdp_l0_btb_ras
                            || !ipdp_ibdp_l0_btb_hit)
                         && l0_btb_st_wait
                         && ib_data_vld
                         && (|ibdp_hn_preturn[7:0])
                         && cp0_ifu_ras_en;
assign l0_btb_ras_mispred = ibctrl_ibdp_if_chgflw_vld
                         && ipdp_ibdp_l0_btb_ras
                         && !ipdp_ibdp_l0_btb_ras_pc_hit
                         && l0_btb_st_wait
                         && ib_data_vld
                         && (|ibdp_hn_preturn[7:0])
                         && cp0_ifu_ras_en; 
```

## 分支跳转预测 (br)

### 表项缺失

如前文所述，筛选出来存入l0 btb的指令PC对应的指令类型有三种，分别为strongly taken的条件跳转指令（预测状态位为11）、绝对跳转指令、以及return指令。

在IP级给出分支表项缺失的信号后，延迟一拍传递给IB级，作为l0 btb更新的控制信号源。

### 预测错误br_mispred

预测错误信号可由ibdp与addrgen两个模块给出。

* ibdp： 

  ibdp的`br_mispred`信号实际上在**IP级**给出，产生 `l0_btb_mispred` 信号传递到IB级用于更新L0 BTB，包括两种情况：

  1. bht预测跳转，l0 btb也发送跳转指令，与bht结果相符。但此时命中的表项并不包含在l1 btb中，违背包含关系
  2. bht预测不跳转，但l0 btb发送了跳转指令。
  
  ```verilog
  assign l0_btb_mispred = ip_if_pcload
                          && !ipdp_ipctrl_l0_btb_ras
                          && !l0_btb_hit_l1_btb 
                          && ip_pcload 
                          && l0_btb_ipctrl_st_wait
                       || ip_chgflw_mistaken
                          && l0_btb_ipctrl_st_wait;
  ```
  
  遇到这两类情况时不包含PC的计算，可提前给出预测错误信号 ，不需等到addrgen中得出PC计算结果后进行更新。
  
* addrgen：专门针对跳转地址预测错误（包括返回地址预测和分支预测），涉及到预测PC（`branch_pred_result`）和正确PC（`branch_cal_result`）的计算。

  由于涉及到l0 btb预测错误的绝大多数情况在ibdp级会提前得到。因此addrgen主要用于处理l0 btb中表项缺失，而l1 btb预测错误的情况。
  
  ```verilog
  assign branch_mispred = (branch_pred_result[PC_WIDTH-2:0] != branch_cal_result[PC_WIDTH-2:0]);
  ```
  
  
  
  * 预测PC：低位和高位拼接，在ip级得到结果
    * 低20位来源于BTB预测结果
    * 高位来源于pcgen
  * 正确PC：基地址base+偏移量offset
    * 均来源于预译码，在ip级得到结果
    * 一个指令的偏移量是用21位表示的，而**不同的分支类型要求提取的偏移量位置不同**，因此在`decd_normal`模块分别提取了**五种分支类型**的偏移量信息。
  * 若预测PC和正确PC不一致则发生addrgen模块的PC重定向，将指令译码后计算得到的有效地址发送到pcgen模块，刷掉前级流水线的有效信息

### 其他情况

除了表项缺失和预测错误之外，在C910源码的ipdp模块中列出了以下几种情况需要更新l0 btb

1. l0 btb命中，但bht结果为弱跳转。根据筛选的原则应将该表项筛出。

   ```verilog
   assign l0_btb_not_saturate = ipctrl_ipdp_if_pcload
                             && l0_btb_hit_l1_btb
                             && ipctrl_ipdp_ip_pcload
                             && !ifdp_ipdp_l0_btb_ras
                             && ipctrl_ipdp_con_br 
                             && (bht_pre_result[1:0] == 2'b10);
   ```

   

2. l0 btb命中，但表项的cnt位为0。且该命中的表项也在BTB中，且L0 BTB中该表项cnt为0，且未产生mistaken的情况，并且预测器预测该命中L0 BTB且命中BTB的条件分支为strongly taken。L0 BTB触发重定向的条件需要同时满足命中L0 BTB中的表项以及表项对应的cnt位为1，故需要更新。满足前文所述条件时，L0 BTB中该表项的cnt位更新为1，意味着之后指令流中的指令再次命中L0 BTB该表项时，会对前端进行重定向。

   ```verilog
   assign l0_btb_counter_zero = ifdp_ipdp_l0_btb_hit
                             && !ifdp_ipdp_l0_btb_counter
                             && l0_btb_hit_l1_btb
                             && !ifdp_ipdp_l0_btb_ras
                             && ipctrl_ipdp_ip_pcload
                             && ipctrl_ipdp_con_br
                             && (bht_pre_result[1:0] == 2'b11);
   ```

3. 指令包在IF级命中L0 BTB并且发生重定向，然而IP级发现该指令包中根本没有预测跳转的分支指令，说明IF级发生了错误的重定向。**实质上也为l0 btb预测错误**，在ip级延迟一拍后将值传递给ibdp的br_mispred信号，使其有效。（基于此，在具体分析更新机制时将该信号归为分支预测错误进行讨论）

   ```verilog
   assign l0_btb_mistaken   = ipctrl_ipdp_ip_mistaken;
   
   assign l0_btb_mispred = ip_if_pcload
                           && !ipdp_ipctrl_l0_btb_ras
                           && !l0_btb_hit_l1_btb 
                           && ip_pcload 
                           && l0_btb_ipctrl_st_wait
                        || ip_chgflw_mistaken
                           && l0_btb_ipctrl_st_wait;
                           
   assign ipctrl_ipdp_ip_mistaken = ip_chgflw_mistaken;
   ```

   <a href="https://imgse.com/i/pCpwtG4"><img src="https://s1.ax1x.com/2023/06/03/pCpwtG4.png" alt="pCpwtG4.png" border="0" /></a>

# l0 btb更新机制

在l0_btb中，新的指令PC对应的预测信息（比如标志、跳转目标、ras位信息等）将传至write preparation模块，在下一个时钟周期写入需要更新的表项中。

```verilog
casez({addrgen_l0_btb_update_vld,
      ibdp_l0_btb_update_vld})
2'b1? : begin
         l0_btb_wen[3:0]           = addrgen_l0_btb_wen[3:0];
         l0_btb_update_vld_bit     = addrgen_l0_btb_update_vld_bit;
         l0_btb_update_cnt_bit     = 1'b0;
         l0_btb_update_ras_bit     = 1'b0;
         l0_btb_update_data[36:0]  = 37'b0;
         end
2'b01 : begin
         l0_btb_wen[3:0]           = ibdp_l0_btb_wen[3:0];
         l0_btb_update_vld_bit     = ibdp_l0_btb_update_vld_bit;
         l0_btb_update_cnt_bit     = ibdp_l0_btb_update_cnt_bit;
         l0_btb_update_ras_bit     = ibdp_l0_btb_update_ras_bit;
         l0_btb_update_data[36:0]  = ibdp_l0_btb_update_data[36:0];
         end
default: begin
         l0_btb_wen[3:0]           = 4'b0;
         l0_btb_update_vld_bit     = 1'b0;
         l0_btb_update_cnt_bit     = 1'b0;
         l0_btb_update_ras_bit     = 1'b0;
         l0_btb_update_data[36:0]  = 37'b0;
         end
endcase
```

可以看到l0 btb的更新数据源来自于两个模块：ibdp和addrgen. 而addrgen更新的优先级更高。

由前文关于需要更新l0 btb的情况分析，addrgen只负责分支预测错误的情况的更新，需要进行PC的运算与比较。由以上源码可知，当addrgen判断预测错误时，会将表项无效（vld位置0），cnt位置零。由于是分支指令，ras位自然为0. 此外，表项中的数据也清零。

由于ibdp涉及到的更新类型较多，此处为其独立设置子目录进行总结。

### ibdp

* 有关ras的表项更新（miss, mispred,mistaken)

  * mistaken时vld位为0，使l0 btb表项无效化。
  * 而miss或mispred的时候vld仍为1，表项仍有效。仍然保持ras位为1。
  * miss的情况和分支预测表项缺失大致相同，但更新的data有区别。高位相同，但icache开启两路路预测，且低20位全复位为0。

*  分支表项缺失

  表项仍然有效。但会进行绝对跳转指令的判断。若为绝对跳转指令，则cnt位仍为1；否则cnt位清0，对应表项的前端的重定向被无效化。

* 不需计算地址的分支跳转预测错误

  将表项无效化，vld, cnt, ras位全部赋为0，表项中的数据也会清零。与addrgen中预测错误进行的更新处理相同。

* 其他需要更新l0 btb的情况

  |                            | vld  | cnt  |
  | -------------------------- | ---- | ---- |
  | bht为弱跳转                | 0    | 0    |
  | 命中表项但counter为0       | 1    | 1    |
  | 命中表项但实际上无分支指令 | 0    | 0    |

```verilog
case({l0_btb_ras_update,
      l0_btb_br_miss,
      l0_btb_br_mispred})
    
  //ras缺失/预测错误
  3'b100: begin
          l0_btb_wen[3:0]          = 4'b1111;
          l0_btb_update_vld_bit    = !l0_btb_ras_mistaken; 
          l0_btb_update_cnt_bit    = 1'b1;
          l0_btb_update_ras_bit    = 1'b1;
          l0_btb_update_data[36:0] = 
            {
             ibdp_vpc[14:0],                 //entry_tag 
             2'b11,                          //entry_way_pred
             20'b0                           //entry_target
            };
          end
    
  //分支预测表项缺失
  3'b010: begin       
          l0_btb_wen[3:0]          = 4'b1111;
          l0_btb_update_vld_bit    = 1'b1;
          l0_btb_update_cnt_bit    = |ibdp_hn_jal[7:0]; 
          l0_btb_update_ras_bit    = 1'b0;
          l0_btb_update_data[36:0] = 
            {ibdp_vpc[14:0],                 //entry_tag 
             ipdp_ibdp_branch_way_pred[1:0], //entry_way_pred
             ipdp_ibdp_branch_result[19:0]   //entry_target
            };
          end
    
  //分支预测错误，全部清零
  3'b001: begin
          l0_btb_wen[3:0]          = 4'b1000;
          l0_btb_update_vld_bit    = 1'b0; //会使entry_vld为0
          l0_btb_update_cnt_bit    = 1'b0;
          l0_btb_update_ras_bit    = 1'b0;
          l0_btb_update_data[36:0] = 37'b0; 
          end

  default: begin //br_update
          l0_btb_wen[3:0]          = ipdp_ibdp_l0_btb_wen[3:0];
          l0_btb_update_vld_bit    = ipdp_ibdp_l0_btb_update_vld_bit;
          l0_btb_update_cnt_bit    = ipdp_ibdp_l0_btb_update_cnt_bit;
          l0_btb_update_ras_bit    = 1'b0;
          l0_btb_update_data[36:0] = 37'b0; 
          end
```

### 关于相关信号的补充说明

#### entry_cnt

表项对应的cnt位主要用于判断命中的表项是否需要进行前端的重定向。**若该表项的cnt位为0，即使命中该L0 BTB表项，在IF级仍不会触发L0 BTB对前端进行重定向**。
```verilog
//Only when Counter == 1,L0 BTB can be hit
assign entry_hit_counter       = entry_hit_flop[0]  & entry0_cnt
                               | entry_hit_flop[1]  & entry1_cnt
                               | entry_hit_flop[2]  & entry2_cnt
                               ...
```
对于某一条条件分支指令，L0 BTB发生miss时，将会清除L0 BTB的cnt位，由于L0 BTB触发重定向的条件需要同时满足命中L0 BTB中的表项以及表项对应的cnt位为1，在条件分支指令对应的表项刚刚更新时将不会触发对前端流水线的重定向。直到L0 BTB的命中信息传递到IP级，在该级发现指令包中存在预测强跳转的条件分支指令，且此时对应L0 BTB命中的表项中的cnt位为0，在该指令包到达IB级下一周期才会将cnt位置1。

之后若命中该条件分支指令对应的表项将会产生重定向信号。之后如果在IP级检测到该条件分支的预测跳转方向由强跳转变为了弱跳转，在该指令包到达IB级下一周期会将该条件分支指令在L0 BTB中对应的表项无效（清零vld位）。

####  entry_ras

L0 BTB中可以对绝对分支指令，强跳转的条件分支指令，以及return指令进行快速的跳转目标预测。RAS代表该L0 BTB表项对应储存的是否是一条return指令的返回地址，即该L0 BTB表项是RAS表项。

```verilog
//l0_btb
assign entry_hit_ras           = entry_hit_flop[0]  & entry0_ras
                               | entry_hit_flop[1]  & entry1_ras
                               | ...
    
assign entry_hit_target[PC_WIDTH-2:0] = (entry_hit_ras)
                                        ? ras_pc[PC_WIDTH-2:0]
                                        : {pcgen_l0_btb_if_pc[PC_WIDTH-2:20],entry_hit_pc[19:0]};    
```

#### 写使能信号wen[3:0]

wen[3]: entry_vld的赋值

```verilog
  else if(entry_wen[3] && entry_update_en)
    entry_vld           <= entry_update_vld;
```

wen[2]: entry_cnt的赋值

```verilog
  else if(entry_wen[2] && entry_update_en)
    entry_cnt           <= entry_update_cnt;
```

wen[1]: entry_ras的赋值

```verilog
  else if(entry_wen[1] && entry_update_en)
    entry_ras           <= entry_update_ras;
```

wen[0]: entry_update_data的赋值--> tag, way_pred, target

```verilog
  else if(entry_wen[0] && entry_update_en)
  begin
    entry_tag[14:0]     <= entry_update_data[36:22];
    entry_way_pred[1:0] <= entry_update_data[21:20];
    entry_target[19:0]  <= entry_update_data[19:0];
  end
```



### 更新机制

#### 表项缺失

如前文所述，发生**未命中L0 BTB**(ras_miss和br_miss)时, 会判断该条件跳转指令或绝对跳转指令**是否存在于BTB内**，

1. 如果存在则发生L0 BTB缺失，
2. 若不存在会首先更新BTB，

   下一次该指令被取出时发生BTB命中而L0 BTB未命中时，才判断L0 BTB缺失，更新L0 BTB。

```verilog
assign l0_btb_br_miss    = l0_btb_br_miss_pre
                        && !ibdp_btb_miss
                        && !l0_btb_ras_update;
```

发生表项缺失时，L0 BTB采用**fifo**的原则更新，决定了下一个时钟周期`entry_fifo[15:0]`是否移位，其信号源来自IB级。

```verilog
//ibdp
assign ibdp_l0_btb_fifo_update_vld   = (l0_btb_ras_miss || l0_btb_br_miss)
                                    && ib_data_vld
                                    && !ipctrl_ibdp_expt_vld
                                    && !ibctrl_ibdp_self_stall;

//l0 btb - fifo
assign l0_btb_create_en = ibdp_l0_btb_fifo_update_vld 
                       && cp0_ifu_btb_en
                       && cp0_ifu_l0btb_en;

always @(posedge l0_btb_create_clk or negedge cpurst_b)
begin
  if(!cpurst_b)
    entry_fifo[15:0] <= 16'b1;
  else if(l0_btb_create_en)
    entry_fifo[15:0] <= {entry_fifo[14:0],entry_fifo[15]};
  else
    entry_fifo[15:0] <= entry_fifo[15:0];
end
```

fifo模块中的`entry_fifo[15:0]`（波形图中对应`io_l0_btb_ibdp_entry_fifo`，通过组合逻辑直接相连，故认为两个信号等价）的每一位分别为对应的l0_btb表项的更新使能信号，即最终决定l0 btb将更新哪个表项。从源码中可知复位时`entry_fifo`的值为1，对应更新第0号表项。发生表项缺失时，每次更新时左移一位，更新完第15号表项后下一次更新第0号表项，如此循环。最终代码运行的效果为：发生表项缺失时，l0_btb按顺序更新表项数据。

[![pCpwUz9.png](https://s1.ax1x.com/2023/06/03/pCpwUz9.png)](https://imgse.com/i/pCpwUz9)

在波形图中，可以看到entry_fifo为0x0200, 0x0400, 0x0800时都发生表项缺失，此时来自于ibdp表项缺失信号与需要更新的表项相关数据在延迟一拍后得出。而关于更新前一拍update_entry为0x00的原因为：由于l0_btb的`entry_fifo[15:0]`刚移位得到更新的值时，ibdp中选择信号l0_btb_ras_miss等尚未更新，需要在ip级延迟一拍才能得到，故此时ibdp_hit_entry[15:0]选择的是第二项ipdp_ibdp_l0_btb_entry_hit[15:0]，而该项的数据源为l0 btb中判断命中的表项的entry_hit[15:0]，该信号会在if级和ip级分别延迟一拍。由于entry_fifo为0x0200与0x0400时都没有命中，entry_hit[15:0]都为0x0000，故在表项为0x0400和0x0800时更新前一拍时ibdp的update_entry[15:0]为0x0000。



```verilog
//ibdp
//括号中的逻辑是对表项缺失情况进行的判断。对于后一项，由于ras_update信号为ras_miss, ras_mispred, ras_mistaken相或。为了排除后两种情况而只考虑分支缺失，ras_update信号须为0并另外和br_miss相与。
assign l0_btb_hit_entry[15:0] = (l0_btb_ras_miss || l0_btb_br_miss_pre && !l0_btb_ras_update)
                              ? l0_btb_ibdp_entry_fifo[15:0]
                              : ipdp_ibdp_l0_btb_entry_hit[15:0];

//output to addrgen
assign ibdp_addrgen_l0_btb_hit_entry[15:0] = (ibctrl_ibdp_l0_btb_hit)
                                           ? ipdp_ibdp_l0_btb_entry_hit[15:0]
                                           : l0_btb_ibdp_entry_fifo[15:0];

assign l0_btb_update_vld_for_gateclk = (l0_btb_br_miss_pre 
                                       || l0_btb_br_mispred_pre
                                       || l0_btb_br_update_pre
                                       || l0_btb_ras_mistaken
                                       || l0_btb_ras_miss
                                       || l0_btb_ras_mispred)
                                    && ib_data_vld;

//addrgen
else if(branch_vld)
addrgen_l0_btb_hit_entry[15:0]         <= l0_btb_hit_entry[15:0];

assign addrgen_l0_btb_update_entry[15:0] = addrgen_l0_btb_hit_entry[15:0];
```



```verilog
//l0 btb - write preparation
assign l0_btb_update_entry[15:0] = 
   {16{addrgen_l0_btb_update_vld}}          & addrgen_l0_btb_update_entry[15:0]
 | {16{l0_btb_update_vld_for_gateclk}}      & ibdp_l0_btb_update_entry[15:0];


//此处以entry0为例，其它15个表项类似。
ct_ifu_l0_btb_entry  x_l0_btb_entry_0 (
    ...
	.entry_update             (l0_btb_update_entry[0]  ),
    ...
);
```

此外，由于l0 btb中还包括对绝对跳转指令目标的预测，在发生分支预测表项缺失时，在IB级还会另外判断当下指令是否为绝对跳转指令。若为绝对跳转指令，即使br_miss信号为真，表项的cnt位仍然为1，下一次命中此表项时仍可触发前端重定位。

[![pCpI9Df.png](https://s1.ax1x.com/2023/06/03/pCpI9Df.png)](https://imgse.com/i/pCpI9Df)

如波形图所示，在entry_fifo为0x0004时和0x0008时都发生了分支预测表项缺失，但不同在于entry_fifo为0x0008时对应的指令为绝对跳转指令，而0x0004不是，故在更新时将0x0004表项的cnt位置0，0x0008表项的cnt位仍为1。

#### 预测错误

由前文对于ibdp和addrgen的entry-fifo表项的分析，ibdp中返回地址预测错误（`ras_mispred` & `ras_mistaken`）和分支预测错误（`br_mispred`）时对应的更新表项为`ipdp_ibdp_l0_btb_entry_hit[15:0]`，为l0 btb中的命中表项`entry_hit[15:0]`在ifdp和ipdp延迟两拍后得到。

[![pCpw0qx.png](https://s1.ax1x.com/2023/06/03/pCpw0qx.png)](https://imgse.com/i/pCpw0qx)



而addrgen判断发生跳转目标预测错误时，由于此时l0 btb表项缺失，判断的是l1 btb的预测跳转目标（比l0 btb的结果晚一拍得出），故addrgen中需要更新的表项为`entry_fifo`中的表项而不是`entry_hit`的表项。此时addrgen中对应的l0 btb中需要更新的表项应比ibdp中的entry_fifo再延迟一拍，即相较于l0 btb中的`entry_fifo`延迟两个周期。

```verilog
assign ibdp_addrgen_l0_btb_hit_entry[15:0] = (ibctrl_ibdp_l0_btb_hit)
                                           ? ipdp_ibdp_l0_btb_entry_hit[15:0]
                                           : l0_btb_ibdp_entry_fifo[15:0];
```

（从波形图中可以看到在`entry_fifo`为0x0800时，l0 btb发生表项缺失，l1 btb发生预测错误）

[![pCpwDZ6.png](https://s1.ax1x.com/2023/06/03/pCpwDZ6.png)](https://imgse.com/i/pCpwDZ6)



预测错误时，不会进行fifo移位操作，`entry_fifo`保持不变。直到对应表项更新完成，将从预测错误前的`entry_fifo`表项重新开始，进行后续的表项更新操作。



#### 其他情况

1. l0 btb命中，但bht结果为弱跳转。根据筛选的原则应将该表项筛出。（not saturate)

   在通过组合逻辑得出not_saturate一系列信号中取ifdp_ipdp_l0_btb_ras进行分析：

   在l0_btb得到entry_hit后，先将其延迟一拍得到entry_hit_flop并进行hit_ras的判断。

   ```verilog
   always@(posedge forever_cpuclk or negedge cpurst_b)
       ...
   	else if(l0_btb_rd_flop && !ifctrl_l0_btb_stall)
       	entry_hit_flop[15:0] <= entry_hit[15:0];
   	...
   
   assign entry_hit_ras           = entry_hit_flop[0]  & entry0_ras
                                  | entry_hit_flop[1]  & entry1_ras
                                  | entry_hit_flop[2]  & entry2_ras
                                  ...
   ```

   然后在ifdp再延迟一拍后传递到ipdp级。

   ```verilog
   //ifdp
   else if(ifctrl_ifdp_pipedown)
   ...
   	l0_btb_ras                      <= l0_btb_ifdp_ras;
   ...
   
   assign ifdp_ipdp_l0_btb_ras                     = l0_btb_ras;
   ```

   因此，not_saturate信号在l0 btb的entry_hit信号后两拍得到。

   而not_saurate信号得出后, ipdp的update_vld会随之得出，并在ipdp内延迟一拍之后传递给ibdp，作为l0_btb的更新信号。

   ```verilog
   assign l0_btb_update_vld    = ipctrl_ipdp_ip_data_vld
                              && (l0_btb_not_saturate
                                  || l0_btb_mistaken
                                  || l0_btb_counter_zero);
   
   else if(pipe_vld && !pipe_stall || rtu_yy_xx_dbgon)
   	...
   	ipdp_ibdp_l0_btb_update_vld               <= pipe_l0_btb_update_vld;
   	... 
   	
   ```

   [![pCpwsIO.png](https://s1.ax1x.com/2023/06/03/pCpwsIO.png)](https://imgse.com/i/pCpwsIO)	

2. l0 btb命中，但表项的cnt位为0，且该命中的表项也在BTB中，且L0 BTB中该表项cnt为0，且未产生mistaken的情况，并且预测器预测该命中L0 BTB且命中BTB的条件分支为strongly taken。此时将L0 BTB中该表项的cnt位更新为1，意味着之后指令流中的指令再次命中L0 BTB该表项时，会对前端进行重定向. 

   [![pCp4X60.png](https://s1.ax1x.com/2023/06/03/pCp4X60.png)](https://imgse.com/i/pCp4X60)
   
   从波形图中可以看出，相关信号时序与not_saturate大致相同，但传递到ibdp的更新信号多保持了一个时钟周期。通过进一步观察波形图可发现，当counter_zero有效时，pipe_vld为0，即ip级的数据无效。
   
   [![pCpHo7V.png](https://s1.ax1x.com/2023/06/03/pCpHo7V.png)](https://imgse.com/i/pCpHo7V)
   
   ```verilog
   //ipdp
   else if(pipe_vld && !pipe_stall || rtu_yy_xx_dbgon)
       ...
       ipdp_ibdp_l0_btb_update_vld               <= pipe_l0_btb_update_vld;
   	...
   else
       ...
       ipdp_ibdp_l0_btb_update_vld               <= ipdp_ibdp_l0_btb_update_vld;
   	...
   
   assign pipe_vld             = ipctrl_ipdp_pipe_vld;
   //ipctrl
   assign ipctrl_ipdp_pipe_vld      = ip_vld;
   assign ip_vld = (ip_data_vld || ip_expt_vld) &&
                   !icache_chk_err_refill &&
                   !icache_chk_err_refill_ff && 
                   !pcgen_ipctrl_cancel && 
                   !ip_self_stall && 
                   !rtu_ifu_xx_dbgon;
   ```
   
   这是由于本应进行命中的l0 btb表项预测目标的前端的重定向，但由于该表项cnt位为0而没有进行。在IP级得出需要重定向的结果后，会将IF级、IP级数据无效化（IF无效化与counter_zero在同一拍得出，IP无效化在IF级无效化之后一拍），导致相关更新信号由于锁存器的作用并未立刻清零，而是将有效后的值保持多了一个时钟周期。
   
   ```verilog
   //counter_zero有效的条件之一为：在ip级得出要进行重定向的结果
   assign l0_btb_counter_zero = ...
                             && ipctrl_ipdp_ip_pcload
                             ...
   //pcgen_chgflw_without_l0_btb为pcgen中控制IF级、IP级有效的信号，在counter_zero有效时，易见该信号同样有效，会使if级和ip级的数据无效，即令if_vld和ip_vld为0。（在同一时钟周期使if无效，将ip级数据无效化需要延迟一个时钟周期）
   assign pcgen_chgflw_without_l0_btb = had_ifu_pcload || 
                                        vector_pcgen_pcload ||
                                        rtu_ifu_chgflw_vld ||
                                        iu_ifu_chgflw_vld ||
                                        addrgen_pcgen_pcload ||
                                        ibctrl_pcgen_pcload ||
                                        ipctrl_pcgen_chgflw_pcload || //counter_zero有效时该信号为1
                                        ipctrl_pcgen_reissue_pcload ||
                                        ifctrl_pcgen_reissue_pcload;
   ```
   
   
