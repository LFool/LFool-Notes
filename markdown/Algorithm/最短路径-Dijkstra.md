# 最短路径-Dijkstra

[743. 网络延迟时间](https://leetcode-cn.com/problems/network-delay-time)

[1514. 概率最大的路径](https://leetcode-cn.com/problems/path-with-maximum-probability)

[1631. 最小体力消耗路径](https://leetcode-cn.com/problems/path-with-minimum-effort)

## Dijkstra

### 适用条件

**<font color='red'>加权有向图，没有负权重边</font>**

### 模版

```java
public class State {
    // 节点 id
    public int id;
    // 当前节点距离起点的距离
    public int distFromStart;
    public State(int id, int distFromStart) {
        this.id = id;
        this.distFromStart = distFromStart;
    }
}

public class Dijkstra {
    public int[] dijkstra(List<Integer>[] graph, int start) {
        // 节点数量
        int n = graph.length;
        // 记录每个点到 start 的最短距离
        int[] distTo = new int[n];
        // 初始化为最大值
        Arrays.fill(distTo, Integer.MAX_VALUE);
        // start to start = 0
        distTo[start] = 0;

        // 优先队列，按照 distFromStart 排序（贪心思想）
        Queue<State> pq = new PriorityQueue<>(Comparator.comparingInt(o -> o.distFromStart));
        // 添加起点
        pq.offer(new State(start, 0));

        while (!pq.isEmpty()) {
            State curState = pq.poll();
            int curNodeId = curState.id;
            int curDistFromStart = curState.distFromStart;
            // 非最优，直接跳过
            if (curDistFromStart > distTo[curNodeId]) {
                continue;
            }
            for (int nextNodeId : adj(curNodeId)) {
                int distToNextNode = distTo[curNodeId] + weight(curNodeId, nextNodeId);
                // 更新到下一个节点的距离
                if (distToNextNode < distTo[nextNodeId]) {
                    distTo[nextNodeId] = distToNextNode;
                    pq.offer(new State(nextNodeId, distToNextNode));
                }
            }
        }
        return distTo;
    }

    /**
     * 权重函数
     * @param from from 顶点
     * @param to to 顶点
     * @return 返回 from -> to 权重
     */
    private int weight(int from, int to) {
        return 0;
    }

    /**
     * 邻接节点
     * @param id 当且节点 Id
     * @return 返回当前节点的邻接节点
     */
    private List<Integer> adj(int id) {
        return new ArrayList<>();
    }
}
```

### 扩展

**<font color='red'>如果我只想计算起点 `start` 到某一个终点 `end` 的最短路径，是否可以修改算法，提升一些效率？</font>**

```java
// 输入起点 start 和终点 end，计算起点到终点的最短距离
int dijkstra(List<Integer>[] graph, int start, int end) {

    // ...

    while (!pq.isEmpty()) {
        State curState = pq.poll();
        int curNodeId = curState.id;
        int curDistFromStart = curState.distFromStart;

        // 在这里加一个判断就行了，其他代码不用改
        if (curNodeID == end) {
            return curDistFromStart;
        }

        if (curDistFromStart > distTo[curNodeID]) {
            continue;
        }

        // ...
    }

    // 如果运行到这里，说明从 start 无法走到 end
    return Integer.MAX_VALUE;
}
```

因为优先级队列自动排序的性质，**每次**从队列里面拿出来的都是 `distFromStart` 值最小的，所以当你**第一次**从队列中拿出终点 `end` 时，此时的 `distFromStart` 对应的值就是从 `start` 到 `end` 的最短距离

这个算法较之前的实现提前 return 了，所以效率有一定的提高
