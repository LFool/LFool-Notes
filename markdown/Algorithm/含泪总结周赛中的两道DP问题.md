# ð¥¹å«æ³ªæ»ç»å¨èµä¸­çä¸¤éãDPãé®é¢

[6109. ç¥éç§å¯çäººæ°](https://leetcode.cn/problems/number-of-people-aware-of-a-secret/)

[6110. ç½æ ¼å¾ä¸­éå¢è·¯å¾çæ°ç®](https://leetcode.cn/problems/number-of-increasing-paths-in-a-grid/)

### <font color=#1FA774>ç¥éç§å¯çäººæ°</font>

**é¢ç®è¯¦æå¯è§ [ç¥éç§å¯çäººæ°](https://leetcode.cn/problems/number-of-people-aware-of-a-secret/)**

æ¯èµçæ¶åä¸ç´å¨æèç¶æè½¬ç§»ï¼èªå·±å¿ä¸­æ³ç`dp[i]`å«ä¹æ¯ãç¬¬`i`å¤©ç¥éç§å¯çæ»äººæ°ãï¼ä¸ç´ä¸ç¥éæä¹æ¨åºç¶æè½¬ç§»

ç´å°çå°å«äººçå®ä¹ï¼`dp[i]`è¡¨ç¤ºç¬¬`i`å¤©**<font color='red'>æ°</font>**ç¥éç§å¯çäººæ°ï¼ç¬é´å°±æäºï¼ï¼ï¼

æä»¥ç¬¬`i`å¤©æ°å¢çäººæ°é½æ¥èª`(i - forget, i - delay]`åºé´åçäººçåäº«ï¼ææï¼$dp[i] = \sum\limits^{i - delay}_{j = i - forget + 1}dp[j]$

å¯¹äºæ¯æ¬¡æ±`dp[i]`é½æä¸ä¸ªåºé´åçç´¯å è¿ç¨ï¼æä»¥å¯ä»¥å©ç¨ãåç¼åãä¼åä¸æ³¢ï¼ï¼

```java
public int peopleAwareOfSecret(int n, int delay, int forget) {
    int mod = (int) 1e9 + 7;
    long[] f = new long[n + 1];
    long[] preSum = new long[n + 1];
    f[1] = 1;
    preSum[1] = 1;
    for (int i = 2; i <= n; i++) {
        int l = Math.max(0, i - forget);
        int r = Math.max(0, i - delay);
        // æ³¨æï¼å¯¹äºç¸åçæ±ä½éè¦å ä¸ä¸ä¸ª modï¼è½¬åææ­£æ°èå´åï¼ï¼
        f[i] = (preSum[r] - preSum[l] + mod) % mod;
        preSum[i] = (preSum[i - 1] + f[i]) % mod;
    }
    return (int) (preSum[n] - preSum[Math.max(0, n - forget)] + mod) % mod;
}
```

### <font color=#1FA774>ç½æ ¼å¾ä¸­éå¢è·¯å¾çæ°ç®</font>

**é¢ç®è¯¦æå¯è§ [ç½æ ¼å¾ä¸­éå¢è·¯å¾çæ°ç®](https://leetcode.cn/problems/number-of-increasing-paths-in-a-grid/)**

è¿ä¸ªé¢ç®å°±æ¯ä¸ä¸ªãè®°å¿åæç´¢ãï¼ä¸ºäºé¿åè·¯å¾éå¤ï¼å¯ä»¥æ±åºä»¥æä¸ä¸ªç¹ä¸ºç»å°¾çææè·¯å¾ï¼æä»¥æææç¹é½éåä¸éå³å¯ï¼

```java
private int m, n;
private int[][] grid;
private int[][] emeo;
private int mod = (int) 1e9 + 7;
private int[][] dirs = new int[][] { {-1, 0}, {1, 0}, {0, -1}, {0, 1} };
public int countPaths(int[][] grid) {
    int ans = 0;
    this.grid = grid;
    this.m = grid.length;
    this.n = grid[0].length;
    emeo = new int[m][n];
    // éåææç¹
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            ans = (ans + dp(i, j)) % mod;
        }
    }
    return ans;
}
// ä»¥ç¹ (i, j) ç»å°¾çææè·¯å¾
private int dp(int i, int j) {
    // å·²ç»æ±è¿äºï¼ç´æ¥è¿å
    if (emeo[i][j] != 0) return emeo[i][j];
    long ans = 1L;
    // å¾åä¸ªæ¹åéå½éå
    for (int[] dir : dirs) {
        int ii = i + dir[0], jj = j + dir[1];
        if (ii >= 0 && ii < m && jj >= 0 && jj < n && grid[ii][jj] < grid[i][j]) {
            ans = (ans + dp(ii, jj)) % mod;
        }
    }
    // ä¿å­ç»æ
    emeo[i][j] = (int) ans;
    return emeo[i][j];
}
```