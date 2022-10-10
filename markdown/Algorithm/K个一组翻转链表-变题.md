# K 个一组翻转链表「变题」

[25. K 个一组翻转链表](https://leetcode.cn/problems/reverse-nodes-in-k-group/)



开头先来一个小插曲，**关于「反转链表」系列总结可见 [反转链表](./反转链表.html)**

这是一个常考题目，先直接给出代码：

```java
public ListNode reverseKGroup(ListNode head, int k) {
    if (head == null) return null;
    ListNode start = head, end = head;
    for (int i = 0; i < k; i++) {
        // 不满 k 个，直接返回 head
        if (end == null) return start;
        end = end.next;
    }
    // 翻转 [start, end)
    ListNode newHead = reverse(start, end);
    // 翻转后，start 变成了尾节点
    // 尾节点需要指向子问题的头节点
    start.next = reverseKGroup(end, k);
    // 返回当前问题的头节点
    return newHead;
}
// 翻转 [start, end) 区间的链表
private ListNode reverse(ListNode start, ListNode end) {
    ListNode prev = end, curr = start, next = null;
    while (curr != end) {
        next = curr.next;
        curr.next = prev;
        prev = curr;
        curr = next;
    }
    return prev;
}
```

### <font color=#1FA774>变题一：迭代写法</font>

**参考 [Java O(n) solution with super detailed explanation & illustration](https://leetcode.com/problems/reverse-nodes-in-k-group/discuss/183356/Java-O(n)-solution-with-super-detailed-explanation-and-illustration)**

```java
public ListNode reverseKGroup(ListNode head, int k) {
    ListNode dummy = new ListNode(-1);
    dummy.next = head;
    ListNode p = dummy;
    while (p != null) {
        ListNode node = p;
        for (int i = 0; i < k && node != null; i++) {
            node = node.next;
        }
        if (node == null) break;
        ListNode start = p.next, end = node.next;
        // reverse() 同上
        ListNode t = reverse(start, end);
        ListNode tail = p.next;
        p.next = t;
        p = tail;
    }
    return dummy.next;
}
```

### <font color=#1FA774>变题二：不足 k 个也要反转</font>

```java
// 递归
public ListNode reverseKGroup(ListNode head, int k) {
    if (head == null) return null;
    ListNode start = head, end = head;
    for (int i = 0; i < k; i++) {
        // 不满 k 个，也要反转
        if (end == null) return reverse(start, end);
        end = end.next;
    }
    // reverse() 同上
    ListNode newHead = reverse(start, end);
    start.next = reverseKGroup(end, k);
    return newHead;
}
// 迭代
public ListNode reverseKGroup(ListNode head, int k) {
    ListNode dummy = new ListNode(-1);
    dummy.next = head;
    ListNode p = dummy;
    while (p != null) {
        ListNode node = p;
        for (int i = 0; i < k && node != null; i++) {
            node = node.next;
        }
        ListNode start = p.next, end = (node == null ? null : node.next);
        // reverse() 同上
        ListNode t = reverse(start, end);
        ListNode tail = start;
        p.next = t;
        p = tail;
    }
    return dummy.next;
}
```

### <font color=#1FA774>变题三：从链尾开始 (不是从头部)，不足的不反转</font>

```java
public ListNode reverseKGroup(ListNode head, int k) {
    int n = 0;
    ListNode p = head;
    while (p != null) {
        p = p.next;
        n++;
    }
    ListNode dummy = new ListNode(-1);
    dummy.next = head;
    p = dummy;
    int mod = n % k;
    for (int i = 0; i < mod; i++) p = p.next;
    while (p != null) {
        ListNode node = p;
        for (int i = 0; i < k && node != null; i++) {
            node = node.next;
        }
        if (node == null) break;
        ListNode start = p.next, end = node.next;
        // reverse() 同上
        ListNode t = reverse(start, end);
        ListNode tail = p.next;
        p.next = t;
        p = tail;
    }
    return dummy.next;
}
```