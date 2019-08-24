# InputStream

## 基本信息

字节输入流。

## 源码解析

```java
public abstract class InputStream implements Closeable {

    // skip最大值
    private static final int MAX_SKIP_BUFFER_SIZE = 2048;

    // 从输入流中读取一个字节
    public abstract int read() throws IOException;

    // 传入一个byte数组
  	// 调用同类中的read(byte b[], int off, int len)方法
    public int read(byte b[]) throws IOException {
        return read(b, 0, b.length);
    }

    // 从输入流中读取长度为len byte的数据到传入的数组b
  	// 读取过程中，输入流数据读取完毕时返回-1
  	// 方法参数为：目标数组，数组下标起始值，读取长度
  	// 方法返回值：读取的字节数
    public int read(byte b[], int off, int len) throws IOException {
      	// 从数组的off位置开始，长度为len
      	// null对象检验
      	// 检测off和len
        if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }

      	// 从输入流中先读取一个字节的数据
      	// 一字节8位，因为是无符号的，即2^8 = 256，对应ASCII表
        int c = read();
      	// -1代表读取到流的尽头
        if (c == -1) {
            return -1;
        }
      	// 将读取出来的字符存储到数组b的off位置
        b[off] = (byte)c;

      	// 因为上面已经读取过一位，所以i从1开始起算（数组下标0起算）
        int i = 1;
        try {
          	// 循环读取输入流中的数据
            for (; i < len ; i++) {
                c = read();
              	// 一旦读取到的是-1，则结束读取
                if (c == -1) {
                    break;
                }
              	// 存储到相应的位置
                b[off + i] = (byte)c;
            }
        } catch (IOException ee) {
        }
      	// 返回读取的字节数
        return i;
    }

    // 跳过指定的n位后开始读取
  	// 方法返回值：跳过的字节数
    public long skip(long n) throws IOException {
				
      	// 定义一个变量，等于需要跳过的位数
      	// remaining译为剩下的
        long remaining = n;
      	// 已经读取的字节数
        int nr;
				
      	// 如果n不是正整数则代表不需要跳过，返回跳过 0 个字节
        if (n <= 0) {
            return 0;
        }
      
				// 创建一个临时的byte[]
      	// 跳过的本质实际还是读取，只是存到这个临时的数组，让流前进
      	// 获取临时数组的size（取2048或者remaining中较小的那个）
        int size = (int)Math.min(MAX_SKIP_BUFFER_SIZE, remaining);
      	// 创建临时数组
        byte[] skipBuffer = new byte[size];
      	// 当remaining > 0证明跳过过程还未结束
        while (remaining > 0) {
          	// 一次最多读取2048个字节到临时数组
          	// 如果remaining < 2048则只需要读取一次
          	// 否则需要多次进行读取
            nr = read(skipBuffer, 0, (int)Math.min(size, remaining));
          	// 如果nr < 0代表流已经读完，结束读取
            if (nr < 0) {
                break;
            }
          	// 否则remaining减去本次读取的字节数
            remaining -= nr;
        }
				
      	// 返回需要跳过的字节数 - 还未跳过的字节数 = 实际跳过的字节数
        return n - remaining;
    }

    // 可读取的数量
    public int available() throws IOException {
        return 0;
    }

    // 关闭流
    public void close() throws IOException {}

    // 标记，具体子类中实现
    public synchronized void mark(int readlimit) {}

		// InputStream不支持reset
    public synchronized void reset() throws IOException {
        throw new IOException("mark/reset not supported");
    }

  	// 是否支持标记
  	// InputStream不支持标记功能
    public boolean markSupported() {
        return false;
    }

}
```

