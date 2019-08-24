# PipedInputStream

## 基本信息

作用是让多线程可以通过管道进行线程间的通讯。

需要注意的是：在使用管道通信时，必须将PipedOutputStream和PipedInputStream配套使用。

使用管道通信时，大致的流程是：我们在线程A中向PipedOutputStream中写入数据，这些数据会自动的发送到与PipedOutputStream对应的PipedInputStream中，进而存储在PipedInputStream的缓冲中；此时，线程B通过读取PipedInputStream中的数据。就可以实现，线程A和线程B的通信。

## 源码解析

```java
public class PipedInputStream extends InputStream {
  
  	// PipedOutputStream关闭标识
    boolean closedByWriter = false;
  	// PipedInputStream关闭标识
    volatile boolean closedByReader = false;
  	// 连接标识，是否连接PipedOutputStream
    boolean connected = false;

    // 输入流端线程
    Thread readSide;
  	// 输出流端线程
    Thread writeSide;
		// 默认管道容量为1024字节
    private static final int DEFAULT_PIPE_SIZE = 1024;

    // 管道容量，默认使用DEFAULT_PIPE_SIZE = 1024字节
    protected static final int PIPE_SIZE = DEFAULT_PIPE_SIZE;

    // PipedInputStream内部的存储数组
    protected byte buffer[];

    // 下一个要写入流中字节的位置
  	// 这里设置为-1是因为，类中方法用来判断是否读取了流中所有的数据的条件是(in == out)
  	// 如果此处设置为0，则无法进行写入
  	// 在方法内部，会判断如果in < 0，会将in赋值为0
    protected int in = -1;

    // 下一个要从流中读取的字节的位置
    protected int out = 0;

    // 构造方法
  	// 调用PipedInputStream(PipedOutputStream src, int pipeSize)
    public PipedInputStream(PipedOutputStream src) throws IOException {
        this(src, DEFAULT_PIPE_SIZE);
    }

    // 构造方法
    public PipedInputStream(PipedOutputStream src, int pipeSize)
            throws IOException {
      	 // 初始化内部byte数组容量
         initPipe(pipeSize);
      	 // 连接PipedInputStream
         connect(src);
    }

    // 无参构造函数
  	// 如果要进行通信，需要主动调用connect方法连接
    public PipedInputStream() {
        initPipe(DEFAULT_PIPE_SIZE);
    }

    // 构造函数
  	// 指定内部数组的容量
    // 如果要进行通信，需要主动调用connect方法连接
    public PipedInputStream(int pipeSize) {
        initPipe(pipeSize);
    }
		
  	// 初始化内部数组容量
    private void initPipe(int pipeSize) {
         if (pipeSize <= 0) {
            throw new IllegalArgumentException("Pipe Size <= 0");
         }
         buffer = new byte[pipeSize];
    }

    // 连接PipedOutputStream，实际上是调用PipedOutputStream中的connect方法
    public void connect(PipedOutputStream src) throws IOException {
        src.connect(this);
    }

    // 接收一个字节到内部数组中
  	// 实际上是PipedOutStream中wirte()的实现
  	// 同步方法
    protected synchronized void receive(int b) throws IOException {
      	// 检查相关的状态
        checkStateForReceive();
      	// 设置写入线程为当前线程
        writeSide = Thread.currentThread();
      	// 如果in == out代表缓冲区满，进入等待
      	// 因为read()中如果out == in,则会将in = -1，故此处代表的是in已经填充了数组所有位置又回到了0
        if (in == out)
          	// 等待方法
          	// 交出锁，让readThread线程进行读取，使程序读取缓存
            awaitSpace();
      	// 如果in < 0，代表缓冲区为空（read方法中设置为-1，代表已读取所有缓存数组的数据）
      	// 初始化in为0
        if (in < 0) {
            in = 0;
            out = 0;
        }
      	// in位置移动一位
      	// &0xFF 确保正数，且为8位
        buffer[in++] = (byte)(b & 0xFF);
      	// 如果写入的位置 >= buffer.length，则代表下标达到最大值
      	// 将下标移动至0，因为之前的数据可能被读取过了，可以重新使用，否则之前的数据将被覆盖
      	// 重复使用，减少内存负担
        if (in >= buffer.length) {
            in = 0;
        }
    }

    // 接收一个字节数组中指定位置的内容到流内部数组中
    synchronized void receive(byte b[], int off, int len)  throws IOException {
      	// 检查状态
        checkStateForReceive();
      	// 获取写入线程
        writeSide = Thread.currentThread();
      	// 需要转换为流的字节数
        int bytesToTransfer = len;
      	// 当bytesToTransfer > 0时
        while (bytesToTransfer > 0) {
          	// 如果in == out代表缓存区满，需要等待读取线程读取之后方可写入
            if (in == out)
                awaitSpace();
          	// 
            int nextTransferAmount = 0;
          	// 如果 out < in，则当前可写入的数量为buffer.length - in
          	// 比如length = 10，in = 5，out = 1，则还可以写入 10 - 5才会区满
            if (out < in) {
              	// 还可写入数
                nextTransferAmount = buffer.length - in;
            // 如果 in < out，代表当前写入数为 length + in，读取数量为out
            // 另一种情况是，in == out，read()将in赋值为-1  	
            } else if (in < out) {
              	// 如果in = -1，代表读取与写入此时一致，可写入数为length
                if (in == -1) {
                    in = out = 0;
                    nextTransferAmount = buffer.length - in;
                } else {
                  	// 否则in的位置不能超过out，因为in实际长度已经比out对了length
                  	// 如果in > out 则会覆盖out未读取的数据
                  	// 比如in = 5, out = 6，则此时仅可在写入 6 - 5 个字节
                    nextTransferAmount = out - in;
                }
            }
          	// 实际可写入数 > 预计写入数时，按预计写入数来
          	// 反之如果实际可写入数 < 预计写入数时，按实际可写入数来
            if (nextTransferAmount > bytesToTransfer)
                nextTransferAmount = bytesToTransfer;
          	// 断言写入数 > 0，否则程序终止
            assert(nextTransferAmount > 0);
          	// 写入数组到流中的缓存数组，长度为nextTransferAmount
            System.arraycopy(b, off, buffer, in, nextTransferAmount);
          	// 因为bytesToTransfer可能大于nextTransferAmount，所以需要多次循环写入
            bytesToTransfer -= nextTransferAmount;
          	// 移动传入数组b的下标
            off += nextTransferAmount;
          	// 移动流中缓存数组的下标
            in += nextTransferAmount;
          	// 最后判断in的位置，如果 >= length，则重置为0
            if (in >= buffer.length) {
                in = 0;
            }
        }
    }
	
  	// 状态检查
    private void checkStateForReceive() throws IOException {
      	// 是否连接
      	// 流是否已经关闭
      	// 读取线程是否存活
        if (!connected) {
            throw new IOException("Pipe not connected");
        } else if (closedByWriter || closedByReader) {
            throw new IOException("Pipe closed");
        } else if (readSide != null && !readSide.isAlive()) {
            throw new IOException("Read end dead");
        }
    }

  	// 输出流无字节可读，需要等待
    private void awaitSpace() throws IOException {
        while (in == out) {
          	// 检查流的状态
            checkStateForReceive();

            // 唤醒所有等待线程
            notifyAll();
            try {
              	// 交出锁，让读取线程读取缓存中的数据
              	// 当前线程等待1000ms后转为就绪态
                wait(1000);
            } catch (InterruptedException ex) {
                throw new java.io.InterruptedIOException();
            }
        }
    }

    // 写入完毕，closedByWriter设置为true
  	// 唤醒等待线程（此处应为读取线程）
  	// 同步方法
    synchronized void receivedLast() {
        closedByWriter = true;
        notifyAll();
    }

    // 从流中读取一个字节
  	// 同步方法
    public synchronized int read()  throws IOException {
      	// 状态检查
        if (!connected) {
            throw new IOException("Pipe not connected");
        } else if (closedByReader) {
            throw new IOException("Pipe closed");
        } else if (writeSide != null && !writeSide.isAlive()
                   && !closedByWriter && (in < 0)) {
            throw new IOException("Write end dead");
        }
				
      	// 获取读取的线程
        readSide = Thread.currentThread();
      	// 确保线程不会死循环，如果写入线程broken，则in一直小于0
        int trials = 2;
      	// 当in < 0
        while (in < 0) {
          	// 且写入流关闭，则无法读取
            if (closedByWriter) {
                return -1;
            }
          	// 且写入线程存在，等待两次唤醒，如果两次未唤醒写入线程，则抛出异常
            if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
                throw new IOException("Pipe broken");
            }
            // 唤醒写入线程
            notifyAll();
            try {
              	// 让出锁，等待写入
                wait(1000);
            } catch (InterruptedException ex) {
                throw new java.io.InterruptedIOException();
            }
        }
      	// 确保八位且非负
        int ret = buffer[out++] & 0xFF;
      	// 如果读取的下标大于等于buffer.length则归零
        if (out >= buffer.length) {
            out = 0;
        }
      	// 已经读取了流中所有的数据
        if (in == out) {
            // 将in位置设置为-1
            in = -1;
        }
				
      	// 返回从缓存数组读取的字节数据
        return ret;
    }

  	// 从输入流中读取指定长度的数据到指定数组
  	// 同步方法
    public synchronized int read(byte b[], int off, int len)  throws IOException {
      	// null校验
        if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }

        // 先行读取一个字节
        int c = read();
      	// 如果c < 0代表已到流末尾，无数据可读
        if (c < 0) {
            return -1;
        }
      	// 否则写入第一个数据到off位置
        b[off] = (byte) c;
      	// 实际已读取的数量
        int rlen = 1;
      	// in >= 0代表out并未完全读取输出流的里的数据
      	// len确保读取的数量 > 0,因为上方已先行读取了一个数据，故此处应为1
        while ((in >= 0) && (len > 1)) {

          	// 可读数量
            int available;

          	// 如果in > out
            if (in > out) {
                available = Math.min((buffer.length - out), (in - out));
            } else {
              	// 反之 out > in，此时先行读取到length - 1的位置，才会使得out = 0
              	// 故可读取的数量为buffer.length - out
                available = buffer.length - out;
            }

            // 可读数量 > 需要读取的数量时，按需读取
            if (available > (len - 1)) {
                available = len - 1;
            }
          	// 读取数据
            System.arraycopy(buffer, out, b, off + rlen, available);
          	// 相应变量的改变
            out += available;
            rlen += available;
            len -= available;
						// 重置out的位置
            if (out >= buffer.length) {
                out = 0;
            }
          	// in == out代表已完全读取
            if (in == out) {
                in = -1;
            }
        }
      	// 返回实际读取的字节数
        return rlen;
    }

    // 流中可读取的字节数
  	// 同步方法
    public synchronized int available() throws IOException {
      	// 如果in < 0则代表，未写入字节，可读字节数为0
        if(in < 0)
            return 0;
      	// 如果in == out，代表缓存区满，读取线程未进行读取，故可读数量为buffer.length
        else if(in == out)
            return buffer.length;
      	// 如果in > out，可读字节数为in - out(输入 - 已读 = 未读)
        else if (in > out)
            return in - out;
      	// 这种情况对应out > in
      	// 代表in已经达到buffer.length过，又被设置为0
      	// 所以写入的总字节数为当前的in + 数组的长度（in + buffer.length）
        else
            return in + buffer.length - out;
    }

    // 关闭PipedInputStream
    public void close()  throws IOException {
      	// 将PipedInputStream关闭标识设置为true
        closedByReader = true;
        synchronized (this) {
          	// 设置读取位置为-1
            in = -1;
        }
    }
}
```

