# BufferedInputStream

## 基本信息

>BufferedInputStream的作用是为其它输入流提供缓冲功能。创建BufferedInputStream时，我们会通过它的构造函数指定某个输入流为参数。BufferedInputStream会将该输入流数据分批读取，每次读取一部分到缓冲中；操作完缓冲中的这部分数据之后，再从输入流中读取下一部分的数据。
>为什么需要缓冲呢？原因很简单，效率问题！缓冲中的数据实际上是保存在内存中，而原始数据可能是保存在硬盘或NandFlash等存储介质中；而我们知道，从内存中读取数据的速度比从硬盘读取数据的速度至少快10倍以上。
>那干嘛不干脆一次性将全部数据都读取到缓冲中呢？第一，读取全部的数据所需要的时间可能会很长。第二，内存价格很贵，容量不像硬盘那么大。

## 源码分析

```java
public
class BufferedInputStream extends FilterInputStream {

  	// 默认缓冲容量
    private static int DEFAULT_BUFFER_SIZE = 8192;

    // 最大缓冲容量
    private static int MAX_BUFFER_SIZE = Integer.MAX_VALUE - 8;

    // 缓冲数组
    protected volatile byte buf[];

    // 缓存数组的原子更新器。
 		// 该成员变量与buf数组的volatile关键字共同组成了buf数组的原子更新功能实现，
 		// 即，在多线程中操作BufferedInputStream对象时，buf和bufUpdater都具有原子性(不同的线程访问到的数据都是相同的)
    private static final
        AtomicReferenceFieldUpdater<BufferedInputStream, byte[]> bufUpdater =
        AtomicReferenceFieldUpdater.newUpdater
        (BufferedInputStream.class,  byte[].class, "buf");

    // 当前读取到缓冲数组中的实际字节数
    protected int count;

    // 当前缓冲数组的下标位置
    protected int pos;

    // markpos用于标记可重复读取的起始下标
  	// -1代表当前没有标记，即缓冲数组中不需要存储已读取的数据
  	// >= 0代表标记
  	// 与reset()联合使用
    protected int markpos = -1;

    // 最大可标记的字节数量
  	// 如果marklimit <= buf.length()则清空可重复读数据
  	// 即markpos = -1
  	// 反之，代表需要存储更多可重复读数据，则扩容buf
    protected int marklimit;

    // 获取输入流(父类的)
    private InputStream getInIfOpen() throws IOException {
        InputStream input = in;
        if (input == null)
            throw new IOException("Stream closed");
        return input;
    }

    // 获取缓冲数组
    private byte[] getBufIfOpen() throws IOException {
        byte[] buffer = buf;
        if (buffer == null)
            throw new IOException("Stream closed");
        return buffer;
    }

    // 构造函数
  	// 使用默认容量构建缓冲数组
    public BufferedInputStream(InputStream in) {
        this(in, DEFAULT_BUFFER_SIZE);
    }

    // 构造方法
  	// 使用传入容量值构建缓冲数组
    public BufferedInputStream(InputStream in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }

    // 填充缓冲区
    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
      	// 情况1
      	// 如果没有标记，则直接重新填充数组
        if (markpos < 0)
            pos = 0;     
      	// markpos >= 0
      	// 缓冲区满
        else if (pos >= buffer.length)
          	// 情况2
          	// markpos > 0,则存储的区间长度(pos - markpos)一定小于buffer.length()
          	// 保存[markpos, pos)区间的数据至buffer[0, pos - markpos)
          	// 重置pos为sz, markpos = 0
            if (markpos > 0) { 
                int sz = pos - markpos;
                System.arraycopy(buffer, markpos, buffer, 0, sz);
                pos = sz;
                markpos = 0;
            // 情况3  
            // markpos = 0,且buffer.length >= marklimit
            // 此时可重复读取的是[0, buffer.length() - 1]  	
            // 此时markpos已经不起效果(pos >= marklimit)
            // 故清空之前存储的可重复读的数据，并重置markpos = -1，最后设置pos = 0
            } else if (buffer.length >= marklimit) {
                markpos = -1;   
                pos = 0;
            // 情况4
            // buffer.length >= MAX_BUFFER_SIZE抛出异常
            } else if (buffer.length >= MAX_BUFFER_SIZE) {
                throw new OutOfMemoryError("Required array size too large");
            // 情况5
            // markpos = 0,且buffer.length < marklimit
            // 此时可重复读取的区间长度大于buffer长度，故需要进行扩容  
            } else {
              	// pos设置为pos * 2或MAX_BUFFER_SIZE
                int nsz = (pos <= MAX_BUFFER_SIZE - pos) ?
                        pos * 2 : MAX_BUFFER_SIZE;
              	// 如果nsz > marklimit
              	// 则取marklimit的值赋给nsz
              	// 取二者中小的值
                if (nsz > marklimit)
                    nsz = marklimit;
              	// 新建一个数组
                byte nbuf[] = new byte[nsz];
              	// 拷贝之前的数据到新数组
                System.arraycopy(buffer, 0, nbuf, 0, pos);
              	// CAS操作更新其他地方的buffer引用为nbuf
                if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                    throw new IOException("Stream closed");
                }
                buffer = nbuf;
            }
      	// pos < buffer.length
        count = pos;
      	// 返回读取的字节数
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);
        if (n > 0)
          	// 更新count字段
            count = n + pos;
    }

    // 读取一个字节数
    public synchronized int read() throws IOException {
      	// 如果读取完毕，则填充
        if (pos >= count) {
            fill();
          	// 如果填充完毕还是pos >= count，则代表读取完毕
            if (pos >= count)
                return -1;
        }
      	// 从buf中读取一个字节数
        return getBufIfOpen()[pos++] & 0xff;
    }

  	// read方法实际的实现
    // 从缓冲中读取长度为len的字节数到b，从b的off位置开始
    private int read1(byte[] b, int off, int len) throws IOException {
      	// count为缓冲数组中的总字节数，pos为下一个读取字节的下标
      	// avail为缓冲数组中可读字节数
        int avail = count - pos;
      	// 如果可读字节数 <= 0,则需要从输入流中读取新的数据到缓冲数组中
        if (avail <= 0) {
            // 如果len >= buf.length() 并且不需要利用缓冲数组进行可重复读(markpos < 0)
           	// 则获取输入流，直接从输入流读取，而不从缓冲数组中读取
            if (len >= getBufIfOpen().length && markpos < 0) {
                return getInIfOpen().read(b, off, len);
            }
          	// 否则，进行缓冲数组的填充
            fill();
          	// 填充完毕后返回可读字节数
            avail = count - pos;
          	// 如果填充完毕后可读字节数还是<= 0 代表流已读完
            if (avail <= 0) return -1;
        }
      	// 如果可读字节数 > 0
      	// 则取min(avail, len) = cnt
        int cnt = (avail < len) ? avail : len;
      	// 拷贝缓冲数组中的数据到传入的数组b，实际长度为cnt
        System.arraycopy(getBufIfOpen(), pos, b, off, cnt);
      	// pos往后移动实际长度的位置
        pos += cnt;
      	// 返回实际读取的字节数
        return cnt;
    }

    // 从缓冲中读取长度为len的字节数到b，从b的off位置开始
  	// 同步方法
    public synchronized int read(byte b[], int off, int len)
        throws IOException
    {
        getBufIfOpen(); // Check for closed stream
        if ((off | len | (off + len) | (b.length - (off + len))) < 0) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }
				
        int n = 0;
      	// 多次进行读取，确保读取完成
        for (;;) {
          	// 使用内部方法read1进行读取，返回实际读取字节数
            int nread = read1(b, off + n, len - n);
          	// 如果实际读取数 <= 0,则代表读取完成
          	// 如果读取完成时，n == 0代表只读取了一次，直接返回nread
          	// 否则返回n（总读取数）
            if (nread <= 0)
                return (n == 0) ? nread : n;
          	// 累计读取数
            n += nread;
          	// 如果累计读取数 >= 需要读取数时则返回n
            if (n >= len)
                return n;
            // 每次读取完成后都检验可读取数是否 <= 0，如果可读数量为0则不再进行循环
            InputStream input = in;
            if (input != null && input.available() <= 0)
                return n;
        }
    }

    // 跳过n个字节后开始读取
    public synchronized long skip(long n) throws IOException {
        getBufIfOpen(); // Check for closed stream
        if (n <= 0) {
            return 0;
        }
      	// 获取当前可读取字节数
        long avail = count - pos;
				
      	// 如果可读取字节数<= 0时
        if (avail <= 0) {
            // 如果没有markpos标记，则直接从输入流中跳过
            if (markpos <0)
                return getInIfOpen().skip(n);

            // 否则填充缓冲数组
            fill();
          	// 再次获取可读字节数
            avail = count - pos;
          	// 如果此时可读字节数还是<= 0 证明已到流的末尾，无法再读取数据
            if (avail <= 0)
                return 0;
        }

	      // 如果可读取字节数> 0时，返回实际可跳过字节数
        long skipped = (avail < n) ? avail : n;
      	// pos往后移动实际跳过的字节数
        pos += skipped;
        return skipped;
    }

    // 获取当前从流及缓冲数组中可读字节数
    public synchronized int available() throws IOException {
      	// n为当前可读字节数
        int n = count - pos;
      	// avail为输入流当前可读字节数
        int avail = getInIfOpen().available();
        return n > (Integer.MAX_VALUE - avail)
                    ? Integer.MAX_VALUE
                    : n + avail;
    }

    // 设置marklimit，并标记当前pos为markpos
    public synchronized void mark(int readlimit) {
        marklimit = readlimit;
        markpos = pos;
    }

    // 将pos设置为markpos
    public synchronized void reset() throws IOException {
        getBufIfOpen(); // Cause exception if closed
        if (markpos < 0)
            throw new IOException("Resetting to invalid mark");
        pos = markpos;
    }

    // 是否支持mark，true
    public boolean markSupported() {
        return true;
    }

    // 关闭流
    public void close() throws IOException {
        byte[] buffer;
        while ( (buffer = buf) != null) {
          	// CAS操作将缓冲数组置空
            if (bufUpdater.compareAndSet(this, buffer, null)) {
                InputStream input = in;
                in = null;
                if (input != null)
                    input.close();
                return;
            }
            // Else retry in case a new buf was CASed in fill()
        }
    }
}
```

