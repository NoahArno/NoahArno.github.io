# Manacher

> 在一个字符串中最长回文子串的大小

暴力：从左到右遍历，依次为对称轴往两边扩，但是还存在虚轴的问题，可以将每个字符中间都垫上一个特殊字符，就能解决，将结果除以2就行 7/2 = 3。这个特殊字符是什么都无所谓

用Manacher算法来解决这个问题就比较优秀了，首先得明白以下概念：

回文半径数组：将每个字符为轴的回文数的回文半径给放到一个数组中

回文最右边界 int R：回文子串中最右侧的位置，

中心：最右边界为R的回文子串的中心

整体流程如下：

1、如果i在R外，那么就按照暴力来算

2、如果i在R内，包括在R上，就一定存在在以c为中心，L为左边界，R为有边界，i存在一个i撇

​	2.1、如果i撇的回文区域在c的回文区域内部，此时i的回文区域就是i撇的回文区域

​	2.2、如果i撇的回文区域左侧和c的回文区域左侧刚好重合，就从R开始试试能不能往外扩

​	2.3、 如果在外部：它的区域就是i到R之间的区域

```java
public static int manacher(String s) {
    if (s == null || s.length() == 0) {
        return 0;
    }
    // 123 -> #1#2#3#
    char[] str = manacherString(s);
    // 回文半径的大小
    int[] parr = new int[str.length];
    // 中心
    int C = -1;
    // 最右的扩成功位置的再下一个位置
    int R = -1;
    int max = Integer.MIN_VALUE;
    for (int i = 0; i < str.length; i++) {
        // R第一个违规的位置 i >= R
        // i位置扩出来的答案。i位置扩的区域至少是多大
        // 2*C - i j
        pArr[i] = R > i ? Math.min(pArr[2 * C - i], R - i) : 1;
        while (i + pArr[i] < str.length && i - pArr[i] > -1) {
            if (str[i + pArr[i]] == str[i - pArr[i]])
                pArr[i]++;
            else {
                break;
            }
        }
        // 如果刷新了R，就让R变得更远
        if (i + pArr[i] > R) {
            R = i + pArr[i];
            C = i;
        }
        max = Math.max(max, pArr[i]);
    }
    return max - 1;
}

public static char[] manacherString(String str) {
    char[] charArr = str.toCharArray();
    char[] res = new char[str.length() * 2 + 1];
    int index = 0;
    for (int i = 0; i != res.length; i++) {
        res[i] = (i & 1) == 0 ? '#' : charArr[index++];
    }
    return res;
}
```

# AddShortestEnd

> 给定str，只能在后面添加字符串，让他整体变成回文数，要最少
>

思路：找出以str最右侧的那个字符为结尾的回文子串，让其中心在最左

```java
public static String shortestEnd(String s) {
    if (s == null || s.length() == 0) {
        return null;
    }
    char[] str = manacherString(s);
    int[] pArr = new int[str.length];
    int C = -1;
    int R = -1;
    int maxContainsEnd = -1;
    for (int i = 0; i != str.length; i++) {
        pArr[i] = R > i ? Math.min(pArr[2 * C - i], R - i) : 1;
        while (i + pArr[i] < str.length && i - pArr[i] > -1) {
            if (str[i + pArr[i]] == str[i - pArr[i]])
                pArr[i]++;
            else {
                break;
            }
        }
        if (i + pArr[i] > R) {
            R = i + pArr[i];
            C = i;
        }
        if (R == str.length) {
            maxContainsEnd = pArr[i];
            break;
        }
    }
    char[] res = new char[s.length() - maxContainsEnd + 1];
    for (int i = 0; i < res.length; i++) {
        res[res.length - 1 - i] = str[i * 2 + 1];
    }
    return String.valueOf(res);
}

public static char[] manacherString(String str) {
    char[] charArr = str.toCharArray();
    char[] res = new char[str.length() * 2 + 1];
    int index = 0;
    for (int i = 0; i != res.length; i++) {
        res[i] = (i & 1) == 0 ? '#' : charArr[index++];
    }
    return res;
}
```

