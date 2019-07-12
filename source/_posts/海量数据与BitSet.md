---
title: 海量数据与BitSet
date: 2018-11-12
tags: [java,算法]
categories: Java
toc: true
---

{% blockquote %}
现在有五十亿个int类型的正整数，从中找出重复的数并返回。
{% endblockquote %}

&emsp;&emsp;这种问题，首先想到的就是内存问题。50亿个int数，一共约占用4*50亿字节=20G内存，对于一般机器来说一次性加载进内存基本是不可能的。
&emsp;&emsp;因为数据量较大，如果采取遍历，那时间复杂度也需要考虑。所有数都是int类型，可以考虑用一个int数组接收所有值，把int正整数n作为数组下标，如果n存在，则对应的值为1，如果不存在，对应的值为0。例如数组arr[n] = 1，表示n存在，arr[n] = 0表示n不存在。由于int非负整数一共有 2^31 个，所以数组的大小需要 2^32 （约21亿）这么大。这样的话，只需要一次遍历，就可以找出所有重复的数了。这样的话，2^32的int数组，其所需内存大约是8G。
&emsp;&emsp;问题只需要找出重复值，不需要统计各重复值个数，所以这个int数组可以用boolean数组代替。
&emsp;&emsp;《Java虚拟机规范》虽然定义了boolean这种类型，但只对它提供了非常有限的支持。在Java虚拟机中没有任何供boolean值专用的字节码指令，Java语言表达式所操作的boolean值，在编译之后都使用Java虚拟机中的int数据类型来代替，而boolean数组将会被编码成Java虚拟机的byte数组，每个元素boolean元素占8位。也就是说，一个boolean值占用内存4个字节的空间；但一个boolean数组中的每一个值占用内存1个字节的空间。这样子，将int数组转换成boolean数组，占用内存可以下降到原来四分之一，也就是2G内存。即使这样，内存要求还是太大了。
&emsp;&emsp;虽然boolean是表示两种状态，但是boolean实际上占用了8bit，按道理8bit是可以表示128种状态的，但现在只能表示两种状态，还是太浪费了。既然这样，不如再大胆一点，直接用一个字节表示状态。

### BitMap算法
&emsp;&emsp;用1位来表示一个数据是否出现过，0为没有出现过，1表示出现过。使用的时候可以根据某一个位是否为0表示此数是否出现过，这种算法称做bitmap算法。JDK中的BitSet集合对是就是对Bitmap的相对简单的实现

### Java中BitSet实现
{% codeblock  lang:java  %}
public class BitSet implements Cloneable, java.io.Serializable {
    private final static int ADDRESS_BITS_PER_WORD = 6;
    private final static int BITS_PER_WORD = 1 << ADDRESS_BITS_PER_WORD;
    private final static int BIT_INDEX_MASK = BITS_PER_WORD - 1;
    private static final long WORD_MASK = 0xffffffffffffffffL;
//bitset的内部实现是long数组
private long[] words;
//bitSet当前word数，也即words长度
private transient int wordsInUse = 0;

//BitSet默认的是一个long整形的大小
public BitSet() {
    initWords(BITS_PER_WORD);
    sizeIsSticky = false;
}

public BitSet(int nbits) {
    // nbits can't be negative; size 0 is OK
    if (nbits < 0)
        throw new NegativeArraySizeException("nbits < 0: " + nbits);

    initWords(nbits);
    sizeIsSticky = true;
}
//初始化words
private void initWords(int nbits) {
    words = new long[wordIndex(nbits-1) + 1];
}
//根据bit数计算对应word数，右移6位
private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
//私有构造
private BitSet(long[] words) {
    this.words = words;
    this.wordsInUse = words.length;
    checkInvariants();
}
}
{% endcodeblock %}

&emsp;&emsp;BitSet的底层实现是使用long数组作为内部存储结构的，这就决定了BitSet的大小为long类型大小(64位)的整数倍。long数组的每一个元素都可以当做是64位的二进制数，也是整个BitSet的子集。在BitSet中把这些子集叫做Word。
&emsp;&emsp;BitSet对外提供了两个公有构造函数，一个无参，默认的初始大小为64bit，另一个构造函数带一个int型参数用于指定大小。默认情况下，每个位的默认值为false(0)。


### BitSet的基本操作
&emsp;&emsp;BitSet对数据的基本操作都是通过位操作来实现的。set()方法是其中的核心。当要增加一个元素时，修改该元素在BitSet中的对应位值为1；当要删除一个元素时，修改该元素在BitSet中的对应位值为0。
#### 新增
{% codeblock lang:java  %}
public void set(int bitIndex, boolean value) {
    if (value)
        set(bitIndex);
    else
        clear(bitIndex);
}

public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
    //获取所在下标
    int wordIndex = wordIndex(bitIndex);
    expandTo(wordIndex);
    //与
    words[wordIndex] |= (1L << bitIndex); // Restores invariants

    checkInvariants();
}
{% endcodeblock %}

&emsp;&emsp;举个栗子，我们要添加一个值n=84。
&emsp;&emsp;先找到n在words数组中的下标index，index = 1。然后再找到n在words[index]中的位置position，这里position = 84%64=20。
{% codeblock  lang:java  %}
index = n / 64 = n >> 6。
position = n % 64 = n & 0x40。
{% endcodeblock %}
&emsp;&emsp;接下来我们把1向←移动position个二进制位，然后把所得的结果和arr[index]做“或(or)”操作就可以了（位移超过64位溢出自动舍弃，1<<65=1<<1），所以对于long类型组成的words[]来说，position的位置可以不用计算：
{% codeblock  lang:java  %}
 words[wordIndex] |= (1L << n)
{% endcodeblock %}
&emsp;&emsp;除此之外，添加元素时还要考虑数组扩容问题:基础扩容倍数是2，扩容后的数组大小取2*当前数组大小与所需大小的最大值。
{% codeblock  lang:java  %}
private void expandTo(int wordIndex) {
    int wordsRequired = wordIndex+1;
    //
    if (wordsInUse < wordsRequired) {
        ensureCapacity(wordsRequired);
        wordsInUse = wordsRequired;
    }
}
private void ensureCapacity(int wordsRequired) {
    if (words.length < wordsRequired) {
        // Allocate larger of doubled size or required size
        int request = Math.max(2 * words.length, wordsRequired);
        words = Arrays.copyOf(words, request);
        sizeIsSticky = false;
    }
}
{% endcodeblock %}

#### 删除
{% codeblock lang:java  %}
//删除某个数据
public void clear(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    int wordIndex = wordIndex(bitIndex);
    if (wordIndex >= wordsInUse)
        return;

    words[wordIndex] &= ~(1L << bitIndex);

    //重新计算数组大小
    recalculateWordsInUse();
    checkInvariants();
}

private void recalculateWordsInUse() {
    // Traverse the bitset until a used word is found
    int i;
    for (i = wordsInUse-1; i >= 0; i--)
        if (words[i] != 0)
            break;

    wordsInUse = i+1; // The new logical size
}
//清空所有数据
 public void clear() {
        while (wordsInUse > 0)
            words[--wordsInUse] = 0;
}
{% endcodeblock %}

&emsp;&emsp;只需要把对应的二进制的1变成0即可，把1←移后的结果取反，然后与arr[index]做“与”操作，每次删除后会重新计算数组大小。

#### 查询
{% codeblock lang:java  %}
public boolean get(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    checkInvariants();

    int wordIndex = wordIndex(bitIndex);
    return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);
}
{% endcodeblock %}

&emsp;&emsp;计算出要查询的数对应words数组中的index，把1←移后，将结果和words[index]做“与”操作，如果结果不为0，则证明存在，否则就不存在

#### 反转
把某一位的1变成0，0变成1，是一个与1的异或(^)运算的操作
{% codeblock  lang:java  %}
public void flip(int bitIndex) {
        if (bitIndex < 0)
            throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);
 
        int wordIndex = wordIndex(bitIndex);
        expandTo(wordIndex);
 
        words[wordIndex] ^= (1L << bitIndex);
 
        recalculateWordsInUse();
        checkInvariants();
}

{% endcodeblock %}

除此之外名，BitSet的新增，清空，查询，反转都提供了批量操作指定范围数据的方法

### BitSet的优缺点
优点
    1. 运算效率高，不进行比较和移位；
    2. 占用内存少，比如最大的数MAX=10000000；只需占用内存为MAX/8=1250000Byte=1.25M。

缺点
    1. 所有的数据不能重复，即不可对重复的数据进行排序。（少量重复数据查找还是可以的，用2-bitmap）。
    2. 当数据类似（1，1000，10万）只有3个数据的时候，用bitmap时间复杂度和空间复杂度相当大，只有当数据比较密集时才有优势。

### 应用
回到先前的问题，五十亿个int类型的正整数，从中找出重复的数，采用bitSet存储的话，所需内存可以降到int数组的1/32，也即250M左右的内存空间即可满足要求。
{% codeblock  lang:java  %}
public static Set<Integer> getRepeats( int arr[]){
    //将重复的值存进set，防止返回重复的数
    Set<Integer> set=new HashSet<>(  );

    //BitSet 双倍扩容机制
    BitSet bitSet=new BitSet( Integer.MAX_VALUE );
    for (int i=0; i<arr.length ;i++){
        int value=arr[i];
        //判断该数是否存在bitSet里，这样一次遍历即可得出结果
        if(bitSet.get( value )){
            set.add( value );
        }else {
            bitSet.set(value, true);
        }
    }
    return set;
}
{% endcodeblock %}
### 相关问题：
{% blockquote %}
&emsp;&emsp;给40亿个不重复的unsigned int的整数，没排过序的，然后再给一个数，如何快速判断这个数是否在那40亿个数当中？
&emsp;&emsp;usinged int只有接近43亿数，用BitSet存储的话只需要0.5G内存，一个bit位代表一个unsigned int值，读入40亿个数，设置相应的bit位；读入要查询的数，查看相应bit位是否为1，为1表示存在，为0表示不存在。
&emsp;&emsp;当然，其实这里还有优化空间，可以考虑去存储那不存在的接近3亿数。一般来说，存储3亿数所需空间是小于存储40亿数据所需空间的，不过这需要先遍历找出这3亿数中的最大值与最小值，所以会增加时间复杂度。
{% endblockquote %}
{% blockquote %}
&emsp;&emsp;已知某个文件内包含一些电话号码，每个号码为8位数字，统计不同号码的个数。
&emsp;&emsp;8位最多99 999 999，大概需要99m个bit，大概10几M字节的内存即可。 （可以理解为从0-99 999 999的数字，每个数字对应一个bit位，所以只需要99M个bit==1.2MBytes，这样，就用了小小的1.2M左右的内存表示了所有的8位数的电话）
{% endblockquote %}
{% blockquote %}
&emsp;&emsp;大数据量无重复数排序：位图法
&emsp;&emsp;大数据量有重复数排序：如果只有少量重复次数，可以将BitSet进行扩展，比如用两个bit位表示一个数，一位表示存在与否，一位表示重复次数（但仅限2次以内）
{% endblockquote %}
