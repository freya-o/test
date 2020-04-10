# 前言
今天给大家带来 DBLE 的内存管理模块源码解析文章。
本文主要分为两部分，一是内存管理模块结构，二是内存管理模块源码解析。

# 内存管理模块结构
DBLE 管理的内存的主要目的有以下两点：

1. 网络读写
2. 查询结果集处理

简单说明一下这两点

第一点比较好理解，网络读的时候需要从通道中读取字节流到内存中，以便进一步处理，网络写的时候也需要将字节流从内存中写入到通道中，以便发送信息。

第二点就比较复杂一些了，DBLE 获取到 MySQL 端的数据后，会经过 DBLE 然后再发送给客户端，如果是简单查询的话，DBLE 端只是起到一个中转的作用，会利用少量内存用于将结果转发给客户端。如果是复杂查询的话，DBLE 会先对结果集进行处理，比如连接、分组排序等操作，然后再将结果发送给客户端，这里涉及到的内存使用就复杂了，对于复杂查询 DBLE 会先使用堆内存，如果使用的堆内存超过限制，则会写内存映射文件，如果写内存映射文件个数再次达到上限，则会写硬盘。是不是很复杂？放心，这一块本文不会介绍，原因我就不说了（太复杂了）……。

我们看一下 DBLE 官方内存结构图：

![内存结构概览](https://upload-images.jianshu.io/upload_images/7567026-bdd94c01870d1377.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

个人觉得上图有一点理解上的疑惑，`DirectByteBufferPool` 占用的内存应该位于物理内存中，JVM 中只是有着堆外内存的引用对象而已。

下图个人觉得可能更好理解。

![内存结构概览](https://upload-images.jianshu.io/upload_images/7567026-362b488c285315ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

结合DBLE管理内存的主要两个作用，网络读写主要用到了堆外内存，即 `DirectByteBufferPool` 管理的部分。结果集处理根据情况有所不同，本文只讨论简单情况，对于简单情况来讲，会使用到堆外内存或堆内存，下文源码中会提到具体情况。

# 内存管理模块源码解析
来看下内存管理模块涉及到的类：

![内存管理相关类图](https://upload-images.jianshu.io/upload_images/7567026-fb6504e7b0b3dd91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图可以看出内存管理模块涉及到的类不多，只有两个：

1. `DirectByteBufferPool` 类即为管理堆外内存的内存池类，可以看到它创建了 `ByteBuffePage` 类，并且与该类有着一对多的关系；
2. `ByteBufferPage` 类为 DBLE 抽象出的堆外内存页的概念，内存页中又有内存块的概念，内存块是内存分配的最小单元。采用此种内存概念主要是为了减少内存碎片问题。

DBLE 官方对于这两个类的内部结构以图的形式表示的很清楚了：

![BufferPool结构](https://upload-images.jianshu.io/upload_images/7567026-bdecddc0d78fe7c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![ByteBufferPage结构](https://upload-images.jianshu.io/upload_images/7567026-5e7ef61a0c35fcdb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中的参数 `bufferPoolPageNumber`、`bufferPoolPageSize` 和 `bufferPoolChunkSize` 可在 `Server.xml` 中配置 ，`bufferPoolPageSize` 默认为 2M, `bufferPoolPageNumber` 默认为 Java 虚拟机的可用的处理器数量 * 20，`bufferPoolChunkSize` 的参数默认为 4k，该参数最好设为 `bufferPoolPageSize` 的约数，否则最后一个会造成浪费。

在看源码前，先看下内存分配的逻辑：

1. 如果不指定分配大小，则默认分配一个最小单元(最小单元由 `bufferPoolChunkSize` 决定）；
2. 如果指定分配大小，则分配放得下分配大小的最小单元的整数倍(向上取整) ，举个例子，如果需要 10k 的大小，则在 `bufferPoolChunkSize` 参数默认为 4k 的情况下，会分配 3 个内存 chunk；
3. 分配逻辑为：
（1）遍历缓冲池从 N+1 页到 `bufferPoolPageNumber` -1 页(上次分配过的记为第 N 页)，然后对单页加锁在每个页中从头寻找未被使用的连续 M 个最小单元 ；
（2）如果没找到，再从第 0 页找到第 N 页；
（3）成功分配内存后更新上次分配页，标记内存页中分配的单元；
（4）如果找不到可存放的单页(比如大于 `bufferPoolPageSize` )，直接分配 On-Heap 内存。

下面我们就开始看看对应的源码。

内存池初始化的代码调用在 `DbleServer#startup` 方法中：

	//通过配置的相关参数初始化内存池
	bufferPool = new DirectByteBufferPool(bufferPoolPageSize, bufferPoolChunkSize, bufferPoolPageNumber);

看下内存池初始化逻辑，代码在 `DirectByteBufferPool` 类的构造方法中：

	public DirectByteBufferPool(int pageSize, short chunkSize, short pageCount) {
	        //初始化内存总页数
	        allPages = new ByteBufferPage[pageCount];
	        //内存页中的内存块大小
	        this.chunkSize = chunkSize;
	        //每个内存页的大小
	        this.pageSize = pageSize;
	        //内存页总数
	        this.pageCount = pageCount;
	        //记录上次分配过的页
	        prevAllocatedPage = new AtomicInteger(0);
	        //循环初始化内存页
	        for (int i = 0; i < pageCount; i++) {
	            allPages[i] = new ByteBufferPage(ByteBuffer.allocateDirect(pageSize), chunkSize);
	        }
	        //记录堆外内存的使用大小
	        memoryUsage = new ConcurrentHashMap<>();
	    }

继续看下内存页的初始化逻辑，在 `ByteBufferPage` 类的构造方法中：

	public ByteBufferPage(ByteBuffer buf, int chunkSize) {
	    //内存块大小
	    this.chunkSize = chunkSize;
	    //计算总内存块数
	    chunkCount = buf.capacity() / chunkSize;
	    //非常巧妙的位图结构，用于标记内存块的使用情况
	    chunkAllocateTrack = new BitSet(chunkCount);
	    //内存页的大小
	    this.buf = buf;
	}

上述代码完成了堆外内存的初始化。

下面让我们看看内存的分配逻辑代码。

分配内存的操作主要在 `DirectByteBufferPool#allocate` 方法中：

	public ByteBuffer allocate(int size) {
	        //计算需要分配的chunk数
	        final int theChunkCount = size / chunkSize + (size % chunkSize == 0 ? 0 : 1);
	        //本次分配的开始页数，为上次分配过的页数（N）+1
	        int selectedPage = prevAllocatedPage.incrementAndGet() % allPages.length;
	        //从N+1页到bufferPoolPageNumber-1页遍历分配，allocateBuffer方法下面会进一步分析
	        ByteBuffer byteBuf = allocateBuffer(theChunkCount, selectedPage, allPages.length);
	        //没有分配成功，则从0页到N页继续遍历分配
	        if (byteBuf == null) {
	            byteBuf = allocateBuffer(theChunkCount, 0, selectedPage);
	        }
	        final long threadId = Thread.currentThread().getId();
	        //分配成功的话，则记录分配到的内存大小
	        if (byteBuf != null) {
	            if (memoryUsage.containsKey(threadId)) {
	                memoryUsage.put(threadId, memoryUsage.get(threadId) + byteBuf.capacity());
	            } else {
	                memoryUsage.put(threadId, (long) byteBuf.capacity());
	            }
	        }
	        //如果遍历完所有页还没有分配成功，则直接分配堆上内存
	        if (byteBuf == null) {
	            //ByteBuffer.allocate方法为分配JVM堆内存
	            return ByteBuffer.allocate(size);
	        }
	        return byteBuf;
	    }

继续看下 `DirectByteBufferPool#allocateBuffer` 方法：

	private ByteBuffer allocateBuffer(int theChunkCount, int startPage, int endPage) {
	        //从指定页数开始遍历分配内存，分配成功则标记当前分配的页数，然后直接返回
	        for (int i = startPage; i < endPage; i++) {
	            //调用了ByteBufferPage类的allocateChunk方法进行内存块的分配，该方法下面也会进一步分析
	            ByteBuffer buffer = allPages[i].allocateChunk(theChunkCount);
	            if (buffer != null) {
	                prevAllocatedPage.getAndSet(i);
	                return buffer;
	            }
	        }
	        return null;
	    }

继续看一下 `ByteBufferPage#allocateChunk` 方法，该方法比较长，分配页中的连续内存块逻辑就在此方法中：


	public ByteBuffer allocateChunk(int theChunkCount) {
	    //对页加状态锁，防止并发操作异常
	    if (!allocLockStatus.compareAndSet(false, true)) {
	        return null;
	    }
	    int startChunk = -1;
	    int continueCount = 0;
	    try {
	        //下面的逻辑为在页中找到符合内存分配大小的连续内存块
	        for (int i = 0; i < chunkCount; i++) {
	            if (!chunkAllocateTrack.get(i)) {
	                if (startChunk == -1) {
	                    startChunk = i;
	                    continueCount = 1;
	                    if (theChunkCount == 1) {
	                        break;
	                    }
	                } else {
	                    if (++continueCount == theChunkCount) {
	                        break;
	                    }
	                }
	            } else {
	                startChunk = -1;
	                continueCount = 0;
	            }
	        }
	        //成功找到后则返回分配的内存块，并且标记相应的内存块位置
	        if (continueCount == theChunkCount) {
	            int offStart = startChunk * chunkSize;
	            int offEnd = offStart + theChunkCount * chunkSize;
	            buf.limit(offEnd);
	            buf.position(offStart);
	
	            ByteBuffer newBuf = buf.slice();
	            markChunksUsed(startChunk, theChunkCount);
	            return newBuf;
	        } else {
	            //分配失败返回null
	            return null;
	        }
	    } finally {
	        //解锁
	        allocLockStatus.set(false);
	    }
	}


到这里，内存分配的逻辑就大概讲完了。

有分配也得有回收，为了文章的完整性，内存回收也顺带讲一下。

内存回收逻辑比较简单，分两种情况，如果是分配的堆内存，则直接 clear 等待 GC 回收，如果是堆外内存，则遍历所有页，找到对应页然后加锁，找到对应块的位置，标记为未使用就可以了。

内存回收的主要代码在 `DirectByteBufferPool#recycle` 方法中：

	public void recycle(ByteBuffer theBuf) {
	        //堆内存的回收
	        if (!(theBuf instanceof DirectBuffer)) {
	            theBuf.clear();
	            return;
	        }
	
	        final long size = theBuf.capacity();
	        //堆外内存的回收
	        boolean recycled = false;
	        DirectBuffer thisNavBuf = (DirectBuffer) theBuf;
	        int chunkCount = theBuf.capacity() / chunkSize;
	        DirectBuffer parentBuf = (DirectBuffer) thisNavBuf.attachment();
	        int startChunk = (int) ((thisNavBuf.address() - parentBuf.address()) / this.chunkSize);
	        //遍历页然后将对应块标记为未使用即可
	        for (ByteBufferPage allPage : allPages) {
	            if ((recycled = allPage.recycleBuffer((ByteBuffer) parentBuf, startChunk, chunkCount))) {
	                break;
	            }
	        }
	        final long threadId = Thread.currentThread().getId();
	
	        if (memoryUsage.containsKey(threadId)) {
	            memoryUsage.put(threadId, memoryUsage.get(threadId) - size);
	        }
	        if (!recycled) {
	            LOGGER.info("warning ,not recycled buffer " + theBuf);
	        }
	    }


内存的分配与回收到这里也讲完了，还记得开始时候说的内存的主要作用吗？一个是网络读写时使用，一个是结果集暂存时使用。

网络读写时使用的例子可以参考这里，`NIOSocketWR#asyncRead` 方法：

	public void asyncRead() throws IOException {
	    ByteBuffer theBuffer = con.readBuffer;
	    if (theBuffer == null) {
	        //这里分配了从通道读取数据时候的内存
	        theBuffer = con.processor.getBufferPool().allocate(con.processor.getBufferPool().getChunkSize());
	        con.readBuffer = theBuffer;
	    }
	    int got = channel.read(theBuffer);
	    con.onReadData(got);
	}

暂存结果集使用的例子可以参考这里，`SingleNodeHandler#rowResponse` 方法：

	//省略无关内容
	......
	//这里分配内存，将结果集写入，然后待发送到客户端
	buffer = session.getSource().writeToBuffer(row, allocBuffer());
	......

当然内存分配的代码调用远不止上面举例的两处，这里只是和内存管理的作用做个呼应。其他地方大家可以自己看。

# 总结
DBLE 的内存管理模块源码阅读的内容如上所述，希望能够对大家更深入的理解 DBLE 有所帮助。