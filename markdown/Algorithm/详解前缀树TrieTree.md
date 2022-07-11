# 详解前缀树「TrieTree」

[208. 实现 Trie (前缀树)](https://leetcode.cn/problems/implement-trie-prefix-tree/)

[648. 单词替换](https://leetcode.cn/problems/replace-words/)

[211. 添加与搜索单词 - 数据结构设计](https://leetcode.cn/problems/design-add-and-search-words-data-structure/)

[677. 键值映射](https://leetcode.cn/problems/map-sum-pairs/)

[676. 实现一个魔法字典](https://leetcode.cn/problems/implement-magic-dictionary/)



「前缀树」又叫「字典树」或「单词查找树」，总之它们是一个意思！

「前缀树」的应用场景：给定一个字符串集合构建一棵前缀树，然后给一个字符串，判断前缀树中是否存在该字符串或者该字符串的前缀

可以结合题目 **[单词替换](https://leetcode.cn/problems/replace-words/)** 理解！我们需要根据`dictionary`构建前缀树，然后判断`sentence`中的每个单词是否在前缀树中

### <font color=#1FA774>分析</font>

一般而言，字符串的集合都是仅由小写字母构成，所以本文章都是基于该情况展开分析！

字符串集合：`[them, zip, team, the, app, that]`。这个样例的前缀树长什么样呢？

<img src="https://cdn.jsdelivr.net/gh/LFool/image-hosting@master/20220711/184426165753626611fhED1.svg" alt="1" style="zoom:80%;" />

由于都是小写字母，所以对于每个节点，均有 26 个孩子节点，上图中没有画出来，省略了而已...，但是要记住：**<font color='red'>每个节点均有 26 个孩子节点</font>**

**<font color='red'>还有一个点要明确：节点仅仅表示从根节点到本节点的路径构成的字符串是否有效而已</font>**

对于上图中橙色的节点，均为有效节点，即：从根节点到橙色节点的路径构成的字符串均在集合中

如果现在要找前缀`te`是否存在，分两步：

- 首先看看表示`te`字符串的路径是否存在，这个例子是存在的
- 其次看看该路径的终点处的节点是否有效，很遗憾，此处为白色，无效
- 所以前缀`te`不存在！！

### <font color=#1FA774>数据结构</font>

现在看看如何表示这棵「前缀树」，即数据结构该如何定义。其实就是一棵多叉树，有 26 个孩子节点的多叉树而已！！

现在来思考节点的值又该如何表示呢？

在上面的例子中，节点仅仅表示路径构成的字符串是否有效而已，所以节点可以用`boolean`类型来表示

还有一类情况就是每个字符串都有一个权值，所以节点的值可以用一个数值来表示

```java
// 前缀树的数据结构
class TrieNode {
    boolean val;
    TrieNode[] children = new TrieNode[26];
}
// 前缀树的根节点
private TrieNode root = new TrieNode();
```

### <font color=#1FA774>常用操作</font>

根据上面的分析，其实「前缀树」常用操作就三种

- 根据所给字符串集合构建前缀树
- 判断前缀树中是否存在目标字符串
- 在前缀树中找出目标字符串的最短前缀

#### <font color=#9933FF>构建前缀树</font>

最初，我们只有一个根节点`root`，孩子节点也都还没初始化！

所以直接看代码：

```java
// 往前缀树中插入一个新的字符串
public void insert(String word) {
    TrieNode p = root;
    for (char c : word.toCharArray()) {
        // char -> int
        int i = c - 'a';
        // 初始化孩子节点
        if (p.children[i] == null) p.children[i] = new TrieNode();
        // 节点下移
        p = p.children[i];
    }
    // 此时 p 指向目标字符串的终点
    p.val = true;
}
```

为了扩展思维，这里再给出递归的实现方法：(和树的遍历很像)

```java
public TrieNode insert(TrieNode node, String word, int index) {
    // 初始化
    if (node == null) node = new TrieNode();
    // 到了终点
    if (index == word.length()) {
        node.val = true;
        return node;
    }
    int i = word.charAt(index) - 'a';
    node.children[i] = insert(node.children[i], word, index + 1);
    return node;
}
// 调用方法
root = insert(root, word, 0);
```

#### <font color=#9933FF>寻找目标字符串</font>

当「前缀树」构建好了后，寻找目标字符串也就大同小异了

复习一下寻找的两个步骤：

- 首先看看表示字符串的路径是否存在
- 其次看看该路径的终点处的节点是否有效

```java
public boolean query(String word) {
    TrieNode p = root;
    for (char c : word.toCharArray()) {
        int i = c - 'a';
        // 路径不存在的情况，直接返回 false
        if (p.children[i] == null) return false;
        p = p.children[i];
    }
    // 路径存在，直接返回该路径的终点处的节点的有效性
    return p.val;
}
```

同样的，为了扩展思维，这里再给出递归的实现方法：(和树的遍历很像)

```java
public boolean query(TrieNode node, String word, int index) {
    // 路径不存在的情况
    if (node == null) return false;
    // 路径存在，直接返回该路径的终点处的节点的有效性
    if (index == word.length()) return node.val;
    
    int i = word.charAt(index) - 'a';
    return query(node.children[i], word, index + 1);
}
// 调用方法
query(root, word, 0);
```

#### <font color=#9933FF>寻找最短前缀</font>

和「寻找目标字符串」差不多，但又有些许不同

- 「寻找目标字符串」必须遍历到目标字符串的末尾，然后再判断路径是否有效

- 「寻找最短前缀」只要在遍历的过程有，首次出现了有效路径，即为找到！！

```java
public String shortestPrefixOf(String word) {
    TrieNode p = root;
    StringBuffer sb = new StringBuffer();
    for (char c : word.toCharArray()) {
        int i = c - 'a';
        // 首次遇到有效路径，直接返回
        if (p.val) return sb.toString();
        ans.append(c);
        // 路径不存在的情况，直接返回 ""
        if (p.children[i] == null) return "";
        p = p.children[i];
    }
    // 没找到
    return "";
}
```

### <font color=#1FA774>高级操作</font>

#### <font color=#9933FF>含有通配符的寻找</font>

顾名思义，`.`可以表示任何字符。比如：`a.c`是可以和`[abc, aec]`匹配的

```java
public boolean keysWithPattern(TrieNode node, String pattern, int index) {
    if (node == null) return false;
    if (index == key.length()) return node.val;
    int i = key.charAt(index) - 'a';
    // 如果是通配符，直接和 26 个字母匹配 (简单粗暴！！)
    if (key.charAt(index) == '.') {
        for (int j = 0; j < 26; j++) {
            if (search(node.children[j], key, index + 1)) return true;
        }
        return false;
    } else {
        return search(node.children[i], key, index + 1);
    }
}
```

### <font color=#1FA774>题目实战</font>

#### <font color=#9933FF>实现 Trie (前缀树)</font>

**题目详情可见 [实现 Trie (前缀树)](https://leetcode.cn/problems/implement-trie-prefix-tree/)**

```java
class Trie {
    class TrieNode {
        boolean val;
        TrieNode[] children = new TrieNode[26];
    }

    private TrieNode root;

    public Trie() {
        root = new TrieNode();
    }
    
    public void insert(String word) {
        TrieNode p = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (p.children[i] == null) p.children[i] = new TrieNode();
            p = p.children[i];
        }
        p.val = true;
    }
    
    public boolean search(String word) {
        TrieNode p = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (p.children[i] == null) return false;
            p = p.children[i];
        }
        return p.val;
    }
    
    public boolean startsWith(String prefix) {
        TrieNode p = root;
        for (char c : prefix.toCharArray()) {
            int i = c - 'a';
            if (p.children[i] == null) return false;
            p = p.children[i];
        }
        return true;
    }
}
```

#### <font color=#9933FF>单词替换</font>

**题目详情可见 [单词替换](https://leetcode.cn/problems/replace-words/)**

```java
class Solution {
    class TrieNode {
        boolean val;
        TrieNode[] children = new TrieNode[26];
    }
    
    private TrieNode root = new TrieNode();
    
    public String replaceWords(List<String> dictionary, String sentence) {
        for (String d : dictionary) root = insert(root, d, 0);
        StringBuffer ans = new StringBuffer();
        for (String s : sentence.split(" ")) {
            String q = query(s);
            if ("".equals(q)) ans.append(s).append(" ");
            else ans.append(q).append(" ");
        }
        ans.deleteCharAt(ans.length() - 1);
        return ans.toString();
    }
    
    private TrieNode insert(TrieNode node, String key, int index) {
        if (node == null) node = new TrieNode();
        if (index == key.length()) {
            node.val = true;
            return node;
        }
        int i = key.charAt(index) - 'a';
        node.children[i] = insert(node.children[i], key, index + 1);
        return node;
    }
    
    private String query(String key) {
        TrieNode p = root;
        StringBuffer ans = new StringBuffer();
        for (char c : key.toCharArray()) {
            int i = c - 'a';
            if (p.val) return ans.toString();
            ans.append(c);
            if (p.children[i] == null) return "";
            p = p.children[i];
        }
        return "";
    }
}
```

#### <font color=#9933FF>添加与搜索单词 - 数据结构设计</font>

**题目详情可见 [添加与搜索单词 - 数据结构设计](https://leetcode.cn/problems/design-add-and-search-words-data-structure/)**

```java
class WordDictionary {

    class TrieNode {
        boolean val;
        TrieNode[] children = new TrieNode[26];
    }

    private TrieNode root;

    public WordDictionary() {
        root = new TrieNode();
    }
    
    public void addWord(String word) {
        TrieNode p = root;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (p.children[i] == null) p.children[i] = new TrieNode();
            p = p.children[i];
        }
        p.val = true;
    }
    
    public boolean search(String word) {
        return search(root, word, 0);
    }

    private boolean search(TrieNode node, String key, int index) {
        if (node == null) return false;
        if (index == key.length()) return node.val;
        int i = key.charAt(index) - 'a';
        if (key.charAt(index) == '.') {
            for (int j = 0; j < 26; j++) {
                if (search(node.children[j], key, index + 1)) return true;
            }
            return false;
        } else {
            return search(node.children[i], key, index + 1);
        }
    }
}
```

#### <font color=#9933FF>键值映射</font>

**题目详情可见 [键值映射](https://leetcode.cn/problems/map-sum-pairs/)**

这个题目些许不同，每个节点表示的不再是是否有效，而是一个值

```java
class MapSum {

    class TrieNode {
        int val;
        TrieNode[] children = new TrieNode[26];
    }

    private TrieNode root;

    public MapSum() {
        root = new TrieNode();
    }
    
    public void insert(String key, int val) {
        TrieNode p = root;
        for (char c : key.toCharArray()) {
            int i = c - 'a';
            if (p.children[i] == null) p.children[i] = new TrieNode();
            p = p.children[i];
        }
        p.val = val;
    }
    
    public int sum(String prefix) {
        TrieNode p = root;
        // 找到前缀 prefix 的最后一个节点
        for (char c : prefix.toCharArray()) {
            int i = c - 'a';
            if (p.children[i] == null) return 0;
            p = p.children[i];
        }
        return getAllSum(p);
    }
    // 辅助函数，求以 node 为根节点的子树的节点和 
    private int getAllSum(TrieNode node) {
        if (node == null) return 0;
        int sum = 0;
        for (int i = 0; i < 26; i++) {
            sum += getAllSum(node.children[i]);
        }
        return sum + node.val;
    }
}
```

#### <font color=#9933FF>实现一个魔法字典</font>

**题目详情可见 [实现一个魔法字典](https://leetcode.cn/problems/implement-magic-dictionary/)**

由于本题目必须替换一次，所以采取了一个傻办法：遍历每一种替换的情况

```java
class MagicDictionary {

    class TrieNode {
        boolean val;
        TrieNode[] children = new TrieNode[26];
    }

    private TrieNode root;

    public MagicDictionary() {
        root = new TrieNode();
    }
    
    public void buildDict(String[] dictionary) {
        for (String word : dictionary) {
            TrieNode p = root;
            for (char c : word.toCharArray()) {
                int i = c - 'a';
                if (p.children[i] == null) p.children[i] = new TrieNode();
                p = p.children[i];
            }
            p.val = true;
        }
    }
    
    public boolean search(String searchWord) {
        // 遍历每一种替换的情况
        for (int i = 0; i < searchWord.length(); i++) {
            if (search(root, searchWord, 0, i)) return true;
        }
        return false;
    }

    private boolean search(TrieNode node, String searchWord, int index, int changeId) {
        if (node == null) return false;
        if (index == searchWord.length()) return node.val;
        int i = searchWord.charAt(index) - 'a';
        if (index == changeId) {
            for (int j = 0; j < 26; j++) {
                if (j == i) continue;
                if (search(node.children[j], searchWord, index + 1, changeId)) return true;
            }
            return false;
        }
        return search(node.children[i], searchWord, index + 1, changeId);
    }
}
```

