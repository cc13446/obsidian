```java
class Solution {
    public int strStr(String haystack, String needle) {
        if (needle.length() == 0)
            return 0;
        int[] next = new int[needle.length()];
        getNext(next, needle);

        // j 代表下一个要匹配的子串字符
        int j = 0;
        for (int i = 0; i < haystack.length(); i++) {
            // 如果下一个要匹配的主串和字串匹配不上，也要回退j
            while (j > 0 && needle.charAt(j) != haystack.charAt(i)) j = next[j - 1];
            if (needle.charAt(j) == haystack.charAt(i))
                j++;
            // 完全匹配上了
            if (j == needle.length())
                return i - needle.length() + 1;
        }
        return -1;

    }
    // 求next数组 next[i] 的意义是i号以及之前字符组成的字符串最长相同前后子缀的长度
    private void getNext(int[] next, String s) {
        // 0号之前为空字符串
        next[0] = 0;
        // 每次新的循环开始，j代表着上一次最长相同前后子缀的长度
        // 也就是说 j - 1 就是上一次可以匹配的最长前缀的最后一个字母下标
        // 因为对于上一次相同的前后子缀，如果增加的字母相同，就得到了新的最长相同前后子缀
        // 比如对于 aaba -> aabaa
        // 前缀: a aa aab
        // 后缀: aba ba a
        // 新的前缀在后面加上后一个字母: 
        // "" -> a 
        // a -> aa 
        // aa -> aab 
        // aab -> aaba
        // 新的后缀在后面加s[3]:
        // aba -> abaa 
        // ba -> baa 
        // a -> aa 
        // "" -> a
        // 上一次最长相同的前后子缀是 a
        // 所以 j = 1 表示最长相同前后子缀 a 的长度
        // 这个最长相同前后子缀 a 的最后一个字符就是 s[j - 1] 也就是s[0]
        // 这次前缀后面增加的就是 s[j] 也就是 s[1] = a
        // 这次后缀增加的就是 s[i] 也就是 s[4] = a
        // 增加的字母相同
        // 所以 j 就是这一次需要这次最长相同前后子缀的长度比上一次加一
        // 如果增加的字母不同，那么j就要倒退
        // 比如对于 aabaa -> aabaaf
        // 前缀: a aa aab aaba -> a aa aab aaba aabaa
        // 后缀: abaa baa aa a -> abaaf baaf aaf af f
        // 对于上一次相同的子缀aa
        // 前缀增加b
        // 后缀增加f
        // 不相同了，但是还有挽救的机会，可以减少j来尝试
        // j 减少就意味着要将前缀减去后面的字母，将后缀减去前面的字母
        // 也就意味着 aab 和 aaf 的比较要变成 aa 和 af 或者 a 和 f
        // 怎么控制 j 的减少呢？
        // 我们发现 aab 和 aaf 的比较变成 aa 和 af 的时候，第一个字母a还是相同的
        // 如果不是aabaaf，而是aabaaa的话
        // aab 和 aaa 的比较变成 aa 和 aa 的比较
        // 此时增加的字母就又相同了，所以 j 的减少是有技巧的
        // 那怎么判断 j 减少多少有机会呢
        // 我们发现 aab 和 aaf 的比较要变成 aa 和 af 此时是有机会的，因为第一个字母a相同
        // 有机会的原因是因为 [a]ab 和 a[a]f 变成 [a]a 和 [a]f 括起来的字母相同
        // 假设类比为 [aba]bab 和 ab[aba]f 的话，[aba]b 和 [aba]f也是有机会的
        // 相似点在哪里呢，在于上次的最长相同前后子缀X，是拥有相同前后子缀Y的
        // 将j从X的长度减少到Y的长度，前缀从后面减字母，后缀从前面减字母
        // 此时要比较的部分，除了最后一个字符要比较以外，还是相同的，也就是Y
        // 所以减少j的方式为:
        // 将j从上次的最长相同前后子缀X的长度减少到X的最长相同前后子缀Y的长度
        // X的最长前后子缀Y的长度存在哪里呢？
        // 存在 next[len(X) - 1]
        // 也就是 j = next[j - 1]
        // j 可以回退很多次，知道匹配到为止
        // 当 j 回退到0的时候，也没有意义再回退了
        int j = 0;
        for (int i = 1; i < s.length(); i++) {
            // 每次循环开始，先看能不能接着匹配，如果不匹配就回退j
            while (j > 0 && s.charAt(j) != s.charAt(i)) j = next[j - 1];
            // 此时回退到匹配上了，或者回退到0
            if (s.charAt(j) == s.charAt(i))
                j++;
            next[i] = j;
        }
    }
}
```