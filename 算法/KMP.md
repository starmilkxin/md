# KMP

分为两步

- 构建 前缀数组。
- 利用 前缀数组 来匹配 原串 和 匹配串。

![KMP](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/KMP.png)

## 构建前缀数组

前缀数组 是为了在 原串 和 匹配串 匹配失败时，可以将 匹配串 快速跳转到指定索引再次匹配，而不是一个个索引依次尝试匹配。

![KMP2](https://cdn.jsdelivr.net/gh/starmilkxin/picturebed/img/KMP2.png)

依靠前缀数组，可以快速知道 以当前匹配字符前一个字符结尾的字符串所匹配的前缀大小。

当匹配字符失败时，我们不能直接将 匹配串 的 j 置为0继续匹配，因为已经匹配的范围中可能有符合的前缀，所以当没有 前缀数组 时，需要重新一个个开始匹配。

当有 前缀述责时，可以直接获得已经匹配的范围中 范围更小的符合的前缀。这也是为什么 j = prefixList[j-1];

```Java
public int[] getPrefixList(String str) {
    int[] prefixList = new int[str.length()];
    for (int i=1, j=0; i < str.length(); i++) {
        while (j > 0 && str.charAt(i) != str.charAt(j)) {
            j = prefixList[j-1];
        }
        if (str.charAt(i) == str.charAt(j)) {
            j++;
        }
        prefixList[i] = j;
    }
    return prefixList;
}
```

## 匹配原串和匹配串

当匹配失败时，无需移动原串中的已匹配索引 i，只需要根据 前缀数组 调整匹配串当前匹配索引 j。

```java
public int strStr(String haystack, String needle) {
    int[] prefixList = getPrefixList(needle);
    for (int i=0, j=0; i < haystack.length(); i++) {
        while (j > 0 && haystack.charAt(i) != needle.charAt(j)) {
            j = prefixList[j-1];
        }
        if (haystack.charAt(i) == needle.charAt(j)) {
            j++;
        }
        if (j == needle.length()) {
            return i - j + 1;
        }
    }
    return -1;
}
```

## 整体

```java
class Solution {
    public int strStr(String haystack, String needle) {
        int[] prefixList = getPrefixList(needle);
        for (int i=0, j=0; i < haystack.length(); i++) {
            while (j > 0 && haystack.charAt(i) != needle.charAt(j)) {
                j = prefixList[j-1];
            }
            if (haystack.charAt(i) == needle.charAt(j)) {
                j++;
            }
            if (j == needle.length()) {
                return i - j + 1;
            }
        }
        return -1;
    }

    public int[] getPrefixList(String str) {
        int[] prefixList = new int[str.length()];
        for (int i=1, j=0; i < str.length(); i++) {
            while (j > 0 && str.charAt(i) != str.charAt(j)) {
                j = prefixList[j-1];
            }
            if (str.charAt(i) == str.charAt(j)) {
                j++;
            }
            prefixList[i] = j;
        }
        return prefixList;
    }
}
```