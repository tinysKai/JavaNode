## 经典算法题 -- 字典树(Trie)

#### 定义

![QQ截图20200308104440.png](http://ww1.sinaimg.cn/large/8bb38904gy1gcmbubu9mpj21gb0p749l.jpg)

Trie的核心思想是**空间换时间**。利用字符串的公共前缀来降低查询时间的开销以达到提高效率的目的。

基本特点

+ 根节点不包含字符，除根节点外每一个节点都只包含一个字符。
+ 从根节点到某一节点，路径上经过的字符连接起来，为该节点对应的字符串。
+ 每个节点的所有子节点包含的字符都不相同。

#### 实现一颗字典树

```java
/**
 * 描述: 字典树的节点
 */
public class TrieNode {
    /**
     * 是否是一个单词的标志
     */
    protected boolean isWord;
    /**
     * 此节点的值
     */
    protected char val;

    /**
     * 此节点对应的子节点
     */
    protected  TrieNode[] children = new TrieNode[26];

    public TrieNode() {}

    public TrieNode(char val) {
        this.val = val;
    }
}
```

```java
/**
 * 描述: 实现一颗Trie树
 *
 * https://leetcode.com/problems/implement-trie-prefix-tree/#/description
 *
 *思路 :
 *      声明TrieNode数据节点
 *      结构为{
 *          val值,
 *          是否为单词,
 *          子节点
 *      }
 *
 *      数据结构小技巧 :
 *          利用英文字符的char的有序性..直接算出对应的数组下标(nchar - 'a')
 */
public class Trie {
    private TrieNode root;

    public Trie() {
        root = new TrieNode(' ');
    }

    /** Inserts a word into the trie. */
    public void insert(String word) {
        TrieNode node = root;
        for (int i = 0; i < word.length(); i++) {
            char val = word.charAt(i);
            int charIndex = getCharIndex(val);
            if (node.children[charIndex] == null){
                node.children[charIndex] = new TrieNode(val);
            }
            node = node.children[charIndex];
        }
        node.isWord = true;
    }



    /** Returns if the word is in the trie. */
    public boolean search(String word) {
        return traversing(word) == null ? false : traversing(word).isWord;
    }

    /** Returns if there is any word in the trie that starts with the given prefix. */
    public boolean startsWith(String prefix) {
        return traversing(prefix) == null ? false : true;
    }

    /**
     * 迭代遍历单词并返回是否能找到这个单词Node
     */
    private TrieNode traversing(String word){
        TrieNode node = root;
        for (int i = 0; i < word.length(); i++) {
            char val = word.charAt(i);
            if (node.children[getCharIndex(val)] == null) return null;

            node = node.children[getCharIndex(val)];
        }
        return node;
    }

    /**
     * 计算字符数组下标
     */
    private int getCharIndex(char val) {
        return val - 'a';
    }



    public static void main(String[] args) {
        Trie root = new Trie();
        root.insert("apple");
        System.out.println("存在单词apple," + root.search("apple"));
        System.out.println("存在单词appl," + root.search("appl"));
        System.out.println("存在前缀appl," + root.startsWith("appl"));

    }
}
```

