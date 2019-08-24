# ByteArrayOutputStream

## 基本信息

`ByteArrayOutputStream` 是字节数组输出流。它继承于`OutputStream`。

## 源码分析

```java
public class ByteArrayOutputStream extends OutputStream {

    // 流中临时数组
    protected byte buf[];

    // 流中长度
    protected int count;

    // 构造函数
  	// 创建长度为32的byte数组
    public ByteArrayOutputStream() {
        this(32);
    }

    // 构造函数
  	// 创建指定长度的byte数组
    public ByteArrayOutputStream(int size) {
        if (size < 0) {
            throw new IllegalArgumentException("Negative initial size: "
                                               + size);
        }
        buf = new byte[size];
    }

    // 容量检测方法
    private void ensureCapacity(int minCapacity) {
        // minCapacity 需要的容量
      	// minCapacity > 数组长度则使用grow()方法扩容，参数为minCapacity
        if (minCapacity - buf.length > 0)
            grow(minCapacity);
    }

    // 
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    // 扩容方法
    private void grow(int minCapacity) {
        // oldCapacity为当前容量 = 当前数组长度
        int oldCapacity = buf.length;
      	// 新的容量newCapacity = oldCapacity * 2
        int newCapacity = oldCapacity << 1;
      	// 如果newCapacity - minCapacity < 0 (minCapacity * 2 > Integer.MAX_VALUE)
      	// 则newCapacity = 目前需要的容量（不进行 *2 扩容）
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
      	// 如果newCapacity介于[MAX_ARRAY_SIZE, Integer.MAX_VALUE]之间
      	// 则newCapacity的值根据minCapacity大小在区间左右端选择
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
      	// 数组复制
        buf = Arrays.copyOf(buf, newCapacity);
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }

    // 向输出流(ByteArrayOutputStream)中写入一个字节
  	// 同步方法
    public synchronized void write(int b) {
      	// 容量检测
        ensureCapacity(count + 1);
      	// 写入数组
        buf[count] = (byte) b;
      	// 计数+1
        count += 1;
    }

    // 向输出流(ByteArrayOutputStream)中写入一个字节数组
  	// 同步方法
    public synchronized void write(byte b[], int off, int len) {
        if ((off < 0) || (off > b.length) || (len < 0) ||
            ((off + len) - b.length > 0)) {
            throw new IndexOutOfBoundsException();
        }
      	// 容量检测
        ensureCapacity(count + len);
      	// 将参数数组的内容复制到ByteArrayOutputStream的buf数组中
        System.arraycopy(b, off, buf, count, len);
      	// 计数增加相应的数量
        count += len;
    }

    // 将ByteArrayOutputStream的buf数组写入到输出流中
  	// 同步方法
    public synchronized void writeTo(OutputStream out) throws IOException {
        out.write(buf, 0, count);
    }

    // 重置流的位置
  	// 同步方法
    public synchronized void reset() {
        count = 0;
    }

    // 将“字节数组输出流”转换成字节数组
  	// byte toByteArray()[] = byte[] toByteArray()
    public synchronized byte toByteArray()[] {
        return Arrays.copyOf(buf, count);
    }

    // 返回流中的字节数
  	// 同步方法
    public synchronized int size() {
        return count;
    }

    // 以字符串的形式输出流中所有的字节
  	// 同步方法
    public synchronized String toString() {
        return new String(buf, 0, count);
    }

    // 使用指定的字符集输出流中所有的字节
  	// 同步方法
    public synchronized String toString(String charsetName)
        throws UnsupportedEncodingException
    {
        return new String(buf, 0, count, charsetName);
    }

    // 过时方法
    @Deprecated
    public synchronized String toString(int hibyte) {
        return new String(buf, hibyte, 0, count);
    }

    // 关闭流
    public void close() throws IOException {
    }

}
```

