# Java数据结构

## 概念预览

```java
// bit
1. bit ：位
一个 二进制 数据0或1，是1bit；

// byte
2. byte ：字节
存储空间的基本计量单位，如：MySQL中定义 VARCHAR(45)  即是指 45个字节；
1 byte = 8 bit


3. 一个英文字符占一个字节；
1 字母 = 1 byte = 8 bit


4. 一个汉字占2个字节；
1 汉字 = 2 byte = 16 bit
byte ：一个字节（8位）（-128~127）（-2的7次方到2的7次方-1）
short ：两个字节（16位）（-32768~32767）（-2的15次方到2的15次方-1）
int ：四个字节（32位）（一个字长）（-2147483648~2147483647）（-2的31次方到2的31次方-1）
long ：八个字节（64位）（-9223372036854774808~9223372036854774807）（-2的63次方到2的63次方-1）
float ：四个字节（32位）（3.402823e+38 ~ 1.401298e-45）（e+38是乘以10的38次方，e-45是乘以10的负45次方）
double ：八个字节（64位）（1.797693e+308~ 4.9000000e-324
```

```java
// 16进制表示范围
从0x00到0xFF

// 二进制转十进制转十六进制
二进制4位一组，遵循8,4,2,1的规律
例如: 1010
    从最高位开始算，数字大小是8*1+4*0+2*1+1*0 = 10，那么十进制就是 10
    那么十六进制就是 A (0xA)
二进制数要转换为十六进制，就是以4位一段，分别转换为十六进制

```

> **16位数对应进制的表示方法**

| 10进制 | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | 10   | 11   | 12   | 13   | 14   | 15   |
| ------ | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 16进制 | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    | 8    | 9    | A    | B    | C    | D    | E    | F    |
| 2进制  | 0000 | 0001 | 0010 | 0011 | 0100 | 0101 | 0110 | 0111 | 1000 | 1001 | 1010 | 1011 | 1100 | 1101 | 1110 | 1111 |

### 演示进制转换demo

```java
package com.test;  
import java.util.Arrays;  
public class Test { 
 
    /**
     * Hex转byte,hex只能含两个字符，如果是一个字符byte高位会是0
     */
    public static byte hexStrTobyte(String hex) {
        return (byte) Integer.parseInt(hex, 16);
    }
 
    /** 
     * 将byte转换为一个长度为8的byte数组，数组每个值代表bit 
     */  
    public static byte[] getBooleanArray(byte b) {  
        byte[] array = new byte[8];  
        for (int i = 7; i >= 0; i--) {  
            array[i] = (byte)(b & 1);  
            b = (byte) (b >> 1);  
        }  
        return array;  
    }
  
    /** 
     * 把byte转为字符串的bit 
     */  
    public static String byteToBit(byte b) {  
        return ""  
                + (byte) ((b >> 7) & 0x1) + (byte) ((b >> 6) & 0x1)  
                + (byte) ((b >> 5) & 0x1) + (byte) ((b >> 4) & 0x1)  
                + (byte) ((b >> 3) & 0x1) + (byte) ((b >> 2) & 0x1)  
                + (byte) ((b >> 1) & 0x1) + (byte) ((b >> 0) & 0x1);  
    }
  
    public static void main(String[] args) {  
        byte b = 0x35; // 0011 0101  
        //byte b = hexStrTobyte("35")
        // 输出 [0, 0, 1, 1, 0, 1, 0, 1]  
        System.out.println(Arrays.toString(getBooleanArray(b)));  
        // 输出 00110101  
        System.out.println(byteToBit(b));  
        // JDK自带的方法，会忽略前面的 0  
        System.out.println(Integer.toBinaryString(0x35));  
    }  
}  
```

