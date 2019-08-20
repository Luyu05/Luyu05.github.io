---
title: 01-Bag
date: 2019-01-15 11:58:54
tags: 01背包
categories: Algorithm
---
动态规划小试牛刀：01背包问题汇总
<!-- more -->

# 物品数目为1，一定承载能力的背包最多装多少kg？
```java
static int max;

static void maxWeight(int[] arr, int target, int i, int cur) {
    //end condition
    if (cur > target) {
        return;
    }
    max = Math.max(cur, max);
    if (i >= arr.length || max == target) {
        return;
    }
    //recur
    maxWeight(arr, target, i + 1, cur);
    maxWeight(arr, target, i + 1, cur + arr[i]);
}

static int dp(int[] arr, int target) {
    boolean[][] dp = new boolean[arr.length][target + 1];
    for (int i = 0; i < arr.length; i++) {
        for (int j = 1; j < target + 1; j++) {
            if (i > 0 && dp[i - 1][j]) {
                dp[i][j] = true;
                if (j + arr[i] <= target) {
                    dp[i][j + arr[i]] = true;
                }
            }
            //由于物品数量为1 所以下面的逻辑要放到递推之后
            if (j == arr[i]) {
                dp[i][j] = true;
            }
        }
    }
    for (int k = target; k >= 0; k--) {
        if (dp[arr.length - 1][k]) return k;
    }
    return -1;
}
```

# 物品数目为1，装满背包最少需要几件物品？
```java
static int min = Integer.MAX_VALUE;

static void recur(int[] arr, int target, int i, int cur, int count) {
    //end
    if (cur == target) {
        min = Math.min(min, count);
        return;
    }
    if (i == arr.length || cur > target) {
        return;
    }
    //recur
    recur(arr, target, i + 1, cur, count);
    recur(arr, target, i + 1, cur + arr[i], count + 1);
}

static int dp(int[] arr, int target) {
    int n = arr.length;
    int[][] dp = new int[n][target + 1];
    for (int i = 0; i < n; i++) {
        for (int j = 1; j <= target; j++) {
            if (i > 0 && dp[i - 1][j] > 0) {
                dp[i][j] = Math.min(dp[i - 1][j], dp[i][j] > 0 ? dp[i][j] : Integer.MAX_VALUE);
                if (j + arr[i] <= target) {
                    dp[i][j + arr[i]] = dp[i][j] + 1;
                }
            }
            //由于物品数量为1 所以下面的逻辑要放到递推之后
            if (j == arr[i]) {
                dp[i][j] = 1;
            }
        }
    }
    int min = Integer.MAX_VALUE;
    for (int k = arr.length - 1; k >= 0; k--) {
        if (dp[k][target] > 0)
            min = Math.min(dp[k][target], min);
    }
    return min;
}
```

# 物品数目不限，一定承载能力的背包最多装多少kg？
```java
static int max = Integer.MIN_VALUE;

static void recur(int[] arr, int target, int i, int cur) {
    //end
    if(cur > target) {
        return;
    }
    max = Math.max(max, cur);
    if(i == arr.length) return;
    //recur
    recur(arr, target, i+1, cur);
    recur(arr, target, i, cur+arr[i]);
}

static int dp(int[] arr, int target) {
    boolean[][] dp = new boolean[arr.length][target+1];
    for(int i = 0; i < arr.length; i++) {
        dp[i][arr[i]] = true;
        for(int j = 0; j <= target; j++) {
            if(i > 0 && dp[i-1][j]) {
                dp[i][j] = true;
            }
            if(dp[i][j] && j+arr[i] <= target) {
                dp[i][j+arr[i]] = true;
            }
        }
    }
    for(int k = target; k >= 0; k--) {
        if(dp[arr.length-1][k]) return k;
    }
    return -1;
}
```

# 物品数目不限，一定承载能力的背包装满需要最少物品的数量？
```java
static int min = Integer.MAX_VALUE;

static void recur(int[] arr, int target, int i, int cur, int count) {
    //end
    if(cur == target) {
        min = Math.min(min, count);
        return;
    }
    if(cur > target || i==arr.length) {
        return;
    }

    //recur
    recur(arr, target, i+1, cur, count);
    recur(arr, target, i, cur+arr[i], count+1);
}

static int dp(int[] arr, int target) {
    int[][] dp = new int[arr.length][target+1];
    for (int i = 0; i < arr.length; i++) {
        dp[i][arr[i]] = 1;
        for (int j = 1; j <= target; j++) {
            //from top
            if(i > 0 && dp[i-1][j] > 0) {
                dp[i][j] = Math.min(dp[i-1][j], dp[i][j] != 0 ? dp[i][j] : Integer.MAX_VALUE);
            }
            //from left
            if(dp[i][j] > 0 && j+arr[i] <= target) {
                dp[i][j+arr[i]] = Math.min(dp[i][j] + 1, dp[i][j+arr[i]] > 0 ? dp[i][j+arr[i]] : Integer.MAX_VALUE);
            }
        }
    }
    int min = Integer.MAX_VALUE;
    for(int k = arr.length-1; k >= 0; k--) {
        min = Math.min(min, dp[k][target]);
    }
    return min;
}
```

# 物品数目不限，装满背包有多少方案？
```java
static int sum;

static void recur(int[]arr, int target, int i, int cur) {
    //end
    if(target == cur) {
        sum ++;
        return;
    }
    if(cur > target || i==arr.length) {
        return;
    }
    //recur
    recur(arr, target, i+1, cur);
    recur(arr, target, i, cur+arr[i]);
}

static int dp(int[]arr, int target) {
    int[][] dp = new int[arr.length][target+1];
    for (int i = 0; i < arr.length; i++) {
        dp[i][arr[i]] = 1;
        for (int j = 1; j <= target; j++) {
            //from top
            if(i > 0 && dp[i-1][j] > 0) {
                dp[i][j] +=  dp[i-1][j];
            }
            //from left
            if(dp[i][j] > 0 && j+arr[i] <= target) {
                dp[i][j+arr[i]] += dp[i][j];
            }
        }
    }
    return dp[arr.length-1][target];
}
```

Tips:
* 01背包，回溯方法都相对简单，对于是否可以重复拿区别只在回溯时候i是否加1
* 01背包 优先采用dp，一般地将物品的重量作为行，0~背包重量作为列
* dp解法中，如果求最小值一般用int[][]，求最大值一般用boolean[][]
* 想不清楚的时候，先画图去递推
* 优雅简洁的代码，写起来也很顺手，思路也比较清楚