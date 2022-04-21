---
title: Java-IO流
date: 2022-04-21
Draft: false
---

简单介绍java的io流的概念

# 一、IO概念
* I/O就是input和output的缩写，是计算机调度把各个储存中（包括内存和外部储存）的数据写入写出的过程
* 所谓的外部设备可以包括硬盘文件, 网络设备, 另个程序等. 也就是当前程序之外的数据设备
* java中用流（stream）来抽象表示这么一个写入写出的功能，封装成一个类，都放在java.io这个包里面

# 二、流（stream）的概念
* stream（流）是从起源（source）到接收（sink）的有序数据。
* 如果把输入输出源比作“水桶”的话，“流”就相当于是连接他们的“管道”，这个管道的粗细、单向性等各种属性就是区分不同流的特性，流中的各种方法就是管道上的按钮，用于操作数据的流通。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200802143851170.jpg)
# 三、IO流的分类
## 1、按照流的方向
* 输入流：只能从中读取数据（主要由InputStream和Reader作为基类）
* 输出流：只能向其写入数据（主要由OutputStream和Writer作为基类）
## 2、按照流的操作颗粒度划分
* 字节流：以字节为单位，可操作任何数据（主要以InputStream和OutputStream作为基类）
* 字符流：以字符为单位，只能操作纯字符数据，比较方便（主要以Reader和Writer为基类）
## 3、按照流的角色划分
* 节点流：可以从/向一个特定的IO设备（如磁盘，网络）读/写数据的流，也叫低级流。
* 处理流：用于对一个已经存在的流进行连接和封装，通过封装后的流实现数据的读/写功能，也叫高级流。
# InputStream
InputStream是Java标准库提供的最基本的输入流。InputStream是一个抽象类，是所有输入类的超类。这个抽象类定义的最重要的方法就是int read（）这样的一个方法

>  public abstract int read() throws IOException;
>  这个方法会读取输入流的下一个字节，并返回字节表示的int值。如果己经>  读到末尾，返回-1，表示不能继续读取了

FileInputStream是InputStream的一个子类，可以用来从文件中获得数据，下面的代码演示了如何完整读取一个文件的字节：

```
public static void main(String[] args) throws IOException {
      InputStream inputStream=new FileInputStream("src/sample.txt");
        for(;;){
            int n=inputStream.read();
            if(n==-1){
                break;
            }
            System.out.println(n);
        }
        inputStream.close();
    }
```
注意：在计算机中，类似文件，网络端口这些资源，都是由操作系统统一管理，应用程序在运行的时候，如果打开了一个文件进行读写，完成后要及时关闭，以便操作系统把资源释放掉，否则，应用程序占用的资源会越来越多，不但白白占用内存还会影响其他应用程序的运行。

## 1.缓冲
在读取流的时候，一次读取一个字节并不是最高效的方法。很多流支持一次性读取多个字节到缓冲区，对于文件和网络流来说，利用缓冲区一次性读取多个字节的效率要高很多。InputStream提供两个重载方法来支持读取多个字节：
> int read(byte[] b): 读取若干字节填充到byte[]数组，返回读取的字节数
> int read(byte[] b,int off,int len): 指定byte[]数组的偏移量和最大填充数
> 
利用上述方法读取多个字节时需要先定义一个byte[]数组作为缓冲区，read()会尽可能多的读取字节到缓冲区，但不会超过缓冲区容量的大小，read（）的返回值变为实际读取的字节数。若返回-1，则表示没有更多数据了。
栗子：

```
public static void main(String[] args) throws IOException{
      InputStream inputStream=new FileInputStream("src/sample.txt");
        byte[] butter=new byte[5];
        int n;
        while((n=inputStream.read(butter))!=-1){
            System.out.println("read"+n+"bytes\n");
        }
        inputStream.close();
    }
```
## 2.阻塞
在调用InputStream的read()方法时，我们说它是阻塞的
> int n;
> n=inputStream.read();
> m=n;

执行到第二行时，必须等read（）方法返回后才可以继续。因为读取IO流比普通代码速度要慢很多，无法确定read（）方法到底要花费多长时间，所以它是阻塞的。
# OutputStream
和InputStream相反，OutputStream是Java库中提供的最基本的输出流。
和InputStream类似，OutputStream也是抽象类，它是所有输出流的超类。这个抽象类定义的最重要的方法是void write(int b):
>  public abstract void write(int b) throws IOException;

这个方法会写入一个字节到输出流。要注意，虽然传入的是int类型的参数，但只会写入一个字节，即只写入int最低8位表示字节的部分。

* 和InputStream一样，也有close（）方法，write（）也是阻塞的
* OutputStream还提供了一个flush()方法，它的作用是将缓冲区的内容真正输入到目的地。

为什么要有flush()方法？因为向网络、磁盘写入数据的时候，出于效率考虑，操作系统并不是输出一个字节就立马写入到文件或发送到网络，而是把输出的字节先放到内存的一个缓冲区（本质上也是一个byte[]数组），等到缓冲区写满，在一次性发送。对于很多IO设备，一次性写一个字节和一次性写1000个字节花费的时间几乎是一样的，所以这个flush方法的存在，就是强制把缓冲区的flush输出。

以FileOutputStream为例，演示将若干字节写入文件流：

```
public static void main(String[] args) throws IOException {
 OutputStream outputStream = new FileOutputStream("src/sample.txt");
        outputStream.write(108);
    }
```
# Filter模式
* 如果我们要给一个FileInputStream添加缓冲功能，则可以从FileInputStream派生出一个类
> BufferedFileInputStream extends FileInputstream
* 如果我们要给一个FileInputStream添加计算签名功能，则可以派生一个类
> DigestFileInputStream extends FileInputstream
* 若要添加加密解密功能，还可以派生一个类
> CipherFileInputStream extends FileInputstream

如果要给一个FileInputStream添加以上三种功能，用继承的方法实现的话，子类会比较多，实现的功能越多，子类就越复杂。因此直接使用继承的方法，为InputStream添加很多功能，根本无法控制代码的复杂度。

为了解决依赖继承导致子类数量失控的问题，JDK首先将InputStream分成两大类：
 1. 直接提供数据的基础InputStream，例如
  * FileInputStream
  * ByteArrayInputStream
 2. 提供额外附加功能的InputStream。例如
 * BufferedInputStream
 * DigestInputStream


当我们给一个基础的的InputStream附加各种功能时，我们先确定这个提供数据源的InputStream，即第一类Inputstream
例如：
> Inputstream file= new FileInputStream("test.gz");

紧接着我们希望FileInputStream能提供缓冲的功能来提高读取效率，因此我们可以用BufferedInputStream来包装这个InputStream，得到的包装类型是BufferedInputStream，他仍可视为InputStream：
> InputStream buffered= new BufferedInputStream(file);

无论包装了多少次，得到的对象始终是InputStream，我们zhijieyongInputStream来引用它，皆可以正常读取。

同理，OutputStream也是如此，这样的一种通过一个基础组件再叠加各种“附加”功能组件的模式，就叫做Filter模式。
# Reader
Reader是Java IO库提供的另一个输入流接口。和InputStream的区别是，Reader是一个字符流，以char为单位读取，而后者是一个字节流，以byte为单位。
|InputStream| Reader |
|--|--|
|字节流，byte  | 字符流，char |
|读取字节（-1，0~255):int read()|读取字符（-1，0~65535):int read()|
Reader定义了所有字符输入流的超类：
* FileReader实现了文字字符流的输入，使用时注意制定编码
* CharArrayReader和StringReader可以在内存中模拟一个字符流输入

Reader 是基于inputStream构造的：可以通过InPutStreanReader在制定编码的同时讲一个InputStream转化为Reader。
> Reader reader=new InputStreamReader(new FileInPutstream("文件名路径”，“编码”));

# Writer
Writer和OutputStream的联系与上面Reader差不多，不再赘述。