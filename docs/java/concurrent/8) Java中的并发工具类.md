##  Java中的并发工具类

在JDK的并发包里提供了几个非常有用的并发工具类。CountDownLatch、CyclicBarrier和 Semaphore工具类提供了一种并发流程控制的手段，Exchanger工具类则提供了在线程间交换数 据的一种手段。

### 等待多线程完成的CountDownLatch

CountDownLatch允许一个或多个线程等待其他线程完成操作。

* 假如有这样一个需求：我们需要解析一个Excel里多个sheet的数据，此时可以考虑使用多线程，每个线程解析一个sheet里的数据，等到所有的sheet都解析完之后，程序需要提示解析完成。在这个需求中，要实现主线程等待所有线程完成sheet的解析操作，最简单的做法是使用join()方法。

* CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完 成，这里就传入N。

* 当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法 会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个 CountDownLatch的引用传递到线程里即可。

* `注意` 计数器必须大于等于0，只是等于0时候，计数器就是零，调用await方法时不会阻塞当前线程。CountDownLatch不可能重新初始化或者修改CountDownLatch对象的内部计数器的值。一个线程调用countDown方法happen-before，另外一个线程调用await方法。

#### 同步屏障CyclicBarrier

* CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会 开门，所有被屏障拦截的线程才会继续运行。

#### CyclicBarrier简介

* CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞。

* CyclicBarrier还提供一个更高级的构造函数CyclicBarrier（int parties，Runnable barrier-Action），用于在线程到达屏障时，优先执行barrierAction，方便处理更复杂的业务场景

#### CyclicBarrier的应用场景

* CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景。例如，用一个Excel保存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户 的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流水

#### CyclicBarrier和CountDownLatch的区别

* CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数 器，并让线程重新执行一次。

* CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得Cyclic-Barrier 阻塞的线程数量。isBroken()方法用来了解阻塞的线程是否被中断。

#### 控制并发线程数的Semaphore

* Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假 如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程 并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这 时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连 接。这个时候，就可以使用Semaphore来做流量控制

#### 线程间交换数据的Exchanger

* Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交 换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过 exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也 执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产 出来的数据传递给对方。

* `Exchanger可以用于遗传算法`，遗传算法里需要选出两个人作为交配对象，这时候会交换两人的数据，并使用交叉规则得出2个交配结果。
* `Exchanger也可以用于校对工作`，比如我们需要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否录入一致