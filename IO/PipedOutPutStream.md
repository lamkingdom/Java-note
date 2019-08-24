# PipedOutputStream

## 基本信息

作用是让多线程可以通过管道进行线程间的通讯。

需要注意的是：在使用管道通信时，必须将PipedOutputStream和PipedInputStream配套使用。

使用管道通信时，大致的流程是：我们在线程A中向PipedOutputStream中写入数据，这些数据会自动的发送到与PipedOutputStream对应的PipedInputStream中，进而存储在PipedInputStream的缓冲中；此时，线程B通过读取PipedInputStream中的数据。就可以实现，线程A和线程B的通信。

## 源码分析

```java
public
class PipedOutputStream extends OutputStream {

    // 管道输入流
    private PipedInputStream sink;

    // 构造方法
  	// 连接两个管道流
    public PipedOutputStream(PipedInputStream snk)  throws IOException {
        connect(snk);
    }

    // 无参构造
  	// 需要主动连接
    public PipedOutputStream() {
    }

    // 连接两个管道流
  	// 实际上PipedInputStream里的connect()方法就是调用此方法
    public synchronized void connect(PipedInputStream snk) throws IOException {
      	// 如果输入流为空
        if (snk == null) {
            throw new NullPointerException();
        // 输入流已连接  	
        } else if (sink != null || snk.connected) {
            throw new IOException("Already connected");
        }
      	// 设置成员变量
        sink = snk;
      	// 设置in、out的初始位置
        snk.in = -1;
        snk.out = 0;
      	// 连接标记设置为true
        snk.connected = true;
    }

    // 向输出流中写入一个字节
  	// 实际上的实现在PipedInputStream里
    public void write(int b)  throws IOException {
        if (sink == null) {
            throw new IOException("Pipe not connected");
        }
        sink.receive(b);
    }

    // 向输出流中写入指定长度字节数组数据
  	// 实际上的实现在PipedInputStream里
    public void write(byte b[], int off, int len) throws IOException {
        if (sink == null) {
            throw new IOException("Pipe not connected");
        } else if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        sink.receive(b, off, len);
    }
		
  	// 目的是让“管道输入流”放弃对当前资源的占有，让其它的等待线程(等待读取管道输出流的线程)读取“管道输出流”的值。
    public synchronized void flush() throws IOException {
        if (sink != null) {
            synchronized (sink) {
                sink.notifyAll();
            }
        }
    }

    // 关闭输出流
  	// 并调用输入流中receivedLast()方法
    public void close()  throws IOException {
        if (sink != null) {
            sink.receivedLast();
        }
    }
}

```

