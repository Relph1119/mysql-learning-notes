# 第23章 后悔了怎么办-undo日志（下）
&emsp;&emsp;上一章我们主要介绍了为什么需要`undo日志`，以及`INSERT`、`DELETE`、`UPDATE`这些会对数据做改动的语句都会产生什么类型的`undo日志`，还有不同类型的`undo日志`的具体格式是什么。本章会继续介绍这些`undo日志`会被具体写到什么地方，以及在写入过程中需要注意的一些问题。

## 通用链表结构
&emsp;&emsp;在写入`undo日志`的过程中会使用到多个链表，很多链表都有同样的节点结构，如图所示：

![][23-01]

&emsp;&emsp;在某个表空间内，我们可以通过一个页的页号和在页内的偏移量来唯一定位一个节点的位置，这两个信息也就相当于指向这个节点的一个指针。所以：
- `Pre Node Page Number`和`Pre Node Offset`的组合就是指向前一个节点的指针
- `Next Node Page Number`和`Next Node Offset`的组合就是指向后一个节点的指针。

&emsp;&emsp;整个`List Node`占用`12`个字节的存储空间。

&emsp;&emsp;为了更好的管理链表，设计`InnoDB`的大佬还提出了一个基节点的结构，里边存储了这个链表的`头节点`、`尾节点`以及链表长度信息，基节点的结构示意图如下：

![][23-02]

其中：
- `List Length`表明该链表一共有多少节点。
- `First Node Page Number`和`First Node Offset`的组合就是指向链表头节点的指针。
- `Last Node Page Number`和`Last Node Offset`的组合就是指向链表尾节点的指针。

&emsp;&emsp;整个`List Base Node`占用`16`个字节的存储空间。

&emsp;&emsp;所以使用`List Base Node`和`List Node`这两个结构组成的链表的示意图就是这样：

![][23-03]

```
小贴士：上述链表结构我们在前面的文章中频频提到，尤其是在表空间那一章重点描述过，不过我不敢奢求大家都记住了，所以在这里又强调一遍，希望大家不要嫌我烦，我只是怕大家忘了学习后续内容吃力而已～
```

## FIL_PAGE_UNDO_LOG页面
&emsp;&emsp;我们前面介绍表空间的时候说过，表空间其实是由许许多多的页面构成的，页面默认大小为`16KB`。这些页面有不同的类型，比如类型为`FIL_PAGE_INDEX`的页面用于存储聚簇索引以及二级索引，类型为`FIL_PAGE_TYPE_FSP_HDR`的页面用于存储表空间头部信息的，还有其他各种类型的页面，其中有一种称之为`FIL_PAGE_UNDO_LOG`类型的页面是专门用来存储`undo日志`的，这种类型的页面的通用结构如下图所示（以默认的`16KB`大小为例）：

![][23-04]

&emsp;&emsp;“类型为`FIL_PAGE_UNDO_LOG`的页”这种说法太绕口，以后我们就简称为`Undo页面`了。上图中的`File Header`和`File Trailer`是各种页面都有的通用结构，我们前面介绍过很多遍了，这里就不赘述了（忘记了的可以到讲述数据页结构或者表空间的章节中查看）。`Undo Page Header`是`Undo页面`所特有的，我们来看一下它的结构：

![][23-05]

&emsp;&emsp;其中各个属性的意思如下：

- `TRX_UNDO_PAGE_TYPE`：本页面准备存储什么种类的`undo日志`。

    &emsp;&emsp;我们前面介绍了好几种类型的`undo日志`，它们可以被分为两个大类：
    
    - `TRX_UNDO_INSERT`（使用十进制`1`表示）：类型为`TRX_UNDO_INSERT_REC`的`undo日志`属于此大类，一般由`INSERT`语句产生，或者在`UPDATE`语句中有更新主键的情况也会产生此类型的`undo日志`。
    - `TRX_UNDO_UPDATE`（使用十进制`2`表示），除了类型为`TRX_UNDO_INSERT_REC`的`undo日志`，其他类型的`undo日志`都属于这个大类，比如我们前面说的`TRX_UNDO_DEL_MARK_REC`、`TRX_UNDO_UPD_EXIST_REC`什么的，一般由`DELETE`、`UPDATE`语句产生的`undo日志`属于这个大类。
    
    &emsp;&emsp;这个`TRX_UNDO_PAGE_TYPE`属性可选的值就是上面的两个，用来标记本页面用于存储哪个大类的`undo日志`，不同大类的`undo日志`不能混着存储，比如一个`Undo页面`的`TRX_UNDO_PAGE_TYPE`属性值为`TRX_UNDO_INSERT`，那么这个页面就只能存储类型为`TRX_UNDO_INSERT_REC`的`undo日志`，其他类型的`undo日志`就不能放到这个页面中了。
    
    ```
    小贴士：之所以把undo日志分成两个大类，是因为类型为TRX_UNDO_INSERT_REC的undo日志在事务提交后可以直接删除掉，而其他类型的undo日志还需要为所谓的MVCC服务，不能直接删除掉，对它们的处理需要区别对待。当然，如果你看这段话迷迷糊糊的话，那就不需要再看一遍了，现在只需要知道undo日志分为2个大类就好了，更详细的东西我们后边会仔细介绍的。
    ```

- `TRX_UNDO_PAGE_START`：表示在当前页面中是从什么位置开始存储`undo日志`的，或者说表示第一条`undo日志`在本页面中的起始偏移量。
- `TRX_UNDO_PAGE_FREE`：与上面的`TRX_UNDO_PAGE_START`对应，表示当前页面中存储的最后一条`undo`日志结束时的偏移量，或者说从这个位置开始，可以继续写入新的`undo日志`。
    
    &emsp;&emsp;假设现在向页面中写入了3条`undo日志`，那么`TRX_UNDO_PAGE_START`和`TRX_UNDO_PAGE_FREE`的示意图就是这样：

    ![][23-06]

    &emsp;&emsp;当然，在最初一条`undo日志`也没写入的情况下，`TRX_UNDO_PAGE_START`和`TRX_UNDO_PAGE_FREE`的值是相同的。
    
- `TRX_UNDO_PAGE_NODE`：代表一个`List Node`结构（链表的普通节点，我们上面刚说的）。

    &emsp;&emsp;下面马上用到这个属性，稍安勿躁。
    
## Undo页面链表

### 单个事务中的Undo页面链表
&emsp;&emsp;因为一个事务可能包含多个语句，而且一个语句可能对若干条记录进行改动，而对每条记录进行改动前，都需要记录1条或2条的`undo日志`，所以在一个事务执行过程中可能产生很多`undo日志`，这些日志可能一个页面放不下，需要放到多个页面中，这些页面就通过我们上面介绍的`TRX_UNDO_PAGE_NODE`属性连成了链表：

![][23-07]

&emsp;&emsp;大家往上再瞅一瞅上面的图，我们特意把链表中的第一个`Undo页面`给标了出来，称它为`first undo page`，其余的`Undo页面`称之为`normal undo page`，这是因为在`first undo page`中除了记录`Undo Page Header`之外，还会记录其他的一些管理信息，这个我们稍后再说。

&emsp;&emsp;在一个事务执行过程中，可能混着执行`INSERT`、`DELETE`、`UPDATE`语句，也就意味着会产生不同类型的`undo日志`。但是我们前面又强调过，同一个`Undo页面`要么只存储`TRX_UNDO_INSERT`大类的`undo日志`，要么只存储`TRX_UNDO_UPDATE`大类的`undo日志`，反正不能混着存，所以在一个事务执行过程中就可能需要2个`Undo页面`的链表，一个称之为`insert undo链表`，另一个称之为`update undo链表`，画个示意图就是这样：

![][23-08]

&emsp;&emsp;另外，设计`InnoDB`的大佬规定对普通表和临时表的记录改动时产生的`undo日志`要分别记录（我们稍后阐释为什么这么做），所以在一个事务中<span style="color:red">最多</span>有4个以`Undo页面`为节点组成的链表：

![][23-09]

&emsp;&emsp;当然，并不是在事务一开始就会为这个事务分配这4个链表，具体分配策略如下：
- 刚刚开启事务时，一个`Undo页面`链表也不分配。
- 当事务执行过程中向普通表中插入记录或者执行更新记录主键的操作之后，就会为其分配一个`普通表的insert undo链表`。
- 当事务执行过程中删除或者更新了普通表中的记录之后，就会为其分配一个`普通表的update undo链表`。
- 当事务执行过程中向临时表中插入记录或者执行更新记录主键的操作之后，就会为其分配一个`临时表的insert undo链表`。
- 当事务执行过程中删除或者更新了临时表中的记录之后，就会为其分配一个`临时表的update undo链表`。

&emsp;&emsp;总结一句就是：<span style="color:red">按需分配，什么时候需要什么时候再分配，不需要就不分配</span>。

### 多个事务中的Undo页面链表
&emsp;&emsp;为了尽可能提高`undo日志`的写入效率，<span style="color:red">不同事务执行过程中产生的undo日志需要被写入到不同的Undo页面链表中</span>。比方说现在有事务`id`分别为`1`、`2`的两个事务，我们分别称之为`trx 1`和`trx 2`，假设在这两个事务执行过程中：

- `trx 1`对普通表做了`DELETE`操作，对临时表做了`INSERT`和`UPDATE`操作。

    `InnoDB`会为`trx 1`分配3个链表，分别是：
    - 针对普通表的`update undo链表`
    - 针对临时表的`insert undo链表`
    - 针对临时表的`update undo链表`。
    
- `trx 2`对普通表做了`INSERT`、`UPDATE`和`DELETE`操作，没有对临时表做改动。

    `InnoDB`会为`trx 2`分配2个链表，分别是：
    - 针对普通表的`insert undo链表`
    - 针对普通表的`update undo链表`。

&emsp;&emsp;综上所述，在`trx 1`和`trx 2`执行过程中，`InnoDB`共需为这两个事务分配5个`Undo页面`链表，画个图就是这样：

![][23-10]

&emsp;&emsp;如果有更多的事务，那就意味着可能会产生更多的`Undo页面`链表。

## undo日志具体写入过程

### 段（Segment）的概念
&emsp;&emsp;如果你有认真看过表空间那一章的话，对这个`段`的概念应该印象深刻，我们当时花了非常大的篇幅来介绍这个概念。简单讲，这个`段`是一个逻辑上的概念，本质上是由若干个零散页面和若干个完整的区组成的。比如一个`B+`树索引被划分成两个段，一个叶子节点段，一个非叶子节点段，这样叶子节点就可以被尽可能的存到一起，非叶子节点被尽可能的存到一起。每一个段对应一个`INODE Entry`结构，这个`INODE Entry`结构描述了这个段的各种信息，比如段的`ID`，段内的各种链表基节点，零散页面的页号有哪些等信息（具体该结构中每个属性的意思大家可以到表空间那一章里再次重温一下）。我们前面也说过，为了定位一个`INODE Entry`，设计`InnoDB`的大佬设计了一个`Segment Header`的结构：

![][23-11]

&emsp;&emsp;整个`Segment Header`占用10个字节大小，各个属性的意思如下：
- `Space ID of the INODE Entry`：`INODE Entry`结构所在的表空间ID。
- `Page Number of the INODE Entry`：`INODE Entry`结构所在的页面页号。
- `Byte Offset of the INODE Ent`：`INODE Entry`结构在该页面中的偏移量

&emsp;&emsp;知道了表空间ID、页号、页内偏移量，不就可以唯一定位一个`INODE Entry`的地址了么～

```
小贴士：这部分关于段的各种概念我们在表空间那一章中都有详细解释，在这里重提一下只是为了唤醒大家沉睡的记忆，如果有任何不清楚的地方可以再次跳回表空间的那一章仔细读一下。
```

### Undo Log Segment Header
&emsp;&emsp;设计`InnoDB`的大佬规定，每一个`Undo页面`链表都对应着一个`段`，称之为`Undo Log Segment`。也就是说链表中的页面都是从这个段里边申请的，所以他们在`Undo页面`链表的第一个页面，也就是上面提到的`first undo page`中设计了一个称之为`Undo Log Segment Header`的部分，这个部分中包含了该链表对应的段的`segment header`信息以及其他的一些关于这个段的信息，所以`Undo`页面链表的第一个页面其实长这样：

![][23-12]

&emsp;&emsp;可以看到这个`Undo`链表的第一个页面比普通页面多了个`Undo Log Segment Header`，我们来看一下它的结构：

![][23-13]

其中各个属性的意思如下：

- `TRX_UNDO_STATE`：本`Undo页面`链表处在什么状态。

    一个`Undo Log Segment`可能处在的状态包括：
    - `TRX_UNDO_ACTIVE`：活跃状态，也就是一个活跃的事务正在往这个段里边写入`undo日志`。
    - `TRX_UNDO_CACHED`：被缓存的状态。处在该状态的`Undo页面`链表等待着之后被其他事务重用。
    - `TRX_UNDO_TO_FREE`：对于`insert undo`链表来说，如果在它对应的事务提交之后，该链表不能被重用，那么就会处于这种状态。
    - `TRX_UNDO_TO_PURGE`：对于`update undo`链表来说，如果在它对应的事务提交之后，该链表不能被重用，那么就会处于这种状态。
    - `TRX_UNDO_PREPARED`：包含处于`PREPARE`阶段的事务产生的`undo日志`。
    
    ```
    小贴士：Undo页面链表什么时候会被重用，怎么重用我们之后会详细说的。事务的PREPARE阶段是在所谓的分布式事务中才出现的，本书中不会介绍更多关于分布式事务的事情，所以大家目前忽略这个状态就好了。
    ```
    
- `TRX_UNDO_LAST_LOG`：本`Undo页面`链表中最后一个`Undo Log Header`的位置。

    ```
    小贴士：关于什么是Undo Log Header，我们稍后马上介绍。
    ```
    
- `TRX_UNDO_FSEG_HEADER`：本`Undo页面`链表对应的段的`Segment Header`信息（就是我们上一节介绍的那个10字节结构，通过这个信息可以找到该段对应的`INODE Entry`）。
- `TRX_UNDO_PAGE_LIST`：`Undo页面`链表的基节点。

    &emsp;&emsp;我们上面说`Undo页面`的`Undo Page Header`部分有一个12字节大小的`TRX_UNDO_PAGE_NODE`属性，这个属性代表一个`List Node`结构。每一个`Undo页面`都包含`Undo Page Header`结构，这些页面就可以通过这个属性连成一个链表。这个`TRX_UNDO_PAGE_LIST`属性代表着这个链表的基节点，当然这个基节点只存在于`Undo页面`链表的第一个页面，也就是`first undo page`中。

### Undo Log Header
&emsp;&emsp;一个事务在向`Undo页面`中写入`undo日志`时的方式是十分简单暴力的，就是直接往里怼，写完一条紧接着写另一条，各条`undo日志`之间是亲密无间的。写完一个`Undo页面`后，再从段里申请一个新页面，然后把这个页面插入到`Undo页面`链表中，继续往这个新申请的页面中写。设计`InnoDB`的大佬认为同一个事务向一个`Undo页面`链表中写入的`undo日志`算是一个组，比方说我们上面介绍的`trx 1`由于会分配3个`Undo页面`链表，也就会写入3个组的`undo日志`；`trx 2`由于会分配2个`Undo页面`链表，也就会写入2个组的`undo日志`。在每写入一组`undo日志`时，都会在这组`undo日志`前先记录一下关于这个组的一些属性，设计`InnoDB`的大佬把存储这些属性的地方称之为`Undo Log Header`。所以`Undo页面`链表的第一个页面在真正写入`undo日志`前，其实都会被填充`Undo Page Header`、`Undo Log Segment Header`、`Undo Log Header`这3个部分，如图所示：

![][23-14]

这个`Undo Log Header`具体的结构如下：

![][23-15]

&emsp;&emsp;哇唔，映入眼帘的又是一大坨属性，我们先大致看一下它们都是什么意思：
- `TRX_UNDO_TRX_ID`：生成本组`undo日志`的事务`id`。
- `TRX_UNDO_TRX_NO`：事务提交后生成的一个需要序号，使用此序号来标记事务的提交顺序（先提交的此序号小，后提交的此序号大）。
- `TRX_UNDO_DEL_MARKS`：标记本组`undo`日志中是否包含由于`Delete mark`操作产生的`undo日志`。
- `TRX_UNDO_LOG_START`：表示本组`undo`日志中第一条`undo日志`的在页面中的偏移量。
- `TRX_UNDO_XID_EXISTS`：本组`undo日志`是否包含XID信息。

    ```
    小贴士：本书不会讲述更多关于XID是个什么东东，有兴趣的同学可以到搜索引擎或者文档中搜一搜。
    ```
    
- `TRX_UNDO_DICT_TRANS`：标记本组`undo日志`是不是由DDL语句产生的。
- `TRX_UNDO_TABLE_ID`：如果`TRX_UNDO_DICT_TRANS`为真，那么本属性表示DDL语句操作的表的`table id`。
- `TRX_UNDO_NEXT_LOG`：下一组的`undo日志`在页面中开始的偏移量。
- `TRX_UNDO_PREV_LOG`：上一组的`undo日志`在页面中开始的偏移量。

    ```
    小贴士：一般来说一个Undo页面链表只存储一个事务执行过程中产生的一组undo日志，但是在某些情况下，可能会在一个事务提交之后，之后开启的事务重复利用这个Undo页面链表，这样就会导致一个Undo页面中可能存放多组Undo日志，TRX_UNDO_NEXT_LOG和TRX_UNDO_PREV_LOG就是用来标记下一组和上一组undo日志在页面中的偏移量的。关于什么时候重用Undo页面链表，怎么重用这个链表我们稍后会详细说明的，现在先理解TRX_UNDO_NEXT_LOG和TRX_UNDO_PREV_LOG这两个属性的意思就好了。
    ```

- `TRX_UNDO_HISTORY_NODE`：一个12字节的`List Node`结构，代表一个称之为`History`链表的节点。

    ```
    小贴士：关于History链表我们后边会格外详细的介绍，现在先不用管。
    ```

### 小结
&emsp;&emsp;对于没有被重用的`Undo页面`链表来说，链表的第一个页面，也就是`first undo page`在真正写入`undo日志`前，会填充`Undo Page Header`、`Undo Log Segment Header`、`Undo Log Header`这3个部分，之后才开始正式写入`undo日志`。对于其他的页面来说，也就是`normal undo page`在真正写入`undo日志`前，只会填充`Undo Page Header`。链表的`List Base Node`存放到`first undo page`的`Undo Log Segment Header`部分，`List Node`信息存放到每一个`Undo页面`的`undo Page Header`部分，所以画一个`Undo页面`链表的示意图就是这样：

![][23-16]

## 重用Undo页面
&emsp;&emsp;我们前面说为了能提高并发执行的多个事务写入`undo日志`的性能，设计`InnoDB`的大佬决定为每个事务单独分配相应的`Undo页面`链表（最多可能单独分配4个链表）。但是这样也造成了一些问题，比如其实大部分事务执行过程中可能只修改了一条或几条记录，针对某个`Undo页面`链表只产生了非常少的`undo日志`，这些`undo日志`可能只占用一丢丢存储空间，每开启一个事务就新创建一个`Undo页面`链表（虽然这个链表中只有一个页面）来存储这么一丢丢`undo日志`岂不是太浪费了么？的确是挺浪费，于是设计`InnoDB`的大佬本着勤俭节约的优良传统，决定在事务提交后在某些情况下重用该事务的`Undo页面`链表。一个`Undo页面`链表是否可以被重用的条件很简单：

- 该链表中只包含一个`Undo页面`。

    &emsp;&emsp;如果一个事务执行过程中产生了非常多的`undo日志`，那么它可能申请非常多的页面加入到`Undo页面`链表中。在该事物提交后，如果将整个链表中的页面都重用，那就意味着即使新的事务并没有向该`Undo页面`链表中写入很多`undo日志`，那该链表中也得维护非常多的页面，那些用不到的页面也不能被别的事务所使用，这样就造成了另一种浪费。所以设计`InnoDB`的大佬们规定，只有在`Undo页面`链表中只包含一个`Undo页面`时，该链表才可以被下一个事务所重用。
    
- 该`Undo页面`已经使用的空间小于整个页面空间的3/4。

&emsp;&emsp;我们前面说过，`Undo页面`链表按照存储的`undo日志`所属的大类可以被分为`insert undo链表`和`update undo链表`两种，这两种链表在被重用时的策略也是不同的，我们分别看一下：

- insert undo链表

    &emsp;&emsp;`insert undo链表`中只存储类型为`TRX_UNDO_INSERT_REC`的`undo日志`，这种类型的`undo日志`在事务提交之后就没用了，就可以被清除掉。所以在某个事务提交后，重用这个事务的`insert undo链表`（这个链表中只有一个页面）时，可以直接把之前事务写入的一组`undo日志`覆盖掉，从头开始写入新事务的一组`undo日志`，如下图所示：

    ![][23-17]
    
    &emsp;&emsp;如图所示，假设有一个事务使用的`insert undo链表`，到事务提交时，只向`insert undo链表`中插入了3条`undo日志`，这个`insert undo链表`只申请了一个`Undo页面`。假设此刻该页面已使用的空间小于整个页面大小的3/4，那么下一个事务就可以重用这个`insert undo链表`（链表中只有一个页面）。假设此时有一个新事务重用了该`insert undo链表`，那么可以直接把旧的一组`undo日志`覆盖掉，写入一组新的`undo日志`。
    
    ```
    小贴士：当然，在重用Undo页面链表写入新的一组undo日志时，不仅会写入新的Undo Log Header，还会适当调整Undo Page Header、Undo Log Segment Header、Undo Log Header中的一些属性，比如TRX_UNDO_PAGE_START、TRX_UNDO_PAGE_FREE等等等等，这些我们就不具体介绍了。
    ```
    
- update undo链表

    &emsp;&emsp;在一个事务提交后，它的`update undo链表`中的`undo日志`也不能立即删除掉（这些日志用于MVCC，我们后边会说的）。所以如果之后的事务想重用`update undo链表`时，就不能覆盖之前事务写入的`undo日志`。这样就相当于在同一个`Undo页面`中写入了多组的`undo日志`，效果看起来就是这样：
    
    ![][23-18]


## 回滚段

### 回滚段的概念
&emsp;&emsp;我们现在知道一个事务在执行过程中最多可以分配4个`Undo页面`链表，在同一时刻不同事务拥有的`Undo页面`链表是不一样的，所以在同一时刻系统里其实可以有许许多多个`Undo页面`链表存在。为了更好的管理这些链表，设计`InnoDB`的大佬又设计了一个称之为`Rollback Segment Header`的页面，在这个页面中存放了各个`Undo页面`链表的`frist undo page`的`页号`，他们把这些`页号`称之为`undo slot`。我们可以这样理解，每个`Undo页面`链表都相当于是一个班，这个链表的`first undo page`就相当于这个班的班长，找到了这个班的班长，就可以找到班里的其他同学（其他同学相当于`normal undo page`）。有时候学校需要向这些班级传达一下精神，就需要把班长都召集在会议室，这个`Rollback Segment Header`就相当于是一个会议室。

&emsp;&emsp;我们看一下这个称之为`Rollback Segment Header`的页面长什么样（以默认的16KB为例）：

![][23-19]

&emsp;&emsp;设计`InnoDB`的大佬规定，每一个`Rollback Segment Header`页面都对应着一个段，这个段就称为`Rollback Segment`，翻译过来就是`回滚段`。与我们之前介绍的各种段不同的是，这个`Rollback Segment`里其实只有一个页面（这可能是设计`InnoDB`的大佬们的一种洁癖，他们可能觉得为了某个目的去分配页面的话都得先申请一个段，或者他们觉得虽然目前版本的`MySQL`里`Rollback Segment`里其实只有一个页面，但可能之后的版本里会增加页面也说不定）。

&emsp;&emsp;了解了`Rollback Segment`的含义之后，我们再来看看这个称之为`Rollback Segment Header`的页面的各个部分的含义都是什么意思：

- `TRX_RSEG_MAX_SIZE`：本`Rollback Segment`中管理的所有`Undo页面`链表中的`Undo页面`数量之和的最大值。换句话说，本`Rollback Segment`中所有`Undo页面`链表中的`Undo页面`数量之和不能超过`TRX_RSEG_MAX_SIZE`代表的值。

    &emsp;&emsp;该属性的值默认为无限大，也就是我们想写多少`Undo页面`都可以。
    ```
    小贴士：无限大其实也只是个夸张的说法，4个字节能表示最大的数也就是0xFFFFFFFF，但是我们之后会看到，0xFFFFFFFF这个数有特殊用途，所以实际上TRX_RSEG_MAX_SIZE的值为0xFFFFFFFE。
    ```
    
- `TRX_RSEG_HISTORY_SIZE`：`History`链表占用的页面数量。
- `TRX_RSEG_HISTORY`：`History`链表的基节点。

    ```
    小贴士：History链表后边讲，稍安勿躁。
    ```

- `TRX_RSEG_FSEG_HEADER`：本`Rollback Segment`对应的10字节大小的`Segment Header`结构，通过它可以找到本段对应的`INODE Entry`。
- `TRX_RSEG_UNDO_SLOTS`：各个`Undo页面`链表的`first undo page`的`页号`集合，也就是`undo slot`集合。

    一个页号占用`4`个字节，对于`16KB`大小的页面来说，这个`TRX_RSEG_UNDO_SLOTS`部分共存储了`1024`个`undo slot`，所以共需`1024 × 4 = 4096`个字节。

### 从回滚段中申请Undo页面链表
&emsp;&emsp;初始情况下，由于未向任何事务分配任何`Undo页面`链表，所以对于一个`Rollback Segment Header`页面来说，它的各个`undo slot`都被设置成了一个特殊的值：`FIL_NULL`（对应的十六进制就是`0xFFFFFFFF`），表示该`undo slot`不指向任何页面。

&emsp;&emsp;随着时间的流逝，开始有事务需要分配`Undo页面`链表了，就从回滚段的第一个`undo slot`开始，看看该`undo slot`的值是不是`FIL_NULL`：

- 如果是`FIL_NULL`，那么在表空间中新创建一个段（也就是`Undo Log Segment`），然后从段里申请一个页面作为`Undo页面`链表的`first undo page`，然后把该`undo slot`的值设置为刚刚申请的这个页面的地址，这样也就意味着这个`undo slot`被分配给了这个事务。
- 如果不是`FIL_NULL`，说明该`undo slot`已经指向了一个`undo链表`，也就是说这个`undo slot`已经被别的事务占用了，那就跳到下一个`undo slot`，判断该`undo slot`的值是不是`FIL_NULL`，重复上面的步骤。

&emsp;&emsp;一个`Rollback Segment Header`页面中包含`1024`个`undo slot`，如果这`1024`个`undo slot`的值都不为`FIL_NULL`，这就意味着这`1024`个`undo slot`都已经名花有主（被分配给了某个事务），此时由于新事务无法再获得新的`Undo页面`链表，就会回滚这个事务并且给用户报错：
```
Too many active concurrent transactions
```
&emsp;&emsp;用户看到这个错误，可以选择重新执行这个事务（可能重新执行时有别的事务提交了，该事务就可以被分配`Undo页面`链表了）。

&emsp;&emsp;当一个事务提交时，它所占用的`undo slot`有两种命运：

- 如果该`undo slot`指向的`Undo页面`链表符合被重用的条件（就是我们上面说的`Undo页面`链表只占用一个页面并且已使用空间小于整个页面的3/4）。
    
    &emsp;&emsp;该`undo slot`就处于被缓存的状态，设计`InnoDB`的大佬规定这时该`Undo页面`链表的`TRX_UNDO_STATE`属性（该属性在`first undo page`的`Undo Log Segment Header`部分）会被设置为`TRX_UNDO_CACHED`。

    &emsp;&emsp;被缓存的`undo slot`都会被加入到一个链表，根据对应的`Undo页面`链表的类型不同，也会被加入到不同的链表：
    
    - 如果对应的`Undo页面`链表是`insert undo链表`，则该`undo slot`会被加入`insert undo cached链表`。
    - 如果对应的`Undo页面`链表是`update undo链表`，则该`undo slot`会被加入`update undo cached链表`。
    
    &emsp;&emsp;一个回滚段就对应着上述两个`cached链表`，如果有新事务要分配`undo slot`时，先从对应的`cached链表`中找。如果没有被缓存的`undo slot`，才会到回滚段的`Rollback Segment Header`页面中再去找。

- 如果该`undo slot`指向的`Undo页面`链表不符合被重用的条件，那么针对该`undo slot`对应的`Undo页面`链表类型不同，也会有不同的处理：

    - 如果对应的`Undo页面`链表是`insert undo链表`，则该`Undo页面`链表的`TRX_UNDO_STATE`属性会被设置为`TRX_UNDO_TO_FREE`，之后该`Undo页面`链表对应的段会被释放掉（也就意味着段中的页面可以被挪作他用），然后把该`undo slot`的值设置为`FIL_NULL`。
    - 如果对应的`Undo页面`链表是`update undo链表`，则该`Undo页面`链表的`TRX_UNDO_STATE`属性会被设置为`TRX_UNDO_TO_PRUGE`，则会将该`undo slot`的值设置为`FIL_NULL`，然后将本次事务写入的一组`undo`日志放到所谓的`History链表`中（需要注意的是，这里并不会将`Undo页面`链表对应的段给释放掉，因为这些`undo`日志还有用呢～）。
    
    ```
    小贴士：更多关于History链表的事我们稍后再说，稍安勿躁。
    ```
    
### 多个回滚段
&emsp;&emsp;我们说一个事务执行过程中最多分配`4`个`Undo页面`链表，而一个回滚段里只有`1024`个`undo slot`，很显然`undo slot`的数量有点少啊。我们即使假设一个读写事务执行过程中只分配`1`个`Undo页面`链表，那`1024`个`undo slot`也只能支持`1024`个读写事务同时执行，再多了就崩溃了。这就相当于会议室只能容下1024个班长同时开会，如果有几千人同时到会议室开会的话，那后来的那些班长就没地方坐了，只能等待前面的人开完会自己再进去开。

&emsp;&emsp;话说在`InnoDB`的早期发展阶段的确只有一个回滚段，但是设计`InnoDB`的大佬后来意识到了这个问题，咋解决这问题呢？会议室不够，多盖几个会议室不就得了。所以设计`InnoDB`的大佬一口气定义了`128`个回滚段，也就相当于有了`128 × 1024 = 131072`个`undo slot`。假设一个读写事务执行过程中只分配`1`个`Undo页面`链表，那么就可以同时支持`131072`个读写事务并发执行（这么多事务在一台机器上并发执行，还真没见过呢～）。
```
小贴士：只读事务并不需要分配Undo页面链表，MySQL 5.7中所有刚开启的事务默认都是只读事务，只有在事务执行过程中对记录做了某些改动时才会被升级为读写事务。
```
&emsp;&emsp;每个回滚段都对应着一个`Rollback Segment Header`页面，有128个回滚段，自然就要有128个`Rollback Segment Header`页面，这些页面的地址总得找个地方存一下吧！于是设计`InnoDB`的大佬在系统表空间的第`5`号页面的某个区域包含了128个8字节大小的格子： 

![][23-20]

&emsp;&emsp;每个8字节的格子的构造就像这样：

![][23-21]
    
&emsp;&emsp;如果所示，每个8字节的格子其实由两部分组成：

- 4字节大小的`Space ID`，代表一个表空间的ID。
- 4字节大小的`Page number`，代表一个页号。
    
&emsp;&emsp;也就是说每个8字节大小的`格子`相当于一个指针，指向某个表空间中的某个页面，这些页面就是`Rollback Segment Header`。这里需要注意的一点事，要定位一个`Rollback Segment Header`还需要知道对应的表空间ID，这也就意味着<span style="color:red">不同的回滚段可能分布在不同的表空间中</span>。

&emsp;&emsp;所以通过上面的叙述我们可以大致清楚，在系统表空间的第`5`号页面中存储了128个`Rollback Segment Header`页面地址，每个`Rollback Segment Header`就相当于一个回滚段。在`Rollback Segment Header`页面中，又包含`1024`个`undo slot`，每个`undo slot`都对应一个`Undo页面`链表。我们画个示意图：

![][23-22]

把图一画出来就清爽多了。

### 回滚段的分类
&emsp;&emsp;我们把这128个回滚段给编一下号，最开始的回滚段称之为`第0号回滚段`，之后依次递增，最后一个回滚段就称之为`第127号回滚段`。这128个回滚段可以被分成两大类：

- 第`0`号、第`33～127`号回滚段属于一类。其中第`0`号回滚段必须在系统表空间中（就是说第`0`号回滚段对应的`Rollback Segment Header`页面必须在系统表空间中），第`33～127`号回滚段既可以在系统表空间中，也可以在自己配置的`undo`表空间中，关于怎么配置我们稍后再说。
    
    &emsp;&emsp;如果一个事务在执行过程中由于对普通表的记录做了改动需要分配`Undo页面`链表时，必须从这一类的段中分配相应的`undo slot`。

- 第`1～32`号回滚段属于一类。这些回滚段必须在临时表空间（对应着数据目录中的`ibtmp1`文件）中。

    &emsp;&emsp;如果一个事务在执行过程中由于对临时表的记录做了改动需要分配`Undo页面`链表时，必须从这一类的段中分配相应的`undo slot`。

&emsp;&emsp;也就是说如果一个事务在执行过程中既对普通表的记录做了改动，又对临时表的记录做了改动，那么需要为这个记录分配2个回滚段，再分别到这两个回滚段中分配对应的`undo slot`。

&emsp;&emsp;不知道大家有没有疑惑，为什么要把针对普通表和临时表来划分不同种类的`回滚段`呢？这个还得从`Undo页面`本身说起，我们说`Undo页面`其实是类型为`FIL_PAGE_UNDO_LOG`的页面的简称，说到底它也是一个普通的页面。我们前面说过，在修改页面之前一定要先把对应的`redo日志`写上，这样在系统奔溃重启时才能恢复到奔溃前的状态。我们向`Undo页面`写入`undo日志`本身也是一个写页面的过程，设计`InnoDB`的大佬为此还设计了许多种`redo日志`的类型，比方说`MLOG_UNDO_HDR_CREATE`、`MLOG_UNDO_INSERT`、`MLOG_UNDO_INIT`等等等等，也就是说我们对`Undo页面`做的任何改动都会记录相应类型的`redo日志`。但是对于临时表来说，因为修改临时表而产生的`undo日志`只需要在系统运行过程中有效，如果系统奔溃了，那么在重启时也不需要恢复这些`undo`日志所在的页面，所以在写针对临时表的`Undo页面`时，并不需要记录相应的`redo日志`。总结一下针对普通表和临时表划分不同种类的`回滚段`的原因：<span style="color:red">在修改针对普通表的回滚段中的Undo页面时，需要记录对应的redo日志，而修改针对临时表的回滚段中的Undo页面时，不需要记录对应的redo日志</span>。

```
小贴士：实际上在MySQL 5.7.21这个版本中，如果我们仅仅对普通表的记录做了改动，那么只会为该事务分配针对普通表的回滚段，不分配针对临时表的回滚段。但是如果我们仅仅对临时表的记录做了改动，那么既会为该事务分配针对普通表的回滚段，又会为其分配针对临时表的回滚段（不过分配了回滚段并不会立即分配undo slot，只有在真正需要Undo页面链表时才会去分配回滚段中的undo slot）。
```
### 为事务分配Undo页面链表详细过程
&emsp;&emsp;上面说了一大堆的概念，大家应该有一点点的小晕，接下来我们以事务对普通表的记录做改动为例，给大家梳理一下事务执行过程中分配`Undo页面`链表时的完整过程，

- 事务在执行过程中对普通表的记录首次做改动之前，首先会到系统表空间的第`5`号页面中分配一个回滚段（其实就是获取一个`Rollback Segment Header`页面的地址）。一旦某个回滚段被分配给了这个事务，那么之后该事务中再对普通表的记录做改动时，就不会重复分配了。
    
    &emsp;&emsp;使用传说中的`round-robin`（循环使用）方式来分配回滚段。比如当前事务分配了第`0`号回滚段，那么下一个事务就要分配第`33`号回滚段，下下个事务就要分配第`34`号回滚段，简单一点的说就是这些回滚段被轮着分配给不同的事务（就是这么简单粗暴，没什么好说的）。
    
- 在分配到回滚段后，首先看一下这个回滚段的两个`cached链表`有没有已经缓存了的`undo slot`，比如如果事务做的是`INSERT`操作，就去回滚段对应的`insert undo cached链表`中看看有没有缓存的`undo slot`；如果事务做的是`DELETE`操作，就去回滚段对应的`update undo cached链表`中看看有没有缓存的`undo slot`。如果有缓存的`undo slot`，那么就把这个缓存的`undo slot`分配给该事务。
- 如果没有缓存的`undo slot`可供分配，那么就要到`Rollback Segment Header`页面中找一个可用的`undo slot`分配给当前事务。

    &emsp;&emsp;从`Rollback Segment Header`页面中分配可用的`undo slot`的方式我们上面也说过了，就是从第`0`个`undo slot`开始，如果该`undo slot`的值为`FIL_NULL`，意味着这个`undo slot`是空闲的，就把这个`undo slot`分配给当前事务，否则查看第`1`个`undo slot`是否满足条件，依次类推，直到最后一个`undo slot`。如果这`1024`个`undo slot`都没有值为`FIL_NULL`的情况，就直接报错喽（一般不会出现这种情况）～
    
- 找到可用的`undo slot`后，如果该`undo slot`是从`cached链表`中获取的，那么它对应的`Undo Log Segment`已经分配了，否则的话需要重新分配一个`Undo Log Segment`，然后从该`Undo Log Segment`中申请一个页面作为`Undo页面`链表的`first undo page`。
- 然后事务就可以把`undo日志`写入到上面申请的`Undo页面`链表了！

&emsp;&emsp;对临时表的记录做改动的步骤和上述的一样，就不赘述了。不错需要再次强调一次，<span style="color:red">如果一个事务在执行过程中既对普通表的记录做了改动，又对临时表的记录做了改动，那么需要为这个记录分配2个回滚段。并发执行的不同事务其实也可以被分配相同的回滚段，只要分配不同的undo slot就可以了</span>。

## 回滚段相关配置

### 配置回滚段数量
&emsp;&emsp;我们前面说系统中一共有`128`个回滚段，其实这只是默认值，我们可以通过启动参数`innodb_rollback_segments`来配置回滚段的数量，可配置的范围是`1~128`。但是这个参数并不会影响针对临时表的回滚段数量，针对临时表的回滚段数量一直是`32`，也就是说：
- 如果我们把`innodb_rollback_segments`的值设置为`1`，那么只会有1个针对普通表的可用回滚段，但是仍然有32个针对临时表的可用回滚段。
- 如果我们把`innodb_rollback_segments`的值设置为`2～33`之间的数，效果和将其设置为`1`是一样的。
- 如果我们把`innodb_rollback_segments`设置为大于`33`的数，那么针对普通表的可用回滚段数量就是该值减去32。

### 配置undo表空间
&emsp;&emsp;默认情况下，针对普通表设立的回滚段（第`0`号以及第`33~127`号回滚段）都是被分配到系统表空间的。其中的第第`0`号回滚段是一直在系统表空间的，但是第`33~127`号回滚段可以通过配置放到自定义的`undo表空间`中。但是这种配置只能在系统初始化（创建数据目录时）的时候使用，一旦初始化完成，之后就不能再次更改了。我们看一下相关启动参数：
- 通过`innodb_undo_directory`指定`undo表空间`所在的目录，如果没有指定该参数，则默认`undo表空间`所在的目录就是数据目录。
- 通过`innodb_undo_tablespaces`定义`undo表空间`的数量。该参数的默认值为`0`，表明不创建任何`undo表空间`。

    &emsp;&emsp;第`33~127`号回滚段可以平均分布到不同的`undo表空间`中。

```
小贴士：如果我们在系统初始化的时候指定了创建了undo表空间，那么系统表空间中的第0号回滚段将处于不可用状态。
```
&emsp;&emsp;比如我们在系统初始化时指定的`innodb_rollback_segments`为`35`，`innodb_undo_tablespaces`为`2`，这样就会将第`33`、`34`号回滚段分别分布到一个`undo表空间`中。

&emsp;&emsp;设立`undo表空间`的一个好处就是在`undo表空间`中的文件大到一定程度时，可以自动的将该`undo表空间`截断（truncate）成一个小文件。而系统表空间的大小只能不断的增大，却不能截断。

  [23-01]: ../images/23-01.png
  [23-02]: ../images/23-02.png
  [23-03]: ../images/23-03.png
  [23-04]: ../images/23-04.png
  [23-05]: ../images/23-05.png
  [23-06]: ../images/23-06.png
  [23-07]: ../images/23-07.png
  [23-08]: ../images/23-08.png
  [23-09]: ../images/23-09.png
  [23-10]: ../images/23-10.png
  [23-11]: ../images/23-11.png
  [23-12]: ../images/23-12.png
  [23-13]: ../images/23-13.png
  [23-14]: ../images/23-14.png
  [23-15]: ../images/23-15.png
  [23-16]: ../images/23-16.png
  [23-17]: ../images/23-17.png
  [23-18]: ../images/23-18.png
  [23-19]: ../images/23-19.png
  [23-20]: ../images/23-20.png
  [23-21]: ../images/23-21.png
  [23-22]: ../images/23-22.png
  
<div STYLE="page-break-after: always;"></div>

