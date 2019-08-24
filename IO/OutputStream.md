# OutputStream

## 基本信息

字节输出流。

## 源码

```java
public abstract class OutputStream implements Closeable, Flushable {
    
  	// 往输出流写一个字节
    public abstract void write(int b) throws IOException;

    // 往输出流写入一个byte数组
  	// 调用本类的write(byte b[], int off, int len)方法
    public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
    }

    // 往输出流写入一个byte数组，从b[off]开始，长度为len
    public void write(byte b[], int off, int len) throws IOException {
      	// null检验
      	// off及len的检验
        if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
      	// 循环调用write()方法从传入的数组读取数据写入输出流
        for (int i = 0 ; i < len ; i++) {
            write(b[off + i]);
        }
    }

    public void flush() throws IOException {
    }

    public void close() throws IOException {
    }

}

```

