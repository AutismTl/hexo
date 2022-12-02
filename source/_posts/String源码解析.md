---
title: String源码解析
date: 2017-2-18 14:38:35
tags: Java
---
-------
# 1、String类
String为final类，所以String为不可变量，String类还实现了Serializable,Comparable,CharSequence接口。
Comparable接口有compareTo（String s）方法。
CharSequence接口有length(),charAt(int index),subSequence(int start,int end)方法。

String类中有一个final的char数组用来存放字符串，一个int类型的hash用来存放哈希值。

# 2、构造方法
<!--more-->
```java
/不含参数，基本没用
public String() {
        this.value = new char[0];
}
//参数为String类型
public String(String original) {
         this.value = original.value;
         this.hash = original.hash;
}
//参数为char数组，使用Arrays.copyOf()给value复制
public String(char value[]) {
         this.value = Arrays.copyOf(value, value.length);
     }
     
//将char数组从offset开始的长度为length的字符复制给value
public String(char value[], int offset, int count) {
         if (offset < 0) {
             throw new StringIndexOutOfBoundsException(offset);
         }
         if (count < 0) {
             throw new StringIndexOutOfBoundsException(count);
         }
         // Note: offset or count might be near -1>>>1.
         if (offset > value.length - count) {
             throw new StringIndexOutOfBoundsException(offset + count);
         }
         this.value = Arrays.copyOfRange(value, offset, offset+count);
     }
     
//将int数组从offset开始的长度为length的数据转型为字符复制给value
  public String(int[] codePoints, int offset, int count) {
         if (offset < 0) {
             throw new StringIndexOutOfBoundsException(offset);
         }
         if (count < 0) {
             throw new StringIndexOutOfBoundsException(count);
         }
         // Note: offset or count might be near -1>>>1.
         if (offset > codePoints.length - count) {
             throw new StringIndexOutOfBoundsException(offset + count);
         }
 
         final int end = offset + count;
 
         // Pass 1: Compute precise size of char[]
         int n = count;
         for (int i = offset; i < end; i++) {
             int c = codePoints[i];
             if (Character.isBmpCodePoint(c))
                 continue;
             else if (Character.isValidCodePoint(c))
                 n++;
             else throw new IllegalArgumentException(Integer.toString(c));
         }
 
         // Pass 2: Allocate and fill in char[]
         final char[] v = new char[n];
 
         for (int i = offset, j = 0; i < end; i++, j++) {
             int c = codePoints[i];
             if (Character.isBmpCodePoint(c))
                 v[j] = (char)c;
             else
                 Character.toSurrogates(c, v, j++);
         }
         this.value = v;
}
public String(byte ascii[], int hibyte, int offset, int count) {
        //检查是否越界
         checkBounds(ascii, offset, count);
         char value[] = new char[count];
 
         if (hibyte == 0) {
             for (int i = count; i-- > 0;) {
                 value[i] = (char)(ascii[i + offset] & 0xff);
             }
         } else {
             hibyte <<= 8;
             for (int i = count; i-- > 0;) {
                 value[i] = (char)(hibyte | (ascii[i + offset] & 0xff));
            }
         }
         this.value = value;
     }
//从bytes数组中的offset位置开始，将长度为length的字节，以charsetName格式编码，拷贝到value
 public String(byte bytes[], int offset, int length, String charsetName)
             throws UnsupportedEncodingException {
         if (charsetName == null)
             throw new NullPointerException("charsetName");
         checkBounds(bytes, offset, length);
         this.value = StringCoding.decode(charsetName, bytes, offset, length);
     }
     
//参数为StringBuffer（线程安全）
public String(StringBuffer buffer) {
         synchronized(buffer) {
            this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
        }
    }
//参数为StringBuilder
public String(StringBuilder builder) {
         this.value = Arrays.copyOf(builder.getValue(), builder.length());
    }
```

# 3、String常用方法
length()
```java
public int length() {
         return value.length;
     }
```

isEmpty()
```java
public boolean isEmpty() {
        return value.length == 0;
    }
```

equals(Object o)
```java
public boolean equals(Object anObject) {
       //是否同一对象
        if (this == anObject) {
            return true;
        }
        //是否为String类型数据
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            //长度是否相同
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                //从后向前判断
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }
```

compareTo(String s)
```java
public int compareTo(String anotherString) {
        int len1 = value.length;
        int len2 = anotherString.value.length;
        //取长度最小值
        int lim = Math.min(len1, len2);
        char v1[] = value;
        char v2[] = anotherString.value;
        
        int k = 0;
        //从前往后直到第lim个进行比较
        while (k < lim) {
            char c1 = v1[k];
            char c2 = v2[k];
            if (c1 != c2) {
                return c1 - c2;
            }
            k++;
     }
       //相同则比较长度
        return len1 - len2;
    }
```

hashCode()
```java
public int hashCode() {
        int h = hash;
        //hash是否计算过
        if (h == 0 && value.length > 0) {
            char val[] = value;
            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            //给hash赋值
            hash = h;
        }
        return h;
    }
```

concat(String s)
```java
ublic String concat(String str) {
       int otherLen = str.length();
       if (otherLen == 0) {
           return this;
       }
       int len = value.length;
       char buf[] = Arrays.copyOf(value, len + otherLen);
       str.getChars(buf, len);
       return new String(buf, true);
   }
```

trim():去掉字符串首尾的空格
```java
public String trim() {
        int len = value.length;
        int st = 0;
        char[] val = value;    /* avoid getfield opcode */
        while ((st < len) && (val[st] <= ' ')) {
            st++;
        }
        while ((st < len) && (val[len - 1] <= ' ')) {
            len--;
        }
        return ((st > 0) || (len < value.length)) ? substring(st, len) : this;
    }
```
intern方法是Native调用，它的作用是在方法区中的常量池里通过equals方法寻找等值的对象，如果没有找到则在常量池中开辟一片空间存放字符串并返回该对应String的引用，否则直接返回常量池中已存在String对象的引用。
```java
//native关键字说明其修饰的方法是一个原生态方法，方法对应的实现不是在当前文件，而是在用其他语言（如C和C++）实现的文件中。Java语言本身不能对操作系统底层进行访问和操作，但是可以通过JNI接口调用其他语言来实现对底层的访问。
public native String intern();
```