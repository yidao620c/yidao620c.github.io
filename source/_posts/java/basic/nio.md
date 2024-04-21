---
title: NIO笔记-缓存器细节
date: 2015-06-20 18:15:42 +0800
toc: false
categories: [ java ]
tags: [ java ]
---

Buffer由数据和可以高效访问以及操纵这些数据的四个索引组成，这四个索引是：mark、position、limit、capacity。
下表是用于设置和复位索引以及查询它们的方法：

 方法                 | 说明                                          
--------------------|---------------------------------------------
 capacity()	        | 返回缓存区容量                                     
 clear()	           | 清空缓存区，position=0，limit=capacity，此方法可覆写缓存区   
 flip()	            | limit=position，position=0，用于准备从缓存区读取已经写入的数据 
 limit()	           | 返回limit的值                                   
 limit(int lim)	    | 设置limit的值                                   
 mark()	            | 将mark设置为position                            
 position()	        | 返回position的值                                
 position(int pos)	 | 设置position的值                                
 remaining()	       | 返回(limit - position)                        
 hasRemaining()	    | 若有介于position和limit之间的元素，返回true              
 reset()	           | 将position设置为mark                            
 rewind()	          | 将position设置为0，也就是缓存区的开始位置                   

在缓存区中插入和提取数据的方法会更新这些索引，用于反映所发生的变化。
<!-- more -->

下面的示例用到一个很简单的算法 - 交换相邻字符，以对CharBuffer中的字符进行编码和译码

```java
public class UsingBuffers {
	private static void symmetricScramble(CharBuffer buffer) {
		while (buffer.hasRemaining()) {
			buffer.mark();
			char c1 = buffer.get();
			char c2 = buffer.get();
			buffer.reset();
			buffer.put(c2).put(c1);
		}
	}

	public static void main(String[] args) {
		char[] data = "UsingBuffers".toCharArray();
		ByteBuffer bb = ByteBuffer.allocate(data.length * 2);
		CharBuffer cb = bb.asCharBuffer();
		cb.put(data);
		print(cb.rewind());
		symmetricScramble(cb);
		print(cb.rewind());
		symmetricScramble(cb);
		print(cb.rewind());
	}
} /*
  * Output: UsingBuffers sUniBgfuefsr UsingBuffers
  */// :~
```

尽管可以通过对某个char数组调用wrap方法直接产生一个CharBuffer，但是在本例中取而代之的是分配一个底层的ByteBuffer，产生的CharBuffer只是ByteBuffer上的一个视图而已。这里强调的是，我们总是以操纵ByteBuffer为目标，因为只有它才可以和通道进行交互。

刚开始的时候，position指向缓存区第一个元素，capacity和limit指向最后一个元素。

调用get方法和put方法的时候，position指针向前移动1个单位，具体移动几个字节byte，要看数据占用位数大小。
有个重载的get和put方法，带索引参数，不过，这些方法不改变position指针。

mark()方法会在当前位置打个标记，为了以后reset的时候position可以返回这个remark处。

可以直接用带参的put方法去设置相应位置的值，也可以先reset到remark处后，put值。

while循环最后，position指向缓存区的末尾了。如果要打印缓存区，只能打印出position和limit之间的字符，也就是啥都没了。因此这时候要显示缓存区的内容，必须使用rewind()
把position设置到缓存区开始位置，这时候mark值变得不明确了。

**内存映射文件 - 创建和修改大文件**

先看一个瞬间创建一个128M的大文件的例子：

```java
public class LargeMappedFiles {
  static int length = 0x8FFFFFF; // 128 MB
  public static void main(String[] args) throws Exception {
    MappedByteBuffer out =
      new RandomAccessFile("test.dat", "rw").getChannel()
      .map(FileChannel.MapMode.READ_WRITE, 0, length);
    for(int i = 0; i < length; i++)
      out.put((byte)'x');
    print("Finished writing");
    for(int i = length/2; i < length/2 + 6; i++)
      printnb((char)out.get(i));
  }
} ///:~
```

为了既能读又能写，我们先由RandomAccessFile开始，获得该文件上的通道，然后调用map()
产生MappedByteBuffer，这是一种特殊类型的直接缓存器。我们必须指定映射文件的初始位置和映射区域的长度，这意味着我们可以映射某个大文件的较小的部分。

用这种方式，很大的文件可达2GB也可以很容易的修改。

相比较旧的IO流而言，映射文件访问方式可以大大的提高性能。映射写必须要用RandomAccessFile。

**文件加锁**

Java的文件加锁直接映射到了本地操作系统的加锁工具上，因此文件锁对于其他操作系统进程是可见的。

```java
public class FileLocking {
  public static void main(String[] args) throws Exception {
    FileOutputStream fos= new FileOutputStream("file.txt");
    FileLock fl = fos.getChannel().tryLock();
    if(fl != null) {
      System.out.println("Locked File");
      TimeUnit.MILLISECONDS.sleep(100);
      fl.release();
      System.out.println("Released Lock");
    }
    fos.close();
  }
} /* Output:
Locked File
Released Lock
*///:~
```

**对映射文件的部分加锁**

文件映射通常应用于极大文件，我们可能需要对这种巨大文件进行部分加锁，其他进程可以修改文件中未被加锁的部分。例如数据库文件就是这样。

```java
public class LockingMappedFiles {
  static final int LENGTH = 0x8FFFFFF; // 128 MB
  static FileChannel fc;
  public static void main(String[] args) throws Exception {
    fc =
      new RandomAccessFile("test.dat", "rw").getChannel();
    MappedByteBuffer out =
      fc.map(FileChannel.MapMode.READ_WRITE, 0, LENGTH);
    for(int i = 0; i < LENGTH; i++)
      out.put((byte)'x');
    new LockAndModify(out, 0, 0 + LENGTH/3);
    new LockAndModify(out, LENGTH/2, LENGTH/2 + LENGTH/4);
  }
  private static class LockAndModify extends Thread {
    private ByteBuffer buff;
    private int start, end;
    LockAndModify(ByteBuffer mbb, int start, int end) {
      this.start = start;
      this.end = end;
      mbb.limit(end);
      mbb.position(start);
      buff = mbb.slice();
      start();
    }
    public void run() {
      try {
        // Exclusive lock with no overlap:
        FileLock fl = fc.lock(start, end, false);
        System.out.println("Locked: "+ start +" to "+ end);
        // Perform modification:
        while(buff.position() < buff.limit() - 1)
          buff.put((byte)(buff.get() + 1));
        fl.release();
        System.out.println("Released: "+start+" to "+ end);
      } catch(IOException e) {
        throw new RuntimeException(e);
      }
    }
  }
}
```

