# ns2 评估路由协议的一般过程

## 1. 运行文件

### 1.1 生成随机数据流

生成随机数据流

cbrgen.tcl

文件位置：~/ns-allinone-2.35/ns-2.35/indep-utils/cmu-scen-gen/

在该位置下输入命令：

```
ns cbrgen.tcl -type cbr -nn 50 -seed 1 -mc 10 -rate 2.0 > cbr1
```

**该命令创建了一个具有50个移动节点、10对通信连接、每秒钟发送2个分组的以CBR为业务源的通信场景文件cbr1**

### 1.2 生成场景文件

在终端~/ns-allinone-2.35/ ns-2.35/indep-utils/cmu-scen-gen/setdest/路径下输入命令：

```
./setdest -n 50 -p 0 -M 20 -t 300 -x 1000 -y 300 > scene1
```

注：如果在终端~/ns-allinone-2.35/ns-2.35/indep-utils/cmu-scen-gen/setdest/路径下输入命令：

cp setdest /usr/local/bin/

将可执行文件setdest复制到/usr/local/bin/文件夹目录里，即可在任意文件夹路径下使用setdest命令。【建议做这一步】

**该命令创建一个具有50个节点、节点在每个地点停留0秒(即不停留)、最大移动速度20m/s,仿真时间300秒，长1000米，宽300米的移动场景文件 scene1**

### 1.3 将上述两个文件复制到需要运行文件同一位置

将生成的随机数据流文件cbr1与场景文件scene1拷贝到运行的程序所在文件夹中(如~/ns-allinone-2.35/ ns-2.35/aodv/testfile/)

### 1.4 运行

在终端输入~/ns-allinone-2.35/ ns-2.35/AODV/testfile/路径下输入命令：

ns aodv.tcl

![image-20230302083658103](C:\Users\27252\AppData\Roaming\Typora\typora-user-images\image-20230302083658103.png)



```tcl
#=====================================
#Define options
#=====================================
set val(chan)           Channel/WirelessChannel    ;#Channel Type
set val(prop)           Propagation/TwoRayGround   ;# radio-propagation model
set val(netif)          Phy/WirelessPhy            ;# network interface type
set val(mac)            Mac/802_11                 ;# MAC type
set val(ifq)            Queue/DropTail/PriQueue    ;# interface queue type
set val(ll)             LL                         ;# link layer type
set val(ant)            Antenna/OmniAntenna        ;# antenna model
set val(ifqlen)         50                         ;# max packet in ifq
set val(nn)             50                          ;# number of mobilenodes
set val(rp)             AODV                       ;# routing protocol
set opt(cp) "cbr1";
set opt(sc) "scene1";

#===============================

# Main Program

#===============================

set ns_ [new Simulator]
set tracefd [open aodv.tr w]
$ns_ trace-all $tracefd
$ns_ use-newtrace
set namtracefd [open aodv.nam w]
$ns_ namtrace-all-wireless $namtracefd 1000 300
set topo [new Topography]
$topo load_flatgrid 1000 300
set god_ [new God]
create-god $val(nn)
 
$ns_ node-config -adhocRouting $val(rp) \
	-llType $val(ll) \
	-macType $val(mac) \
	-ifqType $val(ifq) \
	-ifqLen $val(ifqlen) \
	-antType $val(ant) \
	-propType $val(prop) \
	-phyType $val(netif) \
	-channelType $val(chan) \
	-topoInstance $topo \
	-agentTrace ON \
	-routerTrace ON \
	-macTrace OFF \
	-movementTrace OFF \

for {set i 0} {$i<$val(nn)} {incr i} {
    set node_($i) [$ns_ node]
    $node_($i) random-motion 0
}

source $opt(cp)
source $opt(sc)

for {set i 0} {$i<$val(nn)} {incr i} {
    $ns_ at 300.1 "$node_($i) reset";
}

$ns_ at 300.2 "stop"
$ns_ at 300.3 "puts \"NS EXITING...\"; $ns_ halt"
proc stop {} {
    global ns_ tracefd namtracefd
    $ns_ flush-trace
    close $tracefd
    close $namtracefd
    exec nam aodv.nam & 
    exit 0
}

$ns_ run

```

## 2. 评估

### 2.1 分组投递率

```awk
# getRatio.awk 分组投递率分析
# 初始化设定
BEGIN {
         sendLine = 0;
         recvLine = 0;
         fowardLine = 0;
}
# 应用层收到包
$0 ~/^s.* AGT/ {
        sendLine ++ ;
}
# 应用层发送包
$0 ~/^r.* AGT/ {
         recvLine ++ ;
}
# 路由层转发包 
$0 ~/^f.* RTR/ {
         fowardLine ++ ;
}
# 最后输出结果
END {
         printf "cbr s:%d r:%d, r/s Ratio:%.4f, f:%d \n", sendLine, recvLine, (recvLine/sendLine),fowardLine;
}
 
```

输入命令：

nawk -f getRatio.awk aodv.tr

显示：

cbr s:5012 r:4839, r/s Ratio:0.9655, f:7138

**【本人终端显示为 cbr s:5006 r:4845, r/s Ratio:0.9676, f:8050】**

### 2.2 路由发起频率

```awk
#frequency.awk
BEGIN{
         requests=0;
         frequency=0;
}
{ 
         id=$5;
         source_ip=$57;
         if(($1=="s") && ($61=="REQUEST") && (id==source_ip))
         {requests++;}
}
END{ 
         frequency=requests/300;
         printf("%.4f\n",frequency);
}
```

输入命令：

nawk -f frequency.awk aodv.tr

显示：

0.6767 【本人终端显示为 0.7567】

### 2.3 归一化路由开销

```
#load.awk
BEGIN{
         recvLine=0;
         load=0;
         Normalized_load=0;
}
{
         if(($1=="r") && ($19=="AGT") && ($35=="cbr"))
         {
                   recvLine++;
         }
         if((($1=="s") || ($1=="f")) && ($19=="RTR") && (($35=="AODV")||($35=="message")))
         {
                   load++;
         }
}
END{
         Normalized_load=load/recvLine;
         printf("l:%d,nload:%.4f\n",load,Normalized_load);
}
```

输入命令：

nawk -f load.awk aodv.tr

显示：

l:10360,nload:2.1409 【本人终端显示为 l:11428, nload:2.3587】

### 2.4 平均时延

```
#delay.awk
BEGIN{
         highest_packet_id=0;
         duration_total=0;
}
$0 ~/^r.*AGT/{
         receives++;
}
{
         time=$3;
         packet_id=$41;
         if(($1=="s") && ($19=="AGT") && (start_time[packet_id]==0)) {
                   start_time[packet_id]=time;
                   if(packet_id>highest_packet_id)
                   highest_packet_id=packet_id;
         }
         if(($1=="r") && ($19=="AGT") && (end_tim[packet_id]==0)) {
                   end_time[packet_id]=time;
         }
         if($1=="d"){
                   end_time[packet_id]=-1;
         }
}
END{
         for(packet_id = 0; packet_id <= highest_packet_id; packet_id++) {
                   start=start_time[packet_id];
                   end=end_time[packet_id];
                   if(end != -1 && start < end) {
                   packet_duration=end - start;
                   duration_total = duration_total + packet_duration;
                   }
         }
         printf("%.4f\n",duration_total/receives);
}
 
```

输入命令：

nawk -f delay.awk aodv.tr

显示：

**0.0270  【本人终端显示为 0.0333】**

## 3. 绘制图片并进行性能分析

### 3.1 分组投递率

(1)为了实现对10个和20个CBR业务两种情况分别计算5次分组投递率仿真结果平均值，我们对getRatio.awk作修改，另存为getRatio2.awk，其代码如下：

```
<span style="font-family:Microsoft YaHei;font-size:14px;"># getRatio2.awk 分组投递率分析
# 初始化设定
BEGIN {
         sendLine = 0;
         recvLine = 0;
         fowardLine = 0;
}
# 应用层收到包
$0 ~/^s.* AGT/ {
        sendLine ++ ;
}
# 应用层发送包
$0 ~/^r.* AGT/ {
         recvLine ++ ;
}
# 路由层转发包 
$0 ~/^f.* RTR/ {
         fowardLine ++ ;
}
# 最后输出结果
END {
         #printf "cbr s:%d r:%d, r/s Ratio:%.4f, f:%d \n", sendLine, recvLine, (recvLine/sendLine),fowardLine;
         printf "%d %.4f\n", scr, (recvLine/sendLine) >> outfile;
}</span>
```

(2)shell代码文件run代码如下：

```
#!/bin/sh
#判断以下文件是否存在,如果存在，则将其删除
#$1.1.data
if [ -f $1.1.data ]; then
         rm $1.1.data;
fi
#$1.1.temp
if [ -f $1.1.temp ]; then
         rm $1.1.temp;
fi
#$1.2.data
if [ -f $1.2.data ]; then
         rm $1.2.data;
fi
#$1.2.temp
if [ -f $1.2.temp ]; then
         rm $1.2.temp;
fi
#$1.plot
if [ -f $1.plot ]; then
         rm $1.plot;
fi
 
for j in 0 50 100 200 300
do
         for i in $(seq 1 1 5)
         do
                   setdest -n 50 -p $j -M 20 -t 300 -x 1000 -y 300 > scene-50n-0p-20M-300t-1000-300
 
                   #生成数据流场景1
                   ns cbrgen.tcl -type cbr -nn 50 -seed 1 -mc 10 -rate 2.0 > cbr-50n-10c-2p
                   ns aodv.tcl ; #一次NS运行
                   nawk -v outfile=$1.1.temp -v scr=$i -f getRatio2.awk $1.tr
 
                   #生成数据流场景2
                   ns cbrgen.tcl -type cbr -nn 50 -seed 1 -mc 20 -rate 2.0 > cbr-50n-10c-2p
                   ns aodv.tcl ; #一次NS运行
                   nawk -v outfile=$1.2.temp -v scr=$i -f getRatio2.awk $1.tr
         done
         nawk -v outfile=$1.1.data -v time=$j -f average.awk $1.1.temp
         rm $1.1.temp
 
         nawk -v outfile=$1.2.data -v time=$j -f average.awk $1.2.temp
         rm $1.2.temp
done
 
echo "#'!'/bin/sh" >> $1.plot
echo "set terminal gif small  x255255255" >> $1.plot
echo "set output \"$1.gif\"" >> $1.plot
echo "set ylabel \"Ratio(%)\"" >> $1.plot
echo "set xlabel \"Settle Times(s)\"" >> $1.plot
echo "set key left top box" >> $1.plot
echo "set title \"AODV Ratio result\"" >> $1.plot
echo "plot \"$1.1.data\" title \"$1-cbr 10\" with linespoints, \"$1.2.data\" title \"$1-cbr 20\" with linespoints" >> $1.plot
 
gnuplot $1.plot
```

(3)其中计算平均awk脚本average.awk代码如下：

```
BEGIN{
         average=0.0;
}
{
         if($1=="1"){
                   average=average + $2;
         }
         if($1=="2"){
                   average=average + $2;
         }
         if($1=="3"){
                   average=average + $2;
         }
         if($1=="4"){
                   average=average + $2;
         }
         if($1=="5"){
                   average=average + $2;
         }
}
END {
         printf "%d %.4f \n",time, (average/5) >> outfile;
}
 
```

实验步骤：

在路径~/ns-allinone-2.35/ ns-2.35/aodv/testfile/下输入命令：

sh run aodv

【注意没有.tcl】

其中aodv为运行shell代码文件run的参数。这里要运行很长一段时间。

运行完毕之后，会得到aodv.1.data、aodv.2.data、aodv.jpg与aodv.plot三个文件。aodv.jpg就是我们需要的分组投递率曲线
