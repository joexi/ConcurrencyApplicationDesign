## Concurrency and Application Design
( 2011 Apple Inc. All Rights Reserved. Last updated: 2011-01-19)

## 并发程序设计
( 引自 2011 Apple Inc. All Rights Reserved. Last updated: 2011-01-19)

In the early days of computing, the maximum amount of work per unit of time that a computer could perform was determined by the clock speed of the CPU. But as technology advanced and processor designs became more compact, heat and other physical constraints started to limit the maximum clock speeds of processors. And so, chip manufacturers looked for other ways to increase the total performance of their chips. The solution they settled on was increasing the number of processor cores on each chip. By increasing the number of cores, a single chip could execute more instructions per second without increasing the CPU speed or changing the chip size or thermal characteristics. The only problem was how to take advantage of the extra cores.

在计算机时代的早期，CPU的频率决定了一台计算机单位时间内可以执行的最大工作量。但是随着技术的发展，处理器被设计的更加紧凑，发热问题成为了约束CPU的最大因素，所以芯片制造商们开始寻找其他方式来提升芯片的整体性能。他们的解决方案是在每个芯片上增加处理器核心的数量。随着核心数量的提升，单个芯片可以在不改变CPU的性能以及散热情况下在单位时间内执行更多的指令，唯一的问题是如何来合理的使用额外的核心带来的优势。因为多核的缘故，电脑需要软件能够同时处理多个事情，才能发挥多核的优势.



In order to take advantage of multiple cores, a computer needs software that can do multiple things simultaneously. For a modern, multitasking operating system like Mac OS X or iOS, there can be a hundred or more programs running at any given time, so scheduling each program on a different core should be possible. However, most of these programs are either system daemons or background applications that consume very little real processing time. Instead, what is really needed is a way for individual applications to make use of the extra cores more effectively.

对于一个像Mac OS 或者iOS那样现代化的，多进程的操作系统，可以在任何时间运行100个或者更多的程序，所以把程序放在不同的处理器核心上是可行的。然而，这些程序多数是系统进程或者后台应用程序，他们的消耗都非常少。而我们需要的却恰恰相反，我们需要一个单独的应用程序能尽可能充分的利用额外的处理器核心。



The traditional way for an application to use multiple cores is to create multiple threads. However, as the number of cores increases, there are problems with threaded solutions. The biggest problem is that threaded code does not scale very well to arbitrary numbers of cores. You cannot create as many threads as there are cores and expect a program to run well. What you would need to know is the number of cores that can be used efficiently, which is a challenging thing for an application to compute on its own. Even if you manage to get the numbers correct, there is still the challenge of programming for so many threads, of making them run efficiently, and of keeping them from interfering with one another.

应用程序使用多核内核的传统方法就是创建多个线程。然而，随着内核数量的增加，线程解决方案所导致的问题也接踵而至。其中最大的问题就是，线程代码不能很好的扩展到任意数量的内核，他不具备良好的伸缩性。你不可能创建和内核数量一样多的线程并且指望他们运行正常。你需要知道可以被有效使用的核心数量是哪些，这对于一个应用程序对自身的计算来说是一个挑战。即使你设法获取了正确的核心数量，要使得那么多线程能有效的执行并且相互之间不干扰任然是一个复杂的编程挑战。



So, to summarize the problem, there needs to be a way for applications to take advantage of a variable number of computer cores. The amount of work performed by a single application also needs to be able to scale dynamically to accommodate changing system conditions. And the solution has to be simple enough so as to not increase the amount of work needed to take advantage of those cores. The good news is that Apple’s operating systems provide the solution to all of these problems, and this chapter takes a look at the technologies that comprise this solution and the design tweaks you can make to your code to take advantage of them.
因此，在总结了问题之后，我们需要一个方法能够充分的利用可变数量的计算机核心带来的的优势。单个应用程序的工作量也应当能够动态的扩展以适应不变化的系统条件。并且这个方案必须足够简单，以至于不用额外的工作量就可以充分利用这些核心的优势。好消息是苹果的操作系统对这些问题都提供了解决方案，本章需要在技术，设计和解决方案上进行微调以便使得你的代码能更好的利用这些优势。


### The Move Away from Threads 丢弃线程

Although threads have been around for many years and continue to have their uses, they do not solve the general problem of executing multiple tasks in a scalable way. With threads, the burden of creating a scalable solution rests squarely on the shoulders of you, the developer. You have to decide how many threads to create and adjust that number dynamically as system conditions change. Another problem is that your application assumes most of the costs associated with creating and maintaining any threads it uses.

虽然线程技术存在了很多年，并且现在任然在被使用，但是他们无法采用一个可伸缩的方式执行多个进程这一主要问题任然没有被解决。使用线程，那么创一个可扩展的解决方案的重担自然完全落到了开发者的肩膀上，你必须决定创建多少线程来适应动态变化的系统条件。另一个问题是，你的应用程序可能会花费大量的代价在创建并且维护这些线程上。



Instead of relying on threads, Mac OS X and iOS take an asynchronous design approach to solving the concurrency problem. Asynchronous functions have been present in operating systems for many years and are often used to initiate tasks that might take a long time, such as reading data from the disk. When called, an asynchronous function does some work behind the scenes to start a task running but returns before that task might actually be complete. Typically, this work involves acquiring a background thread, starting the desired task on that thread, and then sending a notification to the caller (usually through a callback function) when the task is done. In the past, if an asynchronous function did not exist for what you want to do, you would have to write your own asynchronous function and create your own threads. But now, Mac OS X and iOS provide technologies to allow you to perform any task asynchronously without having to manage the threads yourself.

Mac OS和iOS采取异步的设计方法来解决并发问题，而不是依赖于线程。异步功能在操作系统中已经存在了许多年了，他经常被用来启动一些耗时非常久的任务，例如从磁盘获取数据等。当一个异步函数被调用时，他在后台完成一些操作去执行任务，但是在任务真正完成前就执行了返回。通常情况下，这项工作包括了开启一个后台线程执行所需要的任务，并且在操作完成时同时调用者。在过去，当你需要的异步方法不存在时，你需要写一个异步函数并且创建自己的线程。但是，现在的Mac OS 和iOS提供的技术使你可以异步的执行任何任务而无需自己管理线程。



One of the technologies for starting tasks asynchronously is Grand Central Dispatch (GCD). This technology takes the thread management code you would normally write in your own applications and moves that code down to the system level. All you have to do is define the tasks you want to execute and add them to an appropriate dispatch queue. GCD takes care of creating the needed threads and of scheduling your tasks to run on those threads. Because the thread management is now part of the system, GCD provides a holistic approach to task management and execution, providing better efficiency than traditional threads.
其中一项用于开启异步任务的技术就是Grand Central Dispatch (GCD).这项技术采用线程管理那些你总是会使用的代码，并将他们下移到系统层。你需要做的仅仅是确定你要执行的任务，并且把他们添加到适当的调度队列。GCD帮你处理了创建哪些需要的线程并且调度你的任务在哪一个线程上执行。由于线程管理是系统的一部分，GCG提供了一个全面的方法管理和执行任务，比传统线程更为效率。



Operation queues are Objective-C objects that act very much like dispatch queues. You define the tasks you want to execute and then add them to an operation queue, which handles the scheduling and execution of those tasks. Like GCD, operation queues handle all of the thread management for you, ensuring that tasks are executed as quickly and as efficiently as possible on the system.

操作队列（Operation queues） 是一个和调度队列（Dispatch queue）非常相像的 Objective-C的对象集。你定义好你要执行的任务，将他们添加到操作队列中，交由他来负责这些任务的调度和执行。就像GCD那样操作队列为你处理所有的线程管理，确保任务能在系统中更快更有效率的执行。



The following sections provide more information about dispatch queues, operation queues, and some other related asynchronous technologies you can use in your applications.

以下各节提供了哪些你可以在程序中使用的有关调度队列，操作队列和其他一些相关的异步技术。



#### Dispatch Queues 调度队列

Dispatch queues are a C-based mechanism for executing custom tasks. A dispatch queue executes tasks either serially or concurrently but always in a first-in, first-out order. (In other words, a dispatch queue always dequeues and starts tasks in the same order in which they were added to the queue.) A serial dispatch queue runs only one task at a time, waiting until that task is complete before dequeuing and starting a new one. By contrast, a concurrent dispatch queue starts as many tasks as it can without waiting for already started tasks to finish.

调度队列是一个用来执行自定义任务的基于C语言的结构，无论是串行或者并行处理一个调度队列，他总是FIFO的（先进先出）,换句话说，调度队列总是按照任务被添加进队列的顺序来执行这些任务的。一个串行调度队列在同一时间只执行一个任务等待一个任务完成并且出队才开始下一个任务，相比之下，并发调度队列不等待队列的完成就开始执行下一个任务。

* Dispatch queues have other benefits:
* 调度队列还有其他的优点：

* They provide a straightforward and simple programming interface.
* 他们提供了一个简单明了的编程接口

* They offer automatic and holistic thread pool management.
* 他们提供了全面自动的线程管理池

* They provide the speed of tuned assembly.
* 他们提供了调整集合的速度

* They are much more memory efficient (because thread stacks do not linger in application memory).
* 他们的内存使用更高效（因为线程堆栈不占用用用程序的内存）

* They do not trap to the kernel under load.
* 他们不适用负载下的内核

* The asynchronous dispatching of tasks to a dispatch queue cannot deadlock the queue.
* 任务的异步调度来调度一个队列将不会产生队列死锁


* They scale gracefully under contention.
* 他具有更好的伸缩性

* Serial dispatch queues offer a more efficient alternative to locks and other synchronization primitives.
* 串行调度队列更有效的代替了锁和其他同步源生方法

The tasks you submit to a dispatch queue must be encapsulated inside either a function or a block object. Block objects are a C language feature introduced in Mac OS X v10.6 and iOS 4.0 that are similar to function pointers conceptually, but have some additional benefits. Instead of defining blocks in their own lexical scope, you typically define blocks inside another function or method so that they can access other variables from that function or method. Blocks can also be moved out of their original scope and copied onto the heap, which is what happens when you submit them to a dispatch queue. All of these semantics make it possible to implement very dynamic tasks with relatively little code.

你提交给你一个调度队列的任务，必须是一个封装了的函数或者是一个块对象。Mac OS 10.6和iOS4.0中的块对象的概念和C语言中的函数指针类似，但有一些额外的好处，我们通常在函数或者方法中定义块对象使得其能访问函数的局部变量，而不是将其定义在其他的地方。当你将一个块对象提交给调度队列时，他将从他们原来的范围中搬出并复制到堆上。上述所有的这些都是为了能使用较少的代码来实现动态的任务。

Dispatch queues are part of the Grand Central Dispatch technology and are part of the C runtime. For more information about using dispatch queues in your applications, see **Dispatch Queues.** For more information about blocks and their benefits, see **Blocks Programming Topics.**

调度队列是GCD技术和C标准函数的一部分。想了解更多关于在您的应用程序中使用的调度队列的信息，请参阅**Dispatch Queues.**对于块和他们的好处有关的信息，请参阅块**Blocks Programming Topics.**

#### Dispatch Sources 调度源

Dispatch sources are a C-based mechanism for processing specific types of system events asynchronously. A dispatch source encapsulates information about a particular type of system event and submits a specific block object or function to a dispatch queue whenever that event occurs. You can use dispatch sources to monitor the following types of system events:

调度源是基于C语言的为了异步处理特定类型的系统事件的结构.一个调度源封装了关于特定类型系统事件的信息，并且在事件发生时提交一个特定的块对象到调度队列。你可以使用调度源监视以下类型的系统事件：

* Timers
* 定时器

* Signal handlers
* 信号处理

* Descriptor-related events
* 事件相关描述

* Process-related events
* 事件相关过程

* Mach port events
* Mach 端口事件

* Custom events that you trigger
* 自定义事件触发

Dispatch sources are part of the Grand Central Dispatch technology. For information about using dispatch sources to receive events in your application, see **Dispatch Sources.**

调度源是GCD技术的一部分。关于接受应用程序系统事件的信息，参见**Dispatch Sources.**

#### Operation Queues 操作队列

An operation queue is the Cocoa equivalent of a concurrent dispatch queue and is implemented by theNSOperationQueue class. Whereas dispatch queues always execute tasks in first-in, first-out order, operation queues take other factors into account when determining the execution order of tasks. Primary among these factors is whether a given task depends on the completion of other tasks. You configure dependencies when defining your tasks and can use them to create complex execution-order graphs for your tasks.

操作队列使用NSOperationQueue实现，相当于并发调度队列。有别于调度队列的先进先出，操作队列在确定任务执行顺序时会考虑到其他因素。其中主要的因素是一个给定的任务是否依赖于其他任务的完成。

The tasks you submit to an operation queue must be instances of the NSOperation class. An operation object is anObjective-C object that encapsulates the work you want to perform and any data needed to perform it. Because theNSOperation class is essentially an abstract base class, you typically define custom subclasses to perform your tasks. However, the Foundation framework does include some concrete subclasses that you can create and use as is to perform tasks.

你提交给操作队列的任务，必须是一个NSOperation类的实例。NSOperation对象是一个包含了你要执行的任务已经执行该任务所需要的数据的Objective-C对象.由于一个NSOperation对象本质上是一个抽象基类，所以你需要自定义一个子类来执行任务，函数类库中已经建立了一些现成的子类可以供你使用。

Operation objects generate key-value observing (KVO) notifications, which can be a useful way of monitoring the progress of your task. Although operation queues always execute operations concurrently, you can use dependencies to ensure they are executed serially when needed.

Operation对象生成一个key-value观察通知来监测任务的完成进度。尽管操作队列总是并发的执行操作，你也可以按你的需要使他们按顺序执行。

For more information about how to use operation queues, and how to define custom operation objects, see **Operation Queues.**

了解更多有关如何使用操作队列，以及如何自定义操作对象的信息，请参阅**Operation Queues.**

### Asynchronous Design Techniques 异步设计技术

Before you even consider redesigning your code to support concurrency, you should ask yourself whether doing so is necessary. Concurrency can improve the responsiveness of your code by ensuring that your main thread is free to respond to user events. It can even improve the efficiency of your code by leveraging more cores to do more work in the same amount of time. However, it also adds overhead and increases the overall complexity of your code, making it harder to write and debug your code.

在你不惜要重构你的代码来使得他们支持并发前，你需要问一问你自己这是否是必须的。并发可以通过确认你的主线程是否可以响应你的事件来提高代码的响应能力。他也可以通过利用更多的内核来使得单位时间处理更多的任务，从而提升你的代码的效率。然而，这也增加了你代码的复杂度，使得你的代码更难于编写，调试和维护。

Because it adds complexity, concurrency is not a feature that you can graft onto an application at the end of your product cycle. Doing it right requires careful consideration of the tasks your application performs and the data structures used to perform those tasks. Done incorrectly, you might find your code runs slower than before and is less responsive to the user. Therefore, it is worthwhile to take some time at the beginning of your design cycle to set some goals and to think about the approach you need to take.

由于他的复杂度，并发并非是你可以在项目后期加入到程序中的。要成功的使用并发必须更小心的考虑你应用程序需要执行的任务和那些任务所需要的数据结构。操作不当时，你会发现你的代码比以前运行的更慢并且对用户的响应也更差。因此，在你的设计周期开始前来设定一些目标，考虑以下你需要采用的方法。

Every application has different requirements and a different set of tasks that it performs. It is impossible for a document to tell you exactly how to design your application and its associated tasks. However, the following sections try to provide some guidance to help you make good choices during the design process.

每一个应用程序在执行任务时都有不用的要求和设置，希望文档告诉你要如何设计你的应用程序以及他的相关任务是不可能的！然而，以下各节提供了一些指导，以便你在设计过程中能更好的选择。

#### Define Your Application’s Expected Behavior 定义程序的预期行为

Before you even think about adding concurrency to your application, you should always start by defining what you deem to be the correct behavior of your application. Understanding your application’s expected behavior gives you a way to validate your design later. It should also give you some idea of the expected performance benefits you might receive by introducing concurrency.

在你考虑将并发添加至你的应用程序前，你总是该先考虑什么才是你的程序应该执行的行为。了解你应用程序的预期行为，为你稍后的设计更合理提供了一种方案。他也可以通过使你了解预期中的性能优势来使得你萌生引入并发的念头。

The first thing you should do is enumerate the tasks your application performs and the objects or data structures associated with each task. Initially, you might want to start with tasks that are performed when the user selects a menu item or clicks a button. These tasks offer discrete behavior and have a well defined start and end point. You should also enumerate other types of tasks your application may perform without user interaction, such as timer-based tasks.

你首先要做的是列出你的应用需要执行的任务，和每个任务所需的数据以及数据结构。最初，你可能希望从用户选中一个菜单项或者点击一个按钮执行任务开始，这些任务没有明确的起点和终点。你要需要列举出那些你应用中不需要要用户交互久会被执行的任务必须基于计时器的任务。

After you have your list of high-level tasks, start breaking each task down further into the set of steps that must be taken to complete the task successfully. At this level, you should be primarily concerned with the modifications you need to make to any data structures and objects and how those modifications affect your application’s overall state. You should also note any dependencies between objects and data structures as well. For example, if a task involves making the same change to an array of objects, it is worth noting whether the changes to one object affect any other objects. If the objects can be modified independently of each other, that might be a place where you could make those modifications concurrently.

在你搞定你的高级任务列表后，开始将每一个任务分离成一个个步骤来使得你的任务成功完成。在这个层面上，你需要主要考虑那些你对数据结构和对象需要做出的修改以及这些修改将如何影响到你的应用程序状态。你也需要估计到对象和数据结构间的依赖，例如，当一个任务涉及到对一个数组中的对象进行相同的修改，那么你需要注意这次修改是否会影响到其他的对象。如果对象可以独立的修改，那么你就可以并发的去执行这些修改。

#### Factor Out Executable Units of Work 分解出可执行的工作单元

From your understanding of your application’s tasks, you should already be able to identify places where your code might benefit from concurrency. If changing the order of one or more steps in a task changes the results, you probably need to continue performing those steps serially. If changing the order has no effect on the output, though, you should consider performing those steps concurrently. In both cases, you define the executable unit of work that represents the step or steps to be performed. This unit of work then becomes what you encapsulate using either a block or an operation object and dispatch to the appropriate queue.

通过你的你的应用程序任务的了解，你应该能够识别你的代码中哪里能够从并发中获益。如果改变任务中一个或者多个步骤将导致结果的改变，你可能还是需要顺序的执行这些步骤。如果改变顺序对结果没有影响，那么你可以考虑并发执行这些步骤。在这两种情况下，你都可以定义可执行工作单元来表示要执行的步骤。这一所谓的工作单元将被你封装成块或者操作对象并调度到合适的队列中去。

For each executable unit of work you identify, do not worry too much about the amount of work being performed, at least initially. Although there is always a cost to spinning up a thread, one of the advantages of dispatch queues and operation queues is that in many cases those costs are much smaller than they are for traditional threads. Thus, it is possible for you to execute smaller units of work more efficiently using queues than you could using threads. Of course, you should always measure your actual performance and adjust the size of your tasks as needed, but initially, no task should be considered too small.

对于你定义的工作单元所正在执行的工作，不用担心的太多，至少在最初的时候。虽然调度一个线程总是会带来消耗，但是调度队列和操作队列带来的优势就是，这些消耗将远小于传统的线程。因此使用队列去操作更小的工作单元将比你直接使用线程更具效率。当然，你总应该更具实际情况来调整任务的大小，但是最初情况下，任务总是不嫌太小的。

#### Identify the Queues You Need 定义你所需要的队列

Now that your tasks are broken up into distinct units of work and encapsulated using block objects or operation objects, you need to define the queues you are going to use to execute that code. For a given task, examine the blocks or operation objects you created and the order in which they must be executed to perform the task correctly.

现在，你的任务被分解成多个工作单元并且封装成块或者操作对象，你需要定义哪些你用来使用和执行代码的队列。对于一个给定的任务，检查你创建的块和操作对象以及哪些待执行任务的顺序是否正确。

If you implemented your tasks using blocks, you can add your blocks to either a serial or concurrent dispatch queue. If a specific order is required, you would always add your blocks to a serial dispatch queue. If a specific order is not required, you can add the blocks to a concurrent dispatch queue or add them to several different dispatch queues, depending on your needs.

如果你使用块实现你的任务，你可以将你的块添加进串行或者并行调度队列中，如果需要特定的执行顺序，那只能选择串行队列，否则则更需你的需要添加到对应的队列中。

If you implemented your tasks using operation objects, the choice of queue is often less interesting than the configuration of your objects. To perform operation objects serially, you must configure dependencies between the related objects. Dependencies prevent one operation from executing until the objects on which it depends have finished their work.

如果你通过操作对象实现你的任务，那么相对于选择队列，不如选择如何配置你的对象。当你需要连续的执行操作对象时候，你必须配置对象间的依赖关系。这确保了一个对象在其依赖对象结束执行前将不会被执行。

#### Tips for Improving Efficiency 提升效率的技巧

In addition to simply factoring your code into smaller tasks and adding them to a queue, there are other ways to improve the overall efficiency of your code using queues:

除了简单的分解成更小的任务，还有使用队列使得你的代码更具效率的方法：

Consider computing values directly within your task if memory usage is a factor. If your application is already memory bound, computing values directly now may be faster than loading cached values from main memory. Computing values directly uses the registers and caches of the given processor core, which are much faster than main memory. Of course, you should only do this if testing indicates this is a performance win.

如果内存消耗是一个需要被考虑的因素，那么考虑直接运算。如果你的应用程序内存被限制了（比如iOS上的应用），那么直接计算结果将要比从缓存中获取结果来的更快,直接算计使用了当前处理器核心的缓存和寄存器，这比使用主存要快的多。当然你要做的仅仅是测试下是否在性能上真的有优势。

Identify serial tasks early and do what you can to make them more concurrent. If a task must be executed serially because it relies on some shared resource, consider changing your architecture to remove that shared resource. You might consider making copies of the resource for each client that needs one or eliminate the resource altogether.

更早的确认你的串行任务们，并且尽可能的使得他们并发执行。如果一个任务必须串行执行，因为他依赖于一些共享资源，考虑改变你的架构来移除这些共享资源。你可以考虑为每一个需要的客户的资源提供备份，或者消除共享的部分。

#### Avoid using locks. 避免使用锁

The support provided by dispatch queues and operation queues makes locks unnecessary in most situations. Instead of using locks to protect some shared resource, designate a serial queue (or use operation object dependencies) to execute tasks in the correct order.

在调度队列和操作队列的支持下，锁在在大部分情况下是不必要的。制定一个串行队列以正确的顺序执行任务来取代锁。

#### Rely on the system frameworks whenever possible. 尽可能的依赖系统框架

The best way to achieve concurrency is to take advantage of the built-in concurrency provided by the system frameworks. Many frameworks use threads and other technologies internally to implement concurrent behaviors. When defining your tasks, look to see if an existing framework defines a function or method that does exactly what you want and does so concurrently. Using that API may save you effort and is more likely to give you the maximum concurrency possible.

最好的实现并发的方式就是利用好系统框架所提供的内置并发的优势，许多框架使用线程和其他技术实现内部并发行为，当你定义你的任务时，看看现有的框架中是否有这样的函数或者方法并发的实现了你想要做的。使用API将节省你的力气，并使你尽可能的实现最大程度的并发。

### Performance Implications 性能的影响

Operation queues, dispatch queues, and dispatch sources are provided to make it easier for you to execute more code concurrently. However, these technologies do not guarantee improvements to the efficiency or responsiveness in your application. It is still your responsibility to use queues in a manner that is both effective for your needs and does not impose an undue burden on your application’s other resources. For example, although you could create 10,000 operation objects and submit them to an operation queue, doing so would cause your application to allocate a potentially nontrivial amount of memory, which could lead to paging and decreased performance.

操作队列，调度队列和调度源将使你更容易的并发执行你的代码，然而这些技术并不保证改善后的效率以及应用程序的响应。在满足你需求的情况且不对你应用程序其他资源增加不必要的负担的情况下如何使用队列还取决于你。比如你大可创建10000个操作对象并将他们提交给队列，这样做将会使你的应用程序分配大量的内存，这将导致分页并且降低你的程序表现。


Before introducing any amount of concurrency to your code—whether using queues or threads—you should always gather a set of baseline metrics that reflect your application’s current performance. After introducing your changes, you should then gather additional metrics and compare them to your baseline to see if your application’s overall efficiency has improved. If the introduction of concurrency makes your application less efficient or responsive, you should use the available performance tools to check for the potential causes.

在将并发引入你的代码前，无论你将要使用队列还是线程，你都需要为你的应用程序现有的表现定义一个性能指标的基准。在你修改你的代码之后，你需要收集更多的指标来和基准进行比较，来看看你的程序性能是否有所改善。如果引入并发降低了你程序的效率或者体验，你应该使用现有的性能工具来检查潜在的原因。

For an introduction to performance and the available performance tools, and for links to more advanced performance-related topics, see **Performance Overview.**

对于性能和可用性能工具的介绍，以及追求更先进的性能相关的主题的链接，请参见**Performance Overview**。

### Concurrency and Other Technologies 并发和其他技术

Factoring your code into modular tasks is the best way to try and improve the amount of concurrency in your application. However, this design approach may not satisfy the needs of every application in every case. Depending on your tasks, there might be other options that can offer additional improvements in your application’s overall concurrency. This section outlines some of the other technologies to consider using as part of your design.

试图提升应用程序并发的数量，将你的代码分解成任务模块是最好的方式，但是这种设计无法满足每一个应用程序在不同环境下的需求。根据你的任务，可能还有其他的选项来提升你应用程序的并发。本节概述了一些其他的技术作为设计中值得考虑的一部分。

#### OpenCL and Concurrency OpenCL 和并发

In Mac OS X, the Open Computing Language (OpenCL) is a standards-based technology for performing general-purpose computations on a computer’s graphics processor. OpenCL is a good technology to use if you have a well-defined set of computations that you want to apply to large data sets. For example, you might use OpenCL to perform filter computations on the pixels of an image or use it to perform complex math calculations on several values at once. In other words, OpenCL is geared more toward problem sets whose data can be operated on in parallel.

在Mac OS X下的开放计算语言（OpenCL）是一个在计算机的图形处理器上进行通用计算的基于标准的技术，如果你有一个希望应用于大规模数据设置的不错的计算设置，使用OpenCL是不错的。比如你可能使用OpenCL的滤波计算图像的像素，或者执行复杂的数学计算。换句话说，OpenCL是面向那些可以被并行操作的数据集的。

Although OpenCL is good for performing massively data-parallel operations, it is not suitable for more general-purpose calculations. There is a nontrivial amount of effort required to prepare and transfer both the data and the required work kernel to a graphics card so that it can be operated on by a GPU. Similarly, there is a nontrivial amount of effort required to retrieve any results generated by OpenCL. As a result, any tasks that interact with the system are generally not recommended for use with OpenCL. For example, you would not use OpenCL to process data from files or network streams. Instead, the work you perform using OpenCL must be much more self-contained so that it can be transferred to the graphics processor and computed independently.

虽然OpenCL执行进行大规模数据并行运算，它是不适合更多的通用计算.普通情况下准备并且将数据以及需要工作的内核传输到图形卡将消耗大量的性能，而这由GPU可以做到。同样，普通情况下需要大量努力来检索的结果OpenCL处理了。因此，任何与系统交互的任务一般不建议使用与OpenCL的。当然，你可以使用OpenCL来处理文件和网络流中的数据。相应的，你使用OpenCL处理的工作必须有更低的耦合度，以便  他可以被转移到图形处理器和独立计算。

For more information about OpenCL and how you use it, see **OpenCL Programming Guide for Mac OS X.**

关于OpenCL以及你该如何使用它的信息，参见**OpenCL Programming Guide for Mac OS X**

#### When to Use Threads 何时使用线程

Although operation queues and dispatch queues are the preferred way to perform tasks concurrently, they are not a panacea. Depending on your application, there may still be times when you need to create custom threads. If you do create custom threads, you should strive to create as few threads as possible yourself and you should use those threads only for specific tasks that cannot be implemented any other way.

尽管操作队列和调度队列提供了并发实现任务的方法，他们也不是万能的。根据你的应用程序，任然有可能在有些时候需要你自定义一些线程。当你创建了一些自定义线程，你应当尽可能努力去少创建线程，只有在没有其他方法时，才不得已为之。

Threads are still a good way to implement code that must run in real time. Dispatch queues make every attempt to run their tasks as fast as possible but they do not address real time constraints. If you need more predictable behavior from code running in the background, threads may still offer a better alternative.

线程任然是实时实现代码的一个好途径，调度队列尽一些努力快速运行他们的任务，但他们并不满足于实时约束，当你需要更清晰的观测在后代运行的代码时，线程任然可以提供一个更好的选择。

As with any threaded programming, you should always use threads judiciously and only when absolutely necessary. For more information about thread packages and how you use them, see **Threading Programming Guide.**

无论如何，你应当只有在绝对必要时谨慎地使用线程，有关线程包的更多信息，以及你如何使用它们，请参见**Threading Programming Guide**。
