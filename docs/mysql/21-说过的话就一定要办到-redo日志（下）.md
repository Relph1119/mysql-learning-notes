# 第21章 说过的话就一定要办到-redo日志（下）

## redo日志文件

### redo日志刷盘时机
&emsp;&emsp;我们前面说`mtr`运行过程中产生的一组`redo`日志在`mtr`结束时会被复制到`log buffer`中，可是这些日志总在内存里呆着也不是个办法，在一些情况下它们会被刷新到磁盘里，比如：

- `log buffer`空间不足时

    &emsp;&emsp;`log buffer`的大小是有限的（通过系统变量`innodb_log_buffer_size`指定），如果不停的往这个有限大小的`log buffer`里塞入日志，很快它就会被填满。设计`InnoDB`的大佬认为如果当前写入`log buffer`的`redo`日志量已经占满了`log buffer`总容量的大约一半左右，就需要把这些日志刷新到磁盘上。
    
- 事务提交时
    
    &emsp;&emsp;我们前面说过之所以使用`redo`日志主要是因为它占用的空间少，还是顺序写，在事务提交时可以不把修改过的`Buffer Pool`页面刷新到磁盘，但是为了保证持久性，必须要把修改这些页面对应的`redo`日志刷新到磁盘。

- 后台线程不停的刷刷刷

    &emsp;&emsp;后台有一个线程，大约每秒都会刷新一次`log buffer`中的`redo`日志到磁盘。

- 正常关闭服务器时
- 做所谓的`checkpoint`时（我们现在没介绍过`checkpoint`的概念，稍后会仔细介绍，稍安勿躁）    
- 其他的一些情况...

### redo日志文件组
&emsp;&emsp;`MySQL`的数据目录（使用`SHOW VARIABLES LIKE 'datadir'`查看）下默认有两个名为`ib_logfile0`和`ib_logfile1`的文件，`log buffer`中的日志默认情况下就是刷新到这两个磁盘文件中。如果我们对默认的`redo`日志文件不满意，可以通过下面几个启动参数来调节：

- `innodb_log_group_home_dir`

    &emsp;&emsp;该参数指定了`redo`日志文件所在的目录，默认值就是当前的数据目录。
    
- `innodb_log_file_size`

    &emsp;&emsp;该参数指定了每个`redo`日志文件的大小，在`MySQL 5.7.21`这个版本中的默认值为`48MB`，

- `innodb_log_files_in_group`

    &emsp;&emsp;该参数指定`redo`日志文件的个数，默认值为2，最大值为100。

&emsp;&emsp;从上面的描述中可以看到，磁盘上的`redo`日志文件不只一个，而是以一个`日志文件组`的形式出现的。这些文件以`ib_logfile[数字]`（`数字`可以是`0`、`1`、`2`...）的形式进行命名。在将`redo`日志写入`日志文件组`时，是从`ib_logfile0`开始写，如果`ib_logfile0`写满了，就接着`ib_logfile1`写，同理，`ib_logfile1`写满了就去写`ib_logfile2`，依此类推。如果写到最后一个文件该咋办？那就重新转到`ib_logfile0`继续写，所以整个过程如下图所示：

![][21-01]

&emsp;&emsp;总共的`redo`日志文件大小其实就是：`innodb_log_file_size × innodb_log_files_in_group`。
```
小贴士：如果采用循环使用的方式向redo日志文件组里写数据的话，那岂不是要追尾，也就是后写入的redo日志覆盖掉前面写的redo日志？当然可能了！所以设计InnoDB的大佬提出了checkpoint的概念，稍后我们重点介绍～
```

### redo日志文件格式
&emsp;&emsp;我们前面说过`log buffer`本质上是一片连续的内存空间，被划分成了若干个`512`字节大小的`block`。<span style="color:red">将log buffer中的redo日志刷新到磁盘的本质就是把block的镜像写入日志文件中</span>，所以`redo`日志文件其实也是由若干个`512`字节大小的block组成。

`redo`日志文件组中的每个文件大小都一样，格式也一样，都是由两部分组成：

- 前2048个字节，也就是前4个block是用来存储一些管理信息的。
- 从第2048字节往后是用来存储`log buffer`中的block镜像的。

&emsp;&emsp;所以我们前面所说的`循环`使用redo日志文件，其实是从每个日志文件的第2048个字节开始算，画个示意图就是这样：

![][21-02]

&emsp;&emsp;普通block的格式我们在介绍`log buffer`的时候都说过了，就是`log block header`、`log block body`、`log block trialer`这三个部分，就不重复介绍了。这里需要介绍一下每个`redo`日志文件前2048个字节，也就是前4个特殊block的格式都是干嘛的，废话少说，先看图：

![][21-03]    
&emsp;&emsp;从图中可以看出来，这4个block分别是：

- `log file header`：描述该`redo`日志文件的一些整体属性，看一下它的结构：

    ![][21-04]
    
    &emsp;&emsp;各个属性的具体释义如下：
    
    |属性名|长度（单位：字节）|描述|
    |:--:|:--:|:--|
    |`LOG_HEADER_FORMAT`|`4`|`redo`日志的版本，在`MySQL 5.7.21`中该值永远为1|
    |`LOG_HEADER_PAD1`|`4`|做字节填充用的，没什么实际意义，忽略～|
    |`LOG_HEADER_START_LSN`|`8`|标记本`redo`日志文件开始的LSN值，也就是文件偏移量为2048字节初对应的LSN值（关于什么是LSN我们稍后再看，看不懂的先忽略）。|
    |`LOG_HEADER_CREATOR`|`32`|一个字符串，标记本`redo`日志文件的创建者是谁。正常运行时该值为`MySQL`的版本号，比如：`"MySQL 5.7.21"`，使用`mysqlbackup`命令创建的`redo`日志文件的该值为`"ibbackup"`和创建时间。|
    |`LOG_BLOCK_CHECKSUM`|`4`|本block的校验值，所有block都有，我们不关心|
    
    ```
    小贴士：设计InnoDB的大佬对redo日志的block格式做了很多次修改，如果你阅读的其他书籍中发现上述的属性和你阅读书籍中的属性有些出入，不要慌，正常现象，忘记以前的版本吧。另外，LSN值我们后边才会介绍，现在千万别纠结LSN是什么。
    ```
    
- `checkpoint1`：记录关于`checkpoint`的一些属性，看一下它的结构：

    ![][21-05]

    各个属性的具体释义如下：
    
    |属性名|长度（单位：字节）|描述|
    |:--:|:--:|:--|
    |`LOG_CHECKPOINT_NO`|`8`|服务器做`checkpoint`的编号，每做一次`checkpoint`，该值就加1。|
    |`LOG_CHECKPOINT_LSN`|`8`|服务器做`checkpoint`结束时对应的`LSN`值，系统奔溃恢复时将从该值开始。|
    |`LOG_CHECKPOINT_OFFSET`|`8`|上个属性中的`LSN`值在`redo`日志文件组中的偏移量|
    |`LOG_CHECKPOINT_LOG_BUF_SIZE`|`8`|服务器在做`checkpoint`操作时对应的`log buffer`的大小|
    |`LOG_BLOCK_CHECKSUM`|`4`|本block的校验值，所有block都有，我们不关心|

    ```
    小贴士：现在看不懂上面这些关于checkpoint和LSN的属性的释义是很正常的，我就是想让大家对上面这些属性混个脸熟，后边我们后详细介绍的。
    ```

- 第三个block未使用，忽略～

- `checkpoint2`：结构和`checkpoint1`一样。

## Log Sequeue Number
&emsp;&emsp;自系统开始运行，就不断的在修改页面，也就意味着会不断的生成`redo`日志。`redo`日志的量在不断的递增，就像人的年龄一样，自打出生起就不断递增，永远不可能缩减了。设计`InnoDB`的大佬为记录已经写入的`redo`日志量，设计了一个称之为`Log Sequeue Number`的全局变量，翻译过来就是：`日志序列号`，简称`lsn`。不过不像人一出生的年龄是`0`岁，设计`InnoDB`的大佬<span style="color:red">规定</span>初始的`lsn`值为`8704`（也就是一条`redo`日志也没写入时，`lsn`的值为`8704`）。

&emsp;&emsp;我们知道在向`log buffer`中写入`redo`日志时不是一条一条写入的，而是以一个`mtr`生成的一组`redo`日志为单位进行写入的。而且实际上是把日志内容写在了`log block body`处。但是在统计`lsn`的增长量时，是按照实际写入的日志量加上占用的`log block header`和`log block trailer`来计算的。我们来看一个例子：

- 系统第一次启动后初始化`log buffer`时，`buf_free`（就是标记下一条`redo`日志应该写入到`log buffer`的位置的变量）就会指向第一个`block`的偏移量为12字节（`log block header`的大小）的地方，那么`lsn`值也会跟着增加12：

    ![][21-06]

- 如果某个`mtr`产生的一组`redo`日志占用的存储空间比较小，也就是待插入的block剩余空闲空间能容纳这个`mtr`提交的日志时，`lsn`增长的量就是该`mtr`生成的`redo`日志占用的字节数，就像这样：

    ![][21-07]

    &emsp;&emsp;我们假设上图中`mtr_1`产生的`redo`日志量为200字节，那么`lsn`就要在`8716`的基础上增加`200`，变为`8916`。

- 如果某个`mtr`产生的一组`redo`日志占用的存储空间比较大，也就是待插入的block剩余空闲空间不足以容纳这个`mtr`提交的日志时，`lsn`增长的量就是该`mtr`生成的`redo`日志占用的字节数加上额外占用的`log block header`和`log block trailer`的字节数，就像这样：

    ![][21-08]
                            
    &emsp;&emsp;我们假设上图中`mtr_2`产生的`redo`日志量为1000字节，为了将`mtr_2`产生的`redo`日志写入`log buffer`，我们不得不额外多分配两个block，所以`lsn`的值需要在`8916`的基础上增加`1000 + 12×2 + 4 × 2 = 1032`。
    
```
小贴士：为什么初始的lsn值为8704呢？我也不太清楚，人家就这么规定的。其实你也可以规定你一生下来算1岁，只要保证随着时间的流逝，你的年龄不断增长就好了。
```
&emsp;&emsp;从上面的描述中可以看出来，<span style="color:red">每一组由mtr生成的redo日志都有一个唯一的LSN值与其对应，LSN值越小，说明redo日志产生的越早</span>。

### flushed_to_disk_lsn
&emsp;&emsp;`redo`日志是首先写到`log buffer`中，之后才会被刷新到磁盘上的`redo`日志文件。所以设计`InnoDB`的大佬提出了一个称之为`buf_next_to_write`的全局变量，标记当前`log buffer`中已经有哪些日志被刷新到磁盘中了。画个图表示就是这样：

![][21-09]

&emsp;&emsp;我们前面说`lsn`是表示当前系统中写入的`redo`日志量，这包括了写到`log buffer`而没有刷新到磁盘的日志，相应的，设计`InnoDB`的大佬提出了一个表示刷新到磁盘中的`redo`日志量的全局变量，称之为`flushed_to_disk_lsn`。系统第一次启动时，该变量的值和初始的`lsn`值是相同的，都是`8704`。随着系统的运行，`redo`日志被不断写入`log buffer`，但是并不会立即刷新到磁盘，`lsn`的值就和`flushed_to_disk_lsn`的值拉开了差距。我们演示一下：

- 系统第一次启动后，向`log buffer`中写入了`mtr_1`、`mtr_2`、`mtr_3`这三个`mtr`产生的`redo`日志，假设这三个`mtr`开始和结束时对应的lsn值分别是：

    - `mtr_1`：8716 ～ 8916
    - `mtr_2`：8916 ～ 9948
    - `mtr_3`：9948 ～ 10000

    &emsp;&emsp;此时的`lsn`已经增长到了10000，但是由于没有刷新操作，所以此时`flushed_to_disk_lsn`的值仍为`8704`，如图：

    ![][21-10]
    
- 随后进行将`log buffer`中的block刷新到`redo`日志文件的操作，假设将`mtr_1`和`mtr_2`的日志刷新到磁盘，那么`flushed_to_disk_lsn`就应该增长`mtr_1`和`mtr_2`写入的日志量，所以`flushed_to_disk_lsn`的值增长到了`9948`，如图：

    ![][21-11]

&emsp;&emsp;综上所述，当有新的`redo`日志写入到`log buffer`时，首先`lsn`的值会增长，但`flushed_to_disk_lsn`不变，随后随着不断有`log buffer`中的日志被刷新到磁盘上，`flushed_to_disk_lsn`的值也跟着增长。<span style="color:red">如果两者的值相同时，说明log buffer中的所有redo日志都已经刷新到磁盘中了</span>。

```
小贴士：应用程序向磁盘写入文件时其实是先写到操作系统的缓冲区中去，如果某个写入操作要等到操作系统确认已经写到磁盘时才返回，那需要调用一下操作系统提供的fsync函数。其实只有当系统执行了fsync函数后，flushed_to_disk_lsn的值才会跟着增长，当仅仅把log buffer中的日志写入到操作系统缓冲区却没有显式的刷新到磁盘时，另外的一个称之为write_lsn的值跟着增长。不过为了大家理解上的方便，我们在讲述时把flushed_to_disk_lsn和write_lsn的概念混淆了起来。
```

### lsn值和redo日志文件偏移量的对应关系
&emsp;&emsp;因为`lsn`的值是代表系统写入的`redo`日志量的一个总和，一个`mtr`中产生多少日志，`lsn`的值就增加多少（当然有时候要加上`log block header`和`log block trailer`的大小），这样`mtr`产生的日志写到磁盘中时，很容易计算某一个`lsn`值在`redo`日志文件组中的偏移量，如图：

![][21-12]

&emsp;&emsp;初始时的`LSN`值是`8704`，对应文件偏移量`2048`，之后每个`mtr`向磁盘中写入多少字节日志，`lsn`的值就增长多少。

### flush链表中的LSN
&emsp;&emsp;我们知道一个`mtr`代表一次对底层页面的原子访问，在访问过程中可能会产生一组不可分割的`redo`日志，在`mtr`结束时，会把这一组`redo`日志写入到`log buffer`中。除此之外，在`mtr`结束时还有一件非常重要的事情要做，就是<span style="color:red">把在mtr执行过程中可能修改过的页面加入到Buffer Pool的flush链表</span>。为了防止大家早已忘记`flush链表`是什么，我们再看一下图：

![][21-13]
        
&emsp;&emsp;当第一次修改某个缓存在`Buffer Pool`中的页面时，就会把这个页面对应的控制块插入到`flush链表`的头部，之后再修改该页面时由于它已经在`flush`链表中了，就不再次插入了。也就是说<span style="color:red">flush链表中的脏页是按照页面的第一次修改时间从大到小进行排序的</span>。在这个过程中会在缓存页对应的控制块中记录两个关于页面何时修改的属性：

- `oldest_modification`：如果某个页面被加载到`Buffer Pool`后进行第一次修改，那么就将修改该页面的`mtr`开始时对应的`lsn`值写入这个属性。

- `newest_modification`：每修改一次页面，都会将修改该页面的`mtr`结束时对应的`lsn`值写入这个属性。也就是说该属性表示页面最近一次修改后对应的系统`lsn`值。

&emsp;&emsp;我们接着上面介绍`flushed_to_disk_lsn`的例子看一下：

- 假设`mtr_1`执行过程中修改了`页a`，那么在`mtr_1`执行结束时，就会将`页a`对应的控制块加入到`flush链表`的头部。并且将`mtr_1`开始时对应的`lsn`，也就是`8716`写入`页a`对应的控制块的`oldest_modification`属性中，把`mtr_1`结束时对应的`lsn`，也就是8916写入`页a`对应的控制块的`newest_modification`属性中。画个图表示一下（为了让图片美观一些，我们把`oldest_modification`缩写成了`o_m`，把`newest_modification`缩写成了`n_m`）：

    ![][21-14]
    
- 接着假设`mtr_2`执行过程中又修改了`页b`和`页c`两个页面，那么在`mtr_2`执行结束时，就会将`页b`和`页c`对应的控制块都加入到`flush链表`的头部。并且将`mtr_2`开始时对应的`lsn`，也就是8916写入`页b`和`页c`对应的控制块的`oldest_modification`属性中，把`mtr_2`结束时对应的`lsn`，也就是9948写入`页b`和`页c`对应的控制块的`newest_modification`属性中。画个图表示一下：

    ![][21-15]

    &emsp;&emsp;从图中可以看出来，每次新插入到`flush链表`中的节点都是被放在了头部，也就是说`flush链表`中前面的脏页修改的时间比较晚，后边的脏页修改时间比较早。
    
- 接着假设`mtr_3`执行过程中修改了`页b`和`页d`，不过`页b`之前已经被修改过了，所以它对应的控制块已经被插入到了`flush`链表，所以在`mtr_3`执行结束时，只需要将`页d`对应的控制块都加入到`flush链表`的头部即可。所以需要将`mtr_3`开始时对应的`lsn`，也就是9948写入`页d`对应的控制块的`oldest_modification`属性中，把`mtr_3`结束时对应的`lsn`，也就是10000写入`页d`对应的控制块的`newest_modification`属性中。另外，由于`页b`在`mtr_3`执行过程中又发生了一次修改，所以需要更新`页b`对应的控制块中`newest_modification`的值为10000。画个图表示一下：

    ![][21-16]

&emsp;&emsp;总结一下上面说的，就是：<span style="color:red">flush链表中的脏页按照修改发生的时间顺序进行排序，也就是按照oldest_modification代表的LSN值进行排序，被多次更新的页面不会重复插入到flush链表中，但是会更新newest_modification属性的值</span>。

## checkpoint
&emsp;&emsp;有一个很不幸的事实就是我们的`redo`日志文件组容量是有限的，我们不得不选择循环使用`redo`日志文件组中的文件，但是这会造成最后写的`redo`日志与最开始写的`redo`日志`追尾`，这时应该想到：<span style="color:red">redo日志只是为了系统奔溃后恢复脏页用的，如果对应的脏页已经刷新到了磁盘，也就是说即使现在系统奔溃，那么在重启后也用不着使用redo日志恢复该页面了，所以该redo日志也就没有存在的必要了，那么它占用的磁盘空间就可以被后续的redo日志所重用</span>。也就是说：<span style="color:red">判断某些redo日志占用的磁盘空间是否可以覆盖的依据就是它对应的脏页是否已经刷新到磁盘里</span>。我们看一下前面一直介绍的那个例子：

![][21-17]

&emsp;&emsp;如图，虽然`mtr_1`和`mtr_2`生成的`redo`日志都已经被写到了磁盘上，但是它们修改的脏页仍然留在`Buffer Pool`中，所以它们生成的`redo`日志在磁盘上的空间是不可以被覆盖的。之后随着系统的运行，如果`页a`被刷新到了磁盘，那么它对应的控制块就会从`flush链表`中移除，就像这样子：

![][21-18]

&emsp;&emsp;这样`mtr_1`生成的`redo`日志就没有用了，它们占用的磁盘空间就可以被覆盖掉了。设计`InnoDB`的大佬提出了一个全局变量`checkpoint_lsn`来代表当前系统中可以被覆盖的`redo`日志总量是多少，这个变量初始值也是`8704`。

&emsp;&emsp;比方说现在`页a`被刷新到了磁盘，`mtr_1`生成的`redo`日志就可以被覆盖了，所以我们可以进行一个增加`checkpoint_lsn`的操作，我们把这个过程称之为做一次`checkpoint`。做一次`checkpoint`其实可以分为两个步骤：

- 步骤一：计算一下当前系统中可以被覆盖的`redo`日志对应的`lsn`值最大是多少。

    &emsp;&emsp;`redo`日志可以被覆盖，意味着它对应的脏页被刷到了磁盘，只要我们计算出当前系统中被最早修改的脏页对应的`oldest_modification`值，那<span style="color:red">凡是在系统lsn值小于该节点的oldest_modification值时产生的redo日志都是可以被覆盖掉的</span>，我们就把该脏页的`oldest_modification`赋值给`checkpoint_lsn`。

    &emsp;&emsp;比方说当前系统中`页a`已经被刷新到磁盘，那么`flush链表`的尾节点就是`页c`，该节点就是当前系统中最早修改的脏页了，它的`oldest_modification`值为8916，我们就把8916赋值给`checkpoint_lsn`（也就是说在redo日志对应的lsn值小于8916时就可以被覆盖掉）。

- 步骤二：将`checkpoint_lsn`和对应的`redo`日志文件组偏移量以及此次`checkpint`的编号写到日志文件的管理信息（就是`checkpoint1`或者`checkpoint2`）中。

    &emsp;&emsp;设计`InnoDB`的大佬维护了一个目前系统做了多少次`checkpoint`的变量`checkpoint_no`，每做一次`checkpoint`，该变量的值就加1。我们前面说过计算一个`lsn`值对应的`redo`日志文件组偏移量是很容易的，所以可以计算得到该`checkpoint_lsn`在`redo`日志文件组中对应的偏移量`checkpoint_offset`，然后把这三个值都写到`redo`日志文件组的管理信息中。
    
    &emsp;&emsp;我们说过，每一个`redo`日志文件都有`2048`个字节的管理信息，但是<span style="color:red">上述关于checkpoint的信息只会被写到日志文件组的第一个日志文件的管理信息中</span>。不过我们是存储到`checkpoint1`中还是`checkpoint2`中呢？设计`InnoDB`的大佬规定，当`checkpoint_no`的值是偶数时，就写到`checkpoint1`中，是奇数时，就写到`checkpoint2`中。

&emsp;&emsp;记录完`checkpoint`的信息之后，`redo`日志文件组中各个`lsn`值的关系就像这样：

![][21-19]

### 批量从flush链表中刷出脏页
&emsp;&emsp;我们在介绍`Buffer Pool`的时候说过，一般情况下都是后台的线程在对`LRU链表`和`flush链表`进行刷脏操作，这主要因为刷脏操作比较慢，不想影响用户线程处理请求。但是如果当前系统修改页面的操作十分频繁，这样就导致写日志操作十分频繁，系统`lsn`值增长过快。如果后台的刷脏操作不能将脏页刷出，那么系统无法及时做`checkpoint`，可能就需要用户线程同步的从`flush链表`中把那些最早修改的脏页（`oldest_modification`最小的脏页）刷新到磁盘，这样这些脏页对应的`redo`日志就没用了，然后就可以去做`checkpoint`了。

### 查看系统中的各种LSN值
&emsp;&emsp;我们可以使用`SHOW ENGINE INNODB STATUS`命令查看当前`InnoDB`存储引擎中的各种`LSN`值的情况，比如：

```
mysql> SHOW ENGINE INNODB STATUS\G

(...省略前面的许多状态)
LOG
---
Log sequence number 124476971
Log flushed up to   124099769
Pages flushed up to 124052503
Last checkpoint at  124052494
0 pending log flushes, 0 pending chkp writes
24 log i/o's done, 2.00 log i/o's/second
----------------------
(...省略后边的许多状态)
```
其中：
- `Log sequence number`：代表系统中的`lsn`值，也就是当前系统已经写入的`redo`日志量，包括写入`log buffer`中的日志。
- `Log flushed up to`：代表`flushed_to_disk_lsn`的值，也就是当前系统已经写入磁盘的`redo`日志量。
- `Pages flushed up to`：代表`flush链表`中被最早修改的那个页面对应的`oldest_modification`属性值。
- `Last checkpoint at`：当前系统的`checkpoint_lsn`值。

## innodb_flush_log_at_trx_commit的用法
&emsp;&emsp;我们前面说为了保证事务的`持久性`，用户线程在事务提交时需要将该事务执行过程中产生的所有`redo`日志都刷新到磁盘上。这一条要求太狠了，会很明显的降低数据库性能。如果有的同学对事务的`持久性`要求不是那么强烈的话，可以选择修改一个称为`innodb_flush_log_at_trx_commit`的系统变量的值，该变量有3个可选的值：

- `0`：当该系统变量值为0时，表示在事务提交时不立即向磁盘中同步`redo`日志，这个任务是交给后台线程做的。

    &emsp;&emsp;这样很明显会加快请求处理速度，但是如果事务提交后服务器挂了，后台线程没有及时将`redo`日志刷新到磁盘，那么该事务对页面的修改会丢失。

- `1`：当该系统变量值为1时，表示在事务提交时需要将`redo`日志同步到磁盘，可以保证事务的`持久性`。`1`也是`innodb_flush_log_at_trx_commit`的默认值。
- `2`：当该系统变量值为2时，表示在事务提交时需要将`redo`日志写到操作系统的缓冲区中，但并不需要保证将日志真正的刷新到磁盘。

    &emsp;&emsp;这种情况下如果数据库挂了，操作系统没挂的话，事务的`持久性`还是可以保证的，但是操作系统也挂了的话，那就不能保证`持久性`了。

## 崩溃恢复
&emsp;&emsp;在服务器不挂的情况下，`redo`日志简直就是个大累赘，不仅没用，反而让性能变得更差。但是万一，我说万一啊，万一数据库挂了，那`redo`日志可是个宝了，我们就可以在重启时根据`redo`日志中的记录就可以将页面恢复到系统奔溃前的状态。我们接下来大致看一下恢复过程是什么样。

### 确定恢复的起点
&emsp;&emsp;我们前面说过，`checkpoint_lsn`之前的`redo`日志都可以被覆盖，也就是说这些`redo`日志对应的脏页都已经被刷新到磁盘中了，既然它们已经被刷盘，我们就没必要恢复它们了。对于`checkpoint_lsn`之后的`redo`日志，它们对应的脏页可能没被刷盘，也可能被刷盘了，我们不能确定，所以需要从`checkpoint_lsn`开始读取`redo`日志来恢复页面。

&emsp;&emsp;当然，`redo`日志文件组的第一个文件的管理信息中有两个block都存储了`checkpoint_lsn`的信息，我们当然是要选取<span style="color:Red">最近发生的那次checkpoint的信息</span>。衡量`checkpoint`发生时间早晚的信息就是所谓的`checkpoint_no`，我们只要把`checkpoint1`和`checkpoint2`这两个block中的`checkpoint_no`值读出来比一下大小，哪个的`checkpoint_no`值更大，说明哪个block存储的就是最近的一次`checkpoint`信息。这样我们就能拿到最近发生的`checkpoint`对应的`checkpoint_lsn`值以及它在`redo`日志文件组中的偏移量`checkpoint_offset`。

### 确定恢复的终点
&emsp;&emsp;`redo`日志恢复的起点确定了，那终点是哪个呢？这个还得从block的结构说起。我们说在写`redo`日志的时候都是顺序写的，写满了一个block之后会再往下一个block中写：

![][21-20]

&emsp;&emsp;普通block的`log block header`部分有一个称之为`LOG_BLOCK_HDR_DATA_LEN`的属性，该属性值记录了当前block里使用了多少字节的空间。对于被填满的block来说，该值永远为`512`。如果该属性的值不为`512`，那么就是它了，它就是此次奔溃恢复中需要扫描的最后一个block。

### 怎么恢复
&emsp;&emsp;确定了需要扫描哪些`redo`日志进行奔溃恢复之后，接下来就是怎么进行恢复了。假设现在的`redo`日志文件中有5条`redo`日志，如图：

![][21-21]

&emsp;&emsp;由于`redo 0`在`checkpoint_lsn`后边，恢复时可以不管它。我们现在可以按照`redo`日志的顺序依次扫描`checkpoint_lsn`之后的各条redo日志，按照日志中记载的内容将对应的页面恢复出来。这样没什么问题，不过设计`InnoDB`的大佬还是想了一些办法加快这个恢复的过程：

- 使用哈希表

    &emsp;&emsp;根据`redo`日志的`space ID`和`page number`属性计算出散列值，把`space ID`和`page number`相同的`redo`日志放到哈希表的同一个槽里，如果有多个`space ID`和`page number`都相同的`redo`日志，那么它们之间使用链表连接起来，按照生成的先后顺序链接起来的，如图所示：

    ![][21-22]
    
    &emsp;&emsp;之后就可以遍历哈希表，因为对同一个页面进行修改的`redo`日志都放在了一个槽里，所以可以一次性将一个页面修复好（避免了很多读取页面的随机IO），这样可以加快恢复速度。另外需要注意一点的是，同一个页面的`redo`日志是按照生成时间顺序进行排序的，所以恢复的时候也是按照这个顺序进行恢复，如果不按照生成时间顺序进行排序的话，那么可能出现错误。比如原先的修改操作是先插入一条记录，再删除该条记录，如果恢复时不按照这个顺序来，就可能变成先删除一条记录，再插入一条记录，这显然是错误的。

- 跳过已经刷新到磁盘的页面

    &emsp;&emsp;我们前面说过，`checkpoint_lsn`之前的`redo`日志对应的脏页确定都已经刷到磁盘了，但是`checkpoint_lsn`之后的`redo`日志我们不能确定是否已经刷到磁盘，主要是因为在最近做的一次`checkpoint`后，可能后台线程又不断的从`LRU链表`和`flush链表`中将一些脏页刷出`Buffer Pool`。这些在`checkpoint_lsn`之后的`redo`日志，如果它们对应的脏页在奔溃发生时已经刷新到磁盘，那在恢复时也就没有必要根据`redo`日志的内容修改该页面了。
    
    &emsp;&emsp;那在恢复时怎么知道某个`redo`日志对应的脏页是否在奔溃发生时已经刷新到磁盘了呢？这还得从页面的结构说起，我们前面说过每个页面都有一个称之为`File Header`的部分，在`File Header`里有一个称之为`FIL_PAGE_LSN`的属性，该属性记载了最近一次修改页面时对应的`lsn`值（其实就是页面控制块中的`newest_modification`值）。如果在做了某次`checkpoint`之后有脏页被刷新到磁盘中，那么该页对应的`FIL_PAGE_LSN`代表的`lsn`值肯定大于`checkpoint_lsn`的值，凡是符合这种情况的页面就不需要重复执行lsn值小于`FIL_PAGE_LSN`的redo日志了，所以更进一步提升了奔溃恢复的速度。

## 遗漏的问题：LOG_BLOCK_HDR_NO是如何计算的
&emsp;&emsp;我们前面说过，对于实际存储`redo`日志的普通的`log block`来说，在`log block header`处有一个称之为`LOG_BLOCK_HDR_NO`的属性（忘记了的话回头再看看），我们说这个属性代表一个唯一的标号。这个属性是初次使用该block时分配的，跟当时的系统`lsn`值有关。使用下面的公式计算该block的`LOG_BLOCK_HDR_NO`值：
```
((lsn / 512) & 0x3FFFFFFFUL) + 1
```
&emsp;&emsp;这个公式里的`0x3FFFFFFFUL`可能让大家有点困惑，其实它的二进制表示可能更亲切一点：

![][21-23]

&emsp;&emsp;从图中可以看出，`0x3FFFFFFFUL`对应的二进制数的前2位为0，后30位的值都为`1`。我们刚开始学计算机的时候就学过，一个二进制位与0做与运算（`&`）的结果肯定是0，一个二进制位与1做与运算（`&`）的结果就是原值。让一个数和`0x3FFFFFFFUL`做与运算的意思就是要将该值的前2个比特位的值置为0，这样该值就肯定小于或等于`0x3FFFFFFFUL`了。这也就说明了，不论lsn多大，`((lsn / 512) & 0x3FFFFFFFUL)`的值肯定在`0`\~`0x3FFFFFFFUL`之间，再加1的话肯定在`1`\~`0x40000000UL`之间。而`0x40000000UL`这个值大家应该很熟悉，这个值就代表着`1GB`。也就是说系统最多能产生不重复的`LOG_BLOCK_HDR_NO`值只有`1GB`个。设计InnoDB的大佬规定`redo`日志文件组中包含的所有文件大小总和不得超过512GB，一个block大小是512字节，也就是说redo日志文件组中包含的block块最多为1GB个，所以有1GB个不重复的编号值也就够用了。

&emsp;&emsp;另外，`LOG_BLOCK_HDR_NO`值的第一个比特位比较特殊，称之为`flush bit`，如果该值为1，代表着本block是在某次将`log buffer`中的block刷新到磁盘的操作中的第一个被刷入的block。

  [21-01]: ../images/21-01.png
  [21-02]: ../images/21-02.png
  [21-03]: ../images/21-03.png
  [21-04]: ../images/21-04.png
  [21-05]: ../images/21-05.png
  [21-06]: ../images/21-06.png
  [21-07]: ../images/21-07.png
  [21-08]: ../images/21-08.png
  [21-09]: ../images/21-09.png
  [21-10]: ../images/21-10.png
  [21-11]: ../images/21-11.png
  [21-12]: ../images/21-12.png
  [21-13]: ../images/21-13.png
  [21-14]: ../images/21-14.png
  [21-15]: ../images/21-15.png
  [21-16]: ../images/21-16.png
  [21-17]: ../images/21-17.png
  [21-18]: ../images/21-18.png
  [21-19]: ../images/21-19.png
  [21-20]: ../images/21-20.png
  [21-21]: ../images/21-21.png
  [21-22]: ../images/21-22.png
  [21-23]: ../images/21-23.png

<div STYLE="page-break-after: always;"></div>

