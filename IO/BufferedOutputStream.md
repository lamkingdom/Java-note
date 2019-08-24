# BufferedOutputStream

## 基本信息

`BufferedOutputStream` 是缓冲输出流。它继承于`FilterOutputStream`。
`BufferedOutputStream `的作用是为另一个输出流提供“缓冲功能”。

## 源码分析

```java
public
class BufferedOutputStream extends FilterOutputStream {
    
  	// 缓冲数组
    protected byte buf[];

    // 缓冲数组中实际字节数
    protected int count;

    // 构造函数
  	// 使用默认容量创建缓冲数组
    public BufferedOutputStream(OutputStream out) {
        this(out, 8192);
    }

    // 构造函数
  	// 使用指定的容量创建缓冲数组
    public BufferedOutputStream(OutputStream out, int size) {
        super(out);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }

    // 将缓冲数组中的数据刷入输出流
  	// out为父类的成员变量
    private void flushBuffer() throws IOException {
        if (count > 0) {
            out.write(buf, 0, count);
            count = 0;
        }
    }

    // 往缓冲数组写入一个字节
    public synchronized void write(int b) throws IOException {
      	// 如果当前缓冲数组已满
      	// 则将缓冲数组写入输出流
        if (count >= buf.length) {
            flushBuffer();
        }
      	// 缓冲数组写入数据
        buf[count++] = (byte)b;
    }

    // 往缓冲数组写入一个字节数组
    public synchronized void write(byte b[], int off, int len) throws IOException {
      	// 如果需要写入的数组长度大于缓冲数组的长度
        if (len >= buf.length) {
           	// 1.将缓冲数组刷入输出流
            flushBuffer();
          	// 2.不通过缓冲数组，直接将数组b写入输出流
            out.write(b, off, len);
            return;
        }
      	// 如果len > 当前可写入的长度(并且len < buf.length)
        if (len > buf.length - count) {
          	// 刷出缓冲数组到输出流
            flushBuffer();
        }
      	// 将数组b写入到缓冲数组
        System.arraycopy(b, off, buf, count, len);
      	// 更新count
        count += len;
    }

    // 刷新缓冲
    public synchronized void flush() throws IOException {
        flushBuffer();
        out.flush();
    }
}
```

