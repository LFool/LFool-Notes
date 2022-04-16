[toc]

# 二叉搜索树（BST）

## 定义

1. 若任意节点的左子树不空，则左子树上所有节点的值均小于它的根节点的值
2. 若任意节点的右子树不空，则右子树上所有节点的值均大于它的根节点的值
3. 任意节点的左、右子树也分别为二叉查找树

## 增删改查

> 淋漓尽致的体现了递归的思维
>
> 明确 「当前节点」「该做什么」「什么时候做」「注意返回值」

### [BST--插入](https://leetcode-cn.com/problems/insert-into-a-binary-search-tree/)

```java
// 当前节点：root
// 该做什么：判断是否为空，如果为空，即为插入位置；生成一个新节点，并返回
// 什么时候做：先序遍历
// 返回值：返回更新后的 root 节点 「root.left = 递归处理左子树后返回的节点」
public TreeNode insertIntoBST(TreeNode root, int val) {
    // 如果当前节点为 null，说明应该在此处插入节点
    if (root == null) return new TreeNode(val);
    // 递归处理左右子树
    if (root.val > val) root.left = insertIntoBST(root.left, val);
    if (root.val < val) root.right = insertIntoBST(root.right, val);
    
    return root;
}
```

### [BST--删除](https://leetcode-cn.com/problems/delete-node-in-a-bst/)

```java
// 当前节点：root
// 该做什么：是否找到 / 找到后根据孩子节点情况删除节点
// 什么时候做：先序遍历
// 返回值：返回更新后的 root 节点 「root.left = 递归处理左子树后返回的节点」
public TreeNode deleteNode(TreeNode root, int key) {
    // 没有找到
    if (root == null) return null;
    // 找到了
    if (key == root.val) {
        // root 为叶子节点
        if (root.left == null && root.right == null) return null;
        // root 只有一个左孩子
        if (root.right == null) return root.left;
        // root 只有一个右孩子
        if (root.left == null) return root.right;
        // root 有两个孩子节点
        // 1. 找到左子树最大值和 root 交换 | 2. 找到右子树最小值和 root 交换
        TreeNode minNode = getMinNode(root.right);
        // 注意：下面两行的顺序不能颠倒
        minNode.right = deleteNode(root.right, minNode.val);
        minNode.left = root.left;
        return minNode;
    } else if (key < root.val) {
        root.left = deleteNode(root.left, key);
    } else if (key > root.val) {
        root.right = deleteNode(root.right, key);
    }
    return root;
}
// 获得右子树最小值节点
private TreeNode getMinNode(TreeNode root) {
    TreeNode p = root;
    while (p.left != null) p = p.left;
    return p;
}
```

### [BST--搜索](https://leetcode-cn.com/problems/search-in-a-binary-search-tree/)

```java
// 当前节点：root
// 该做什么：判断 root.val 和 val 的关系
// 什么时候做：先序遍历
// 返回值：如果找到，返回该节点；如果没有找到，返回 null
public TreeNode searchBST(TreeNode root, int val) {
    if (root == null) return null;
    if (val == root.val) return root;
    else if (val < root.val) return searchBST(root.left, val);
    else return searchBST(root.right, val);
}
```

## 典型题目

### 验证/恢复/构建 BST

#### [验证 BST](https://leetcode-cn.com/problems/validate-binary-search-tree/submissions/)

```java
// 当前节点：root
// 该做什么：判断左子树是否合法 & 判断右子树是否合法 & 判断加上 root 后是否合法
// 什么时候做：后序遍历
// 返回值：long[]
public boolean isValidBST(TreeNode root) {
    if (root == null) return true;
    long[] res = isValidBSTHelper(root);
    return res[0] == 1;
}
// 辅助函数 | 后序遍历
// 返回值：int[] 大小为 3
// 1: 是否为合法 BST 「1 合法 0 非法」
// 2: 当前树的最大值
// 3: 当前树的最小值
private long[] isValidBSTHelper(TreeNode root) {
    // base
    if (root == null) return new long[]{1, Long.MIN_VALUE, Long.MAX_VALUE};
    long[] left = isValidBSTHelper(root.left);
    long[] right = isValidBSTHelper(root.right);
    long[] res = new long[3];
    // 合法情况：左子树合法 & 右子树合法 & 加上 root 后依旧合法
    if (left[0] == 1 && right[0] == 1 && root.val > left[1] && root.val < right[2]) {
        res[0] = 1;
        res[1] = Math.max(root.val, right[1]);
        res[2] = Math.min(root.val, left[2]);
    } else { // 非法
        res[0] = 0;
    }
    return res;
}
```

#### [恢复 BST](https://leetcode-cn.com/problems/recover-binary-search-tree/)

> 前提：已知只有两个节点位置被错误的交换

```java
// 递归：遍历思维（中序遍历）
// 借助全局变量
private TreeNode prev = null;
private TreeNode first = null;
private TreeNode second = null;
public void recoverTree(TreeNode root) {
    recoverTreeHelper(root);
    int tmp = first.val;
    first.val = second.val;
    second.val = tmp;
}
private void recoverTreeHelper(TreeNode root) {
    if (root == null) return ;
    recoverTreeHelper(root.left);
    if (prev != null) {
        if (root.val < prev.val) {
            if (first == null) {
                first = prev;
            }
            second = root;
        }
    }
    prev = root;
    recoverTreeHelper(root.right);
}
```

#### [有序链表转换 BST](https://leetcode-cn.com/problems/convert-sorted-list-to-binary-search-tree/)

> 类似：[有序数组转换 BST](https://leetcode-cn.com/problems/convert-sorted-array-to-binary-search-tree/)

```java
// 递归：分解子问题求解
public TreeNode sortedListToBST(ListNode head) {

    // 转换成数组
    List<Integer> list = new ArrayList<>();
    while (head != null) {
        list.add(head.val);
        head = head.next;
    }
    return sortedListToBSTHelper(list, 0, list.size() - 1);
}
// 当前节点：root
// 该做什么：根据下标创建新节点
// 什么时候做：后序遍历
// 返回值：更新后的 root 节点
private TreeNode sortedListToBSTHelper(List<Integer> list, int lo, int hi) {

    if (lo > hi) return null;
    int mid = lo - (lo - hi) / 2;

    TreeNode left = sortedListToBSTHelper(list, lo, mid - 1);
    TreeNode root = new TreeNode(list.get(mid));
    root.left = left;
    root.right = sortedListToBSTHelper(list, mid + 1, hi);

    return root;
}
// -----------------------------------------------------
// 递归：遍历思维（中序遍历）
// 借助全局变量
private ListNode cur;
public TreeNode sortedListToBST(ListNode head) {
    cur = head;
    int count = 0;
    // 计算长度
    while (head != null) {
        count++;
        head = head.next;
    }
    return sortedListToBSTHelper(0, count - 1);
}
private TreeNode sortedListToBSTHelper(int lo, int hi) {
    if (lo > hi) return null;
    int mid = lo - (lo - hi) / 2;
    TreeNode left = sortedListToBSTHelper(lo, mid - 1);
    TreeNode root = new TreeNode(cur.val);
    root.left = left;
    cur = cur.next;
    root.right = sortedListToBSTHelper(mid + 1, hi);
    return root;
}
```

#### [序列化 & 反序列化 BST](https://leetcode-cn.com/problems/serialize-and-deserialize-bst/)

```java
public class Codec {
    private int curIndex;
    // Encodes a tree to a single string.
    public String serialize(TreeNode root) {
        if (root == null) return "#";
        // 先序遍历
        return root.val + "," + serialize(root.left) + "," + serialize(root.right);        
    }
    // Decodes your encoded data to tree.
    public TreeNode deserialize(String data) {
        curIndex = 0;
        String[] split = data.split(",");
        return deserializeHelper(split);
    }
    private TreeNode deserializeHelper(String[] split) {
        if (curIndex == split.length) return null;
        if ("#".equals(split[curIndex])) {
            curIndex++;
            return null;
        }
        TreeNode root = new TreeNode(Integer.parseInt(split[curIndex]));
        curIndex++;
        root.left = deserializeHelper(split);
        root.right = deserializeHelper(split);
        return root;
    }
}
```

#### [前序遍历构造 BST](https://leetcode-cn.com/problems/construct-binary-search-tree-from-preorder-traversal/)

> 和反序列化很相似

```java
// 当前节点：preorder[lo]
// 该做什么：找出左右子树，然后递归处理
// 什么时候做：先序遍历
// 返回值：更新后的当前节点
public TreeNode bstFromPreorder(int[] preorder) {
    return bstFromPreorderHelper(preorder, 0, preorder.length - 1);
}
private TreeNode bstFromPreorderHelper(int[] preorder, int lo, int hi) {

    if (lo > hi) return null;

    TreeNode root = new TreeNode(preorder[lo]);
    int index;
    for (index = lo + 1; index <= hi; index++) {
        if (preorder[lo] < preorder[index]) break;
    }
    root.left = bstFromPreorderHelper(preorder, lo + 1, index - 1);
    root.right = bstFromPreorderHelper(preorder, index, hi);

    return root;
}
```



#### [不同的 BST](https://leetcode-cn.com/problems/unique-binary-search-trees/)

> 扩展：[不同的 BST II](https://leetcode-cn.com/problems/unique-binary-search-trees-ii/)

```java
// 该做什么：计算出该区间内的所有可能性数量（遍历根的所有可能性）
// 什么时候做：后序遍历
// 返回值：返回该区间内的所有可能性数量
// 备忘录
private int[][] emeo;
public int numTrees(int n) {
    emeo = new int[n + 1][n + 1];
    return numTreesHelper(1, n);
}
private int numTreesHelper(int lo, int hi) {
    // base case：为空也是一种情况
    if (lo > hi) return 1;
    if (emeo[lo][hi] != 0) return emeo[lo][hi];
    int res = 0;
    for (int rootIndex = lo; rootIndex <= hi; rootIndex++) {
        int leftNum = numTreesHelper(lo, rootIndex - 1);
        int rightNum = numTreesHelper(rootIndex + 1, hi);
        res += leftNum * rightNum;
    }
    emeo[lo][hi] = res;
    return res;
}
// -----------------------------------------------------
public List<TreeNode> generateTrees(int n) {
    return generateTreesHelper(1, n);
}
private List<TreeNode> generateTreesHelper(int lo, int hi) {
    List<TreeNode> res = new ArrayList<>();
    if (lo > hi) {
        res.add(null);
        return res;
    }
    for (int rootIndex = lo; rootIndex <= hi; rootIndex++) {
        List<TreeNode> leftRes = generateTreesHelper(lo, rootIndex - 1);
        List<TreeNode> rightRes = generateTreesHelper(rootIndex + 1, hi);
        for (TreeNode left : leftRes) {
            for (TreeNode right : rightRes) {
                TreeNode root = new TreeNode(rootIndex);
                root.left = left;
                root.right = right;
                res.add(root);
            }
        }
    }
    return res;
}
```

### [BST 第k小元素](https://leetcode-cn.com/problems/kth-smallest-element-in-a-bst/submissions/)

> 扩展：无序时，求第k 小/大 元素
>
> 借助 **优先队列**

```java
// 递归：遍历思维（中序遍历）
// 借助全局变量
private int rank = 0;
private int res = 0;
public int kthSmallest(TreeNode root, int k) {
    kthSmallestHelper(root, k);
    return res;
}
private void kthSmallestHelper(TreeNode root, int k) {
    if (root == null) return ;
    kthSmallestHelper(root.left, k);

    rank++;
    if (rank == k) {
        res = root.val;
        return ;
    }
    
    kthSmallestHelper(root.right, k);
}
```

### [BST 转化累加树](https://leetcode-cn.com/problems/convert-bst-to-greater-tree/)

```java
// 递归：遍历思维（逆中序遍历--右根左）
// 借助全局遍历记录和
private int sum = 0;
public TreeNode convertBST(TreeNode root) {
    if (root == null) return null;
    convertBST(root.right);
    sum += root.val;
    root.val = sum;
    convertBST(root.left);
    return root;
}
```

### [BST 最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)

> 扩展：[二叉树最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)

```java
// 当前节点：root
// 该做什么：如下
// 什么时候做：先序遍历
// 返回值：最近公共祖先节点
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    
    // 目的：使 p.val < q.val，方便后续操作
    if (p.val > q.val) return lowestCommonAncestor(root, q, p);

    // 该做什么
    if (root == null) return null;
    if (root == p || root == q) return root;
    if (root.val > p.val && root.val < q.val) return root;

    if (q.val < root.val) return lowestCommonAncestor(root.left, p, q);
    else return lowestCommonAncestor(root.right, p, q);
}

// 二叉树最近公共祖先
// 当前节点：root
// 该做什么：如下
// 什么时候做：后序遍历
// 返回值：最近公共祖先节点
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    // base case
    // 如果为空，返回 null
    if (root == null) return null;
    // 如果和 root 节点相等，返回 root
    if (root == p || root == q) return root;

    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);

    // 说明左右子树没有找到节点 p q
    if (left == null && right == null) return null;
    // 说明节点 p q 分别在左右子树中，故最近公共祖先为当前节点 root
    if (left != null && right != null) return root;
    // 说明节点 p q 在左子树或者右子树中
    return left == null ? right : left;   
}
```
