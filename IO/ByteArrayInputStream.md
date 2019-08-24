# ByteArrayInputStream

## 基本信息

`ByteArrayInputStream` 是字节数组输入流。它继承于`InputStream`。
它包含一个内部缓冲区，该缓冲区包含从流中读取的字节；通俗点说，它的内部缓冲区就是一个字节数组，而`ByteArrayInputStream`本质就是通过字节数组来实现的。
我们都知道，`InputStream`通过read()向外提供接口，供它们来读取字节数据；而`ByteArrayInputStream` 的内部额外的定义了一个计数器，它被用来跟踪 read() 方法要读取的下一个字节。

实际上也就是将`ByteArrayInputStream`当成一个输入流来使用。

对应`ByteArrayOutputStream`。

## 源码

```java
public
class ByteArrayInputStream extends InputStream {

    // 内部缓冲数组
    protected byte buf[];

    // 下一个会被读取的字节的索引
    protected int pos;

    // 标记位，初始为0
    protected int mark = 0;

    // 字节流的长度
    protected int count;

    // 构造器
    public ByteArrayInputStream(byte buf[]) {
      	// 初始化buf为传入的byte数组
        this.buf = buf;
      	// 初始化下一个要被读取的字节索引号为0
        this.pos = 0;
      	// 初始化字节流长度为byte数组长度
        this.count = buf.length;
    }

    // 构造器
    public ByteArrayInputStream(byte buf[], int offset, int length) {
        this.buf = buf;
      	// 初始化下一个要被读取的字节索引号为offset
        this.pos = offset;
      	// 初始化count为流的实际字节长度
        this.count = Math.min(offset + length, buf.length);
      	// 标记位设置为流最开始的位置offset
        this.mark = offset;
    }

    // 从字节数组输入流中读取一个字节
    public synchronized int read() {
      	// 如果pos >= count代表到流的末尾，返回-1
      	// 否则返回buf[pos]，pos进一
      	// 这里& 0xff是防止读取出来的为负数，进行且操作，保证得到的是正数
        return (pos < count) ? (buf[pos++] & 0xff) : -1;
    }

    // 从字节数组输入流中读取指定长度的字节到传入的字节数组b中
    public synchronized int read(byte b[], int off, int len) {
      	// null的校验
      	// off和len的检验
        if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        }
				
      	// 如果pos已经到达缓冲数组的末尾，则返回-1，代表流已达末尾，无法读取字节数
        if (pos >= count) {
            return -1;
        }
				
      	// avail = 剩余可读字节数
        int avail = count - pos;
      	
      	// 如果需要读取的长度len > 剩余可读字节数，则len = avail
        if (len > avail) {
            len = avail;
        }
      	// 如果len <= 0 代表不需要读取，返回读取0个字节
        if (len <= 0) {
            return 0;
        }
      	// 数组拷贝，从buf的pos位置拷贝长度为len的字节到数组b（从off位置开始，长度为len） 
        System.arraycopy(buf, pos, b, off, len);
      	// pos向后移动
        pos += len;
      	// 返回实际获取的字节数
        return len;
    }

    // 读取时跳过n个字节后开始读取
    public synchronized long skip(long n) {
      	// k为当前可读取的字节数
        long k = count - pos;
      	// 如果需要跳过的字节数小于可读取的字节数
        if (n < k) {
          	// 则k = n
            k = n < 0 ? 0 : n;
        }
				// 否则代表需要跳过的字节数大于可读的字节数
      	// 即n > k
      	// 故最大只能跳过剩余的字节数
      	// 移动pos，前进k位
        pos += k;
      	// 返回实际跳过的字节数
        return k;
    }

    // 流中当前可读取的字节数
    public synchronized int available() {
      	// 总字节 - 已读取的字节数 = 剩余字节数
        return count - pos;
    }

    // ByteArrayInputStream是否支持标记
    public boolean markSupported() {
        return true;
    }

    // 设置标记位为当前流的索引位
  	// 传入的int参数实际无作用
    public void mark(int readAheadLimit) {
        mark = pos;
    }

		// 将流当前的索引位置设置为标记位
    public synchronized void reset() {
        pos = mark;
    }

    public void close() throws IOException {
    }

```

