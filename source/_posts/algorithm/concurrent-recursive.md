---
title: 利用递归算法并行化解决谜题框架
date: 2015-04-14 15:41:16 +0800
toc: true
categories: [ 算法之美 ]
tags: [ 算法 ]
---

我们将谜题定义为：包含一个初始位置，一个目标位置，以及用于判断是否是有效移动的规则集。

规则集包含两部分：计算从指定位置开始的所有合法移动，以及每次移动的结果位置。

下面先给出表示谜题的抽象类，其中的类型参数P和M表示位置类和移动类。根据这个接口，
我们可以写一个简单的串行求解程序，该程序将在谜题空间Puzzle Space中查找，
直到找到一个解答或者找遍了整个空间都没有发现答案。注：一个移动M代表一步。

```java
/** 表示 搬箱子 之类谜题的抽象类*/
public interface Puzzle<P, M> {
    P initialPosition();
 
    boolean isGoal(P position);
 
    Set<M> legalMoves(P position);
 
    P move(P position, M move);
}
```
<!-- more -->

下面的PuzzleNode代表通过一系列的移动到达的一个位置，其中保存了到达该位置的移动以及前一个Node。
只要沿着PuzzleNode链接逐步回溯，就可以重新构建出达到当前位置的移动序列。

```java
/** 用于谜题解决框架的链接节点 */
@Immutable
public class PuzzleNode<P, M> {
    final P pos;
    final M move;
    final PuzzleNode<P, M> prev;
 
    public PuzzleNode(P pos, M move, PuzzleNode<P, M> prev) {
        this.pos = pos;
        this.move = move;
        this.prev = prev;
    }
 
    List<M> asMoveList() {
        List<M> solution = new LinkedList<M>();
        for (PuzzleNode<P, M> n = this; n.move != null; n = n.prev)
            solution.add(0, n.move);
        return solution;
    }
}
```
下面的SequentialPuzzleSolver给出了谜题框架的串行解决方案，它在谜题空间中执行深度优先搜索，
当找到解答方案，不一定是最短的解决方案，结束搜索。
```java
/** 串行的谜题解答器*/
public class SequentialPuzzleSolver<P, M> {
    private final Puzzle<P, M> puzzle;
    private final Set<P> seen = new HashSet<P>();
 
    public SequentialPuzzleSolver(Puzzle<P, M> puzzle) {
        this.puzzle = puzzle;
    }
 
    public List<M> solve() {
        P pos = puzzle.initialPosition();
        return search(new PuzzleNode<P, M>(pos, null, null));
    }
 
    private List<M> search(PuzzleNode<P, M> node) {
        if (!seen.contains(node.pos)) {
            seen.add(node.pos);
            if (puzzle.isGoal(node.pos))
                return node.asMoveList();
            for (M move : puzzle.legalMoves(node.pos)) {
                P pos = puzzle.move(node.pos, move);
                PuzzleNode<P, M> child = new PuzzleNode<P, M>(pos, move, node);
                List<M> result = search(child);
                if (result != null)
                    return result;
            }
        }
        return null;
    }
}
```

## 并行方案

接下来我们给出并行解决方案，ConcurrentPuzzleSolver中使用了一个内部类SolverTask，
这个类扩展了PuzzleNode并实现了Runnable。大多数工作都是在run中完成的：首先计算下一步肯能到达的所有位置，
并去掉已经到达的位置，然后判断（这个任务或者其他某个任务）是否已经成功完成，最后将尚未搜索过的位置提交给Executor。
```java
public class ConcurrentPuzzleSolver<P, M> {
    private final Puzzle<P, M> puzzle;
    private final ExecutorService exec;
    private final ConcurrentMap<P, Boolean> seen;
    protected final ValueLatch<PuzzleNode<P, M>> solution = new ValueLatch<PuzzleNode<P, M>>();
 
    public ConcurrentPuzzleSolver(Puzzle<P, M> puzzle) {
        this.puzzle = puzzle;
        this.exec = initThreadPool();
        this.seen = new ConcurrentHashMap<P, Boolean>();
        if (exec instanceof ThreadPoolExecutor) {
            ThreadPoolExecutor tpe = (ThreadPoolExecutor) exec;
            tpe.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        }
    }
 
    private ExecutorService initThreadPool() {
        return Executors.newCachedThreadPool();
    }
 
    public List<M> solve() throws InterruptedException {
        try {
            P p = puzzle.initialPosition();
            exec.execute(newTask(p, null, null));
            // block until solution found
            PuzzleNode<P, M> solnPuzzleNode = solution.getValue();
            return (solnPuzzleNode == null) ? null : solnPuzzleNode.asMoveList();
        } finally {
            exec.shutdown();
        }
    }
 
    protected Runnable newTask(P p, M m, PuzzleNode<P, M> n) {
        return new SolverTask(p, m, n);
    }
 
    protected class SolverTask extends PuzzleNode<P, M> implements Runnable {
        SolverTask(P pos, M move, PuzzleNode<P, M> prev) {
            super(pos, move, prev);
        }
 
        public void run() {
            if (solution.isSet() || seen.putIfAbsent(pos, true) != null)
                return; // already solved or seen this position
            if (puzzle.isGoal(pos))
                solution.setValue(this);
            else
                for (M m : puzzle.legalMoves(pos))
                    exec.execute(newTask(puzzle.move(pos, m), m, this));
        }
    }
}

@ThreadSafe
public class ValueLatch<T> {
    @GuardedBy("this")
    private T value = null;
    private final CountDownLatch done = new CountDownLatch(1);
 
    public boolean isSet() {
        return (done.getCount() == 0);
    }
 
    public synchronized void setValue(T newValue) {
        if (!isSet()) {
            value = newValue;
            done.countDown();
        }
    }
 
    public T getValue() throws InterruptedException {
        done.await();
        synchronized (this) {
            return value;
        }
    }
}
```

## 算法原理

比较串行和并行算法可知：并发方法引入了一种新形式的限制并去掉了一种原有的限制，新的限制在这个问题域中更合适。
串行版本的程序执行深度优先搜索，因此搜索过程将受限于栈的大小。并发版本程序执行广度优先搜索，因此不会受到栈大小的限制。

第一个找到解答的线程还会关闭Executor，从而阻止接受显得任务。
要避免处理RejectedExecutionException（等待队列满员或者是Executor关闭后提交的任务），
需要将拒绝执行处理器设置为DiscardPolicy 。

如果不存在解答，那么ConcurrentPuzzleSolver就会永远的等待下去，getSolution一直阻塞下去。
通过记录活动任务数量，当该值为零时将解答设置为null，如下：

```java
public class PuzzleSolver<P, M> extends ConcurrentPuzzleSolver<P, M> {
    PuzzleSolver(Puzzle<P, M> puzzle) {
        super(puzzle);
    }
 
    private final AtomicInteger taskCount = new AtomicInteger(0);
 
    protected Runnable newTask(P p, M m, PuzzleNode<P, M> n) {
        return new CountingSolverTask(p, m, n);
    }
 
    class CountingSolverTask extends SolverTask {
        CountingSolverTask(P pos, M move, PuzzleNode<P, M> prev) {
            super(pos, move, prev);
            taskCount.incrementAndGet();
        }
 
        public void run() {
            try {
                super.run();
            } finally {
                if (taskCount.decrementAndGet() == 0)
                    solution.setValue(null);
            }
        }
    }
}
```

另外，还可以将ValueLatch设置为限时的，将getValue使用await的限时版实现，
那么就可以指定多少时间内搜索结果，搜不到就超时中断。

