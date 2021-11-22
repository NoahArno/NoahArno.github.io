# 蓄水池算法

## ReservoirSampling

蓄水池算法

[蓄水池抽样算法 - 链表随机节点 - 力扣（LeetCode） (leetcode-cn.com)](https://leetcode-cn.com/problems/linked-list-random-node/solution/xu-shui-chi-chou-yang-suan-fa-by-jackwener/)

[蓄水池抽样及实现 - handspeaker - 博客园 (cnblogs.com)](https://www.cnblogs.com/hrlnw/archive/2012/11/27/2777337.html#:~:text=蓄水池抽样（Reservoir Sampling,）是一个很有趣的问题，它能够在o（n）时间内对n个数据进行等概率随机抽取，例如：从1000个数据中等概率随机抽取出100个。 另外，如果数据集合的量特别大或者还在增长（相当于未知数据集合总量），该算法依然可以等概率抽样。)

```java
public static class RandomBox {
    private int[] bag;
    private int N;
    private int count;

    public RandomBox(int capacity) {
        bag = new int[capacity];
        N = capacity;
        count = 0;
    }

    private int rand(int max) {
        return (int) (Math.random() * max) + 1;
    }

    public void add(int num) {
        count++;
        if (count <= N) {
            bag[count - 1] = num;
        } else {
            if (rand(count) <= N) {
                bag[rand(N) - 1] = num;
            }
        }
    }

    public int[] choices() {
        int[] ans = new int[N];
        for (int i = 0; i < N; i++) {
            ans[i] = bag[i];
        }
        return ans;
    }

}
```



# bfprt算法

## FindMinKth

> 无序arr中如何找到第k小的数，O(N)，k以1开头

解题思路，我们可以改写快排，每次将arr给分为三个区域，然后判断k是在哪个区域

```java
public static int minKth(int[] array, int k) {
    int[] arr = copyArray(array);
    return process(arr, 0, arr.length - 1, k - 1);
}

public static int[] copyArray(int[] arr) {
    int[] ans = new int[arr.length];
    for (int i = 0; i != ans.length; i++) {
        ans[i] = arr[i];
        return ans;
    }
}

public static int process(int[] arr, int L, int R, int index) {
    if (L == R) {
        return arr[L];
    }
    int pivot = arr[L + (int) (Math.random() * (R - L + 1))];
    
    int[] range = partition(arr, L, R, pivot);
    if (index >= range[0] && index <= range[1]) {
        return arr[index];
    } else if (index < range[0]) {
        return process(arr, L, range[0] - 1, index);
    } else {
        return process(arr, range[1] + 1, R, index);
    }
}

public static int[] partition(int[] arr, int L, int R, int pivot) {
    int less = L - 1;
    int more = R + 1;
    int cur = L;
    while (cur < more) {
        if (arr[cur] < pivot) {
            swap(arr, ++less, cur++);
        } else if (arr[cur] > pivot) {
            swap(arr, cur, --more);
        } else {
            cur++;
        }
    }
    return new int[] { less + 1, more - 1 };
}
```

但是上面那个方法只是在概率上收敛于O(N)，而bfprt在上述方法对于随机选择分界数上进行了优化。其余的部分和上述方法几乎一样。

在优化方面，将整个arr分为5个一组，然后分别对这些小组进行内部排序，找出他们各个的中位数然后放到一个数组中，然后返回该数组中的中位数

```java
public static int minKth3(int[] array, int k) {
    int[] arr = copyArray(array);
    return bfprt(arr, 0, arr.length - 1, k - 1);
}

public static int bfprt(int[] arr, int L, int R, int index) {
    if (L == R) {
        return arr[L];
    }
    int pivot = medianOfMedians(arr, L, R);
    int[] range = partition(arr, L, R, pivot);
    if (index >= range[0] && index <= range[1]) {
        return arr[index];
    } else if (index < range[0]) {
        return bfprt(arr, L, range[0] - 1, index);
    } else {
        return bfprt(arr, range[1] + 1, R, index);
    }
}
```

```java
public static int medianOfMedians(int[] arr, int L, int R) {
    int size = R - L + 1;
    int offset = size % 5 == 0 ? 0 : 1;
    int[] mArr = new int[size / 5 + offset];
    for (int team = 0; team < mArr.length; team++) {
        int teamFirst = L + team * 5;
        // L ... L + 4
        // L + 5 ... L + 9
        mArr[team] = getMedian(arr, teamFirst, Math.min(R, teamFirst + 4));
        // getMedian就是将数组中进行排序，然后找出中位数，如果是1234就是2
    }
    // marr中找到中位数
    // marr 在0到marr中找到mArr.length / 2大的数所在的位置，也就是m
    return bfprt(mArr, 0, mArr.length - 1, mArr.length / 2);
}
```

