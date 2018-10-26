---
layout: post
title: ForkJoinPool
date: 2018-10-25 09:42:31
categories: Java
share: y
excerpt_separator: <!--more-->
---

- ForkJoinPool 不是为了替代 ExecutorService，而是它的补充，在某些应用场景下性能比 ExecutorService 更好。
- ForkJoinPool 主要用于实现“分而治之”的算法，特别是分治之后递归调用的函数，例如 quick sort 等。
- ForkJoinPool 最适合的是计算密集型的任务，如果存在 I/O，线程间同步，sleep() 等会造成线程长时间阻塞的情况时，最好配合使用 ManagedBlocker。

<!--more-->

## demo

```
public class SumTask extends RecursiveTask {

    private static final int THRESHOLD = 100;
    private long[] array;
    private int start;
    private int end;

    public SumTask(long[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ignored) {
            }
            System.out.println(String.format("compute %d~%d = %d", start, end, sum));
            return sum;
        }

        int middle = (end + start) / 2;
        System.out.println(String.format("split %d~%d ==> %d~%d, %d~%d", start, end, start, middle, middle, end));
        SumTask subtask1 = new SumTask(this.array, start, middle);
        SumTask subtask2 = new SumTask(this.array, middle, end);
        invokeAll(subtask1, subtask2);
        Long subresult1 = (Long) subtask1.join();
        Long subresult2 = (Long) subtask2.join();
        Long result = subresult1 + subresult2;
        System.out.println("result = " + subresult1 + " + " + subresult2 + " ==> " + result);
        return result;
    }

    public static void main(String[] args) {
        long[] array = new long[400];
        fillRandom(array);

        ForkJoinPool fjp = new ForkJoinPool(4); // 最大并发数4
        ForkJoinTask<Long> task = new SumTask(array, 0, array.length);
        long startTime = System.currentTimeMillis();
        Long result = fjp.invoke(task);
        long endTime = System.currentTimeMillis();
        System.out.println("Fork/join sum: " + result + " in " + (endTime - startTime) + " ms.");
    }

    private static void fillRandom(long[] array) {
        for (int i = 0; i < array.length; i++) {
            array[i] = RandomUtils.nextInt(1, 100);
        }
    }
}
```

运行结果：

```
split 0~400 ==> 0~200, 200~400
split 0~200 ==> 0~100, 100~200
split 200~400 ==> 200~300, 300~400
compute 0~100 = 5380
compute 300~400 = 5024
compute 200~300 = 4797
compute 100~200 = 4939
result = 4797 + 5024 ==> 9821
result = 5380 + 4939 ==> 10319
result = 10319 + 9821 ==> 20140
Fork/join sum: 20140 in 1022 ms.
```

## 源码解析

ForkJoinPool在jdk1.7引入，在jdk1.8进行了优化，本文的源码基于jdk1.8最新版本。

![work-stealing](../images/work-stealing.png)

ForkJoinPool核心是work-stealing算法.

ForkJoinPool里有三个重要的角色：

- ForkJoinWorkerThread（下文简称worker）：包装Thread；
- WorkQueue：任务队列，双向；
- ForkJoinTask：worker执行的对象，实现了Future。两种类型，一种叫submission，另一种就叫task。

ForkJoinPool 中的 WorkQueue[] 偶数下标，存储的是外部提交的submission，没有worker；奇数下标，存储的是task,有自己的worker,所以在push，pop的时候不需要加锁。

worker 操作自己的WorkQueue是LIFO；steal其他WorkQueue里的任务是FIFO。

## JAVA Stream 中的parall

- 如果是CPU计算密集型的任务则推荐采用并行
- 如果是IO密集型或者是长时间执行会阻塞的任务时，则不推荐parallel。因为这样的阻塞会导致整个服务的block。

