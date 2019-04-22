##插曲：线程API
本章简单的覆盖了线程API主要部分。每个部分将会在后续的章节被进一步介绍，因此我们展示如何使用这个API。更多细节可以在各种书和在线资源看到。我们应该注意到后续章节通过很多例子慢慢介绍锁和条件变量的概念，本章最好被用来作为一个引用。
>### 症结：如何创建和控制线程
>OS为了线程创建和控制应该呈现什么接口？这些接口应该被怎么设计从而确保易用和实用？
###27.1 线程创建
在编写多线程程序第一个要做的事就是创建新线程，所以必须要存在某种创建线程的接口，在POSIX中，很简单：
```c
#include <pthread.h>
int pthread_create(pthread_t *thread,
                   const pthread_attr_t *attr,
                   void *(*start_routine)(void *),
                   void *arg);
```
这个声明看起来有点复杂(特别是如果你没有使用过C语言的函数指针)，但是实际上没有那么糟糕。这里有四个参数，`thread`，`attr`，`start_routine`和`arg`。第一个形参，`thread`，是指向`pthread_t`类型结构的一个指针。我们使用这个结构与线程交互，所以我们需要传递给`pthread_create()`从而初始化它。

第二个形参，`attr`，用来指定这个线程可能拥有的任何attributes。有些例子包括栈大小或者关于这个线程调度优先级的信息。attribute的初始化是在`pthread_attr_init()`单独调用的，查看手册了解跟多细节。然而，在大多数情况中，默认值就工作的很好了；这种情况下，我们就只用传入`NULL`值。

第三个形参是最复杂的，但它只是再问：这个线程应该从哪个函数开始执行？在C中，我们称这个是 __函数指针(funciton pointer)__，这个参数告诉我们参数应该满足：一个名字叫做`(start_routine)`，它只接受一个类型是`void *`的参数(由在`start_routine`后的小括号表示)，并返回一个`void *`类型的值(例如，一个 __void指针(void pointer)__)。

如果这个例程有要求一个整型参数而不是一个void指针，声明可能是如下：
```c
int pthread_create(..., // first two args are the same
                   void *(*start_routine)(int),
                   int arg);
```
如果这个例程接收一个void指针做形参，但是返回一个整型，声明可能如下：
```c
int pthread_create(..., // first two args are the same
                   int (*start_routine)(void *),
                   void *arg);
```
最后，第四个形参，`arg`，就是线程在开始传递给将要执行的那个函数的实参。你可能会问，为什么我们需要这是void指针？答案很简单：使用void指针做`start_routine`函数的形参让我们可以传递 _任意_ 类型的参数；让void指针做返回值允许线程返回 _任意_ 类型做为结果。

让我们看一个例子。这里我们创建了一个接受两个参数的线程，这两个参数被打包到我们为自己定义的一个类型(`myarg_t`)。这个线程，一旦被创建，可以简单的把参数转换为它期待的类型然后按照需要解压这些参数。

这就是了！一旦你创建一个线程，你就拥有了另一个活跃的执行实体，拥有自己的调用栈，和这个程序中所有的当前已存在线程运行在 _同一个_ 地址空间。游戏开始了！
###27.2 线程完成
上面的例子显示了如何创建线程。但是，如果你想等待线程完成会发生什么？为了等待完成，你需要做一些特别的事情；具体来说，你需要调用`pthread_join()`例程。
```c
int pthread_join(pthread_t thread, void **value_ptr);
```
这个例程需要两个参数。一个参数是`pthread_t`类型，用来指定需要等待的是哪个线程。这个变量是线程创建例程初始化的(当你给`pthread_create()`传入一个指向这个类型的指针)；如果你一直保持这这个指针，你可以使用它来等待他对应的线程终结。

第二个参数是指向返回值的指针。由于调用例程可以返回任何值，这个参数被定义为一个指向void的指针；由于`pthread_join()`例程会 _修改_ 传入的参数值，所以你需要一个指向值的指针，而不是值本身(意思是你需要一个指向指针的指针)。
```c
#include <stdio.h>
#include <pthread.h>
#include <assert.h>
#include <stdlib.h>

typedef struct __myarg_t
{
    int a;
    int b;
} myarg_t;

typedef struct __myret_t
{
    int x;
    int y;
} myret_t;

void *mythread(void *arg)
{
    myarg_t *m = (myarg_t *)arg;
    printf("%d %d\n", m->a, m->b);
    myret_t *r = Malloc(sizeof(myret_t));
    r->x = 1;
    r->y = 2;
    return (void *)r;
}

int main(int argc, char *argv[])
{
    pthread_t p;
    myret_t *m;

    myarg_t args = {10, 20};
    Pthread_create(&p, NULL, mythread, &args);
    Pthread_join(p, (void **)&m);
    printf("returned %d %d\n", m->x, m->y);
    free(m);
    return 0;
}
```
让我们看看另一个例子。在这个代码中，再一次创建了单一线程，然后通过`myarg_t`结构传入了一组参数。通过使用`myret_t`类型接收返回值。一旦线程运行完成，main线程，已经在`pthread_join()`例程[<sup id ="content1">1</sup>](#1)中等待着，返回，然后我们可以访问线程返回的值，也即任何放在`myret_t`的值。

关于这个例子有几点需要注意。首先，通常我们不需要做所有这些痛苦的参数打包解包操作。例如，我们构造线程是不传递参数，就可以在线程创建的时候传入`NULL`作为参数。类似的，如果我们不关心返回值，就可以在`pthread_join()`中传入`NULL`。

第二，如果我们只是传入一个单值(例如，一个int)，我们就不必打包成一个参数了。下面的代码显示了这个例子。在这个例子中，什么都很简单，我们不用包装参数和返回值到一个结构中。
```C
void *mythread(void *arg)
{
    int m = (int)arg;
    printf("%d\n", m);
    return (void *)(arg + 1);
}
int main(int argc, char *argv[])
{
    pthread_t p;
    int rc, m;
    Pthread_create(&p, NULL, mythread, (void *)100);
    Pthread_join(p, (void **)&m);
    printf("returned %d\n", m);
    return 0;
}
```
第三，我们应该要特别对如何从线程返回值特别仔细。具体的说，永远不要返回从线程调用栈分配的东西。如果你这样做了，你认为会发生什么？(仔细想想！)这里有一个危险的代码例子：
```C
void *mythread(void *arg)
{
    myarg_t *m = (myarg_t *)arg;
    printf("%d %d\n", m->a, m->b);
    myret_t r; // ALLOCATED ON STACK: BAD!
    r.x = 1;
    r.y = 2;
    return (void *)&r;
}
```
在这个例子中，变量`r`是从`mythread`的栈上分配的。然而，当它返回时，这个值被解分配了(deallocated)(这也是为什么栈这么容易使用的原因)，然后，传递回一个指向已经解分配变量的指针会导致各种类型的糟糕结果。但你打印你认为你返回的值，你可能会吃惊。自己尝试一下。[<sup id ="content2">2</sup>](#2)

最后,你可能注意到使用`pthread_create()`创建线程的用法，紧跟着立即调用`pthread_join()`，是一个很奇怪的方式创建线程。事实上，有一个更简单的方式来完成这个任务；叫做 __过程调用(procedure call)__。显然，我们通常会创建不止一个线程并等待它们完成，不然就没有什么目的使用线程了。

我们应该注意，并不是所有多线程代码都使用join例程。例如，一个多线程web服务器可能创建一些工作线程，但后使用main线程接收请求并传递给工作线程。这样一个长期运行的程序就不需要join。然而，一个并行程序创建线程执行特定的任务(并行方式)很可能使用join来确保所有这些工作在退出或者移动到下一阶段计算前都完成了。

###27.3 锁
除了线程创建和join，可能下一个由POSIX线程库提供的最有用的函数集合是通过 __锁__ 对关键区互斥操作。最基本的一对针对这个目的使用的例程如下：
```C
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```
这对例程应该很好理解和使用。当你有一段代码是 __关键区__，它们需要被保护以确保正确的操作，锁就很有用了。你可能想象代码如下：
```C
pthread_mutex_t lock;
pthread_mutex_lock(&lock);
x = x + 1; // or whatever your critical section is
pthread_mutex_unlock(&lock);
```
这段代码的意义如下：当调用`pthread_mutex_lock()`时没有其他线程持有锁，这个线程就会请求到锁然后进入关键区。如果其它线程确实持有锁，线程尝试获取锁将不会从调用返回直到它成功获取到锁(意味着原先持有锁的线程通过了unlock调用释放了锁)。当然，在给定的时刻很多线程可能阻塞在等待锁申请函数中；只有获取到锁的线程可以调用解锁操作。

不幸的是，这段代码是错误的，在两个重要的方面。首先第一个问题是 __缺少恰当的初始化(lack of proper initialization)__。所有锁都必须被恰当的初始化，为了保证它们有正确的值开始，然后在按照所需的在加锁和解锁时正确工作。

使用POSIX线程，有两个方法初始化锁。一个是使用`PTHREAD_MUTEX_INITIALIZER`，如下：
```C
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```
这样可以给锁设置一个默认值从而让锁可用。如果想要动态的使用(例如，在运行时)，可以调用`pthread_mutex_init()`，如下：
```C
int rc = pthread_mutex_init(&lock, NULL);
assert(rc == 0); // always check success!
```
第一个参数是锁的地址，第二个参数是可选的属性(attributes)集合。请自己去了解这些属性跟多的内容，这里传递`NULL`就是使用默认值。两种方法都可以工作，但是我们通常使用动态(后一种)的方法。记住对应的函数调用`pthread_mutex_destory()`在你用玩锁时调用，查看手册了解更多细节。

上面代码第二个错误是调用加锁和解锁的时候没有检查错误码。就像几乎所有你在UNIX系统中调用的库例程，这些例程本身也会失败！如果你的带啊吗没有良好的检测错误码，这些错误会静默失败，这会导致多个线程进入关键区。最简单的，使用包装方法，这些包装方法会断言例程的成功(如下代码),更优雅的(不是玩具)程序，当出现问题不是简单的退出，应该检测错误并在加锁或者解锁失败的时候做一些恰当的事情。
```C
// Use this to keep your code clean but check for failures
// Only use if exiting program is OK upon failure
void Pthread_mutex_lock(pthread_mutex_t *mutex)
{
    int rc = pthread_mutex_lock(mutex);
    assert(rc == 0);
}
```
加锁和解锁例程不是pthreads库中唯一和锁交互的例程。具体的，还有两个有意思的例程：
```C
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex,struct timespec *abs_timeout);
```
这两个调用用于对锁的获取过程。`trylock`调用会在锁已经被持有时返回失败；`timedlock`版本请求锁会在超时或者请求到锁后返回，无论谁先发生。如果`timedlock`的超时时间是零就退化为`trylock`。通常这些版本都应该避免；然而，有某些情况下为了避免锁获取例程被阻塞(可能是一直阻塞)它们就很有用，我们将在后面的章节了解到(例如，我们当我们学习什么是死锁)。
###27.4 条件变量
所有线程库另一个重要部分，当然也包括POSIX线程，就是 __条件变量(condition variable)__。条件变量在进程间互相发送某种信号时是很有用的，如果一个线程正在等待另一个线程完成后才能继续。程序中使用这两个主要例程以这种方式交互：
```C
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```
为了使用条件变量，需要有一个锁和这个条件相关联。当上述两个例程任意一个被调用，锁应该是被调用线程持有的。

第一个例程，`pthread_cond_wait()`，会把调用线程设置为sleep，然后等待其它线程发送信号给它，通常当程序中有某些当前睡眠的线程可能会关心的东西被改变了。一个典型的使用方式如下：
```C
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
Pthread_mutex_lock(&lock);
while (ready == 0)
    Pthread_cond_wait(&cond, &lock);
Pthread_mutex_unlock(&lock);
```
在这个代码中，在对相关锁和条件变量初始化后[<sup id="content3">3</sup>](#1)，一个线程检查变量`ready`是否已经被设置为除0外的其它值。如果没有被设置，这个线程就调用wait例程进入睡眠直到其它线程唤醒它。

唤醒线程的代码，是运行在其它线程中，模式如下：
```C
Pthread_mutex_lock(&lock);
ready = 1;
Pthread_cond_signal(&cond);
Pthread_mutex_unlock(&lock);
```
这个代码序列有一些事情需要注意。首先，当发送信号(signaling)时(也就是当修改了全局变量`ready`)，我们总是确保已经持有锁。这确保了我们不会意外的在我们代码里面引入竟态条件。

第二，你可能注意到wait调用使用锁作为第二个参数，但是signal调用只用了一个条件变量。存在这种差别的原因是wait调用，除了要把调用线程设置为睡眠，还同时要 _释放(releases)_ 所说调用者持有的锁。想象一下如果没有释放锁：其它线程如何申请这个锁然后发送信号给这个线程唤醒这个线程？被唤醒后返回 _前_，`pthread_cond_wait()`重新申请锁，从而确保在任何时间，等待中的线程在从在wait代码序列开始阶段申请锁到末尾释放锁之间整个运行过程中都持有锁。

最后一个古怪的地方：等待线程在`while`循环中重新检查条件，而不是简单的`if`语句。在后间学些条件变量的章节中我们会讨论这个问题的细节，但是总的来说，使用一个`while`循环是简单且安全的事情。尽管它重新检查了条件(可能增加了一点消耗)，有某些pthread实现可能会意外的唤醒一个等待线程；在这种情况下，没有重新检查，等待线程会继续执行，因为他认为条件已经改变了，尽管并没有。把唤醒操作作为一个某些事情已经被改变了的提示比看成绝对的事实更安全。

有时候会有尝试使用一个单一标志在两个线程间做signal操作，而不是使用条件变量和关联的锁。例如，我们可以重写上面的等待代码如下：
```C
while (ready == 0)
    ; // spin
```
相关的singaling代码可能如下：
```C
ready = 1;
```
永远不要着要做，由于如下原因。首先，它在大多数情况下运行的很糟糕(长时间空转只是浪费CPU循环)。第二，它容易出错。最近的研究显示，它意外的容易出错，在使用标志flags(就像上面的代码)同步线程；研究指出，使用上述(_ad hoc_)的同步中有大概一半是充满bug的。不要懒惰，使用条件变量，即使当你认为在不使用条件变量的情况下你可以完成任务。

如果条件变量听起来让人困惑，不要太过于担心(暂时)-在后面的章节我们会事无巨细的介绍它们。到那个时候，你应该有能力知道它们存在并且对于如何以及为何使用它们有了一些了解。
###完成并运行
本章所有代码样例都相对简单构造并运行。为了编译它们，你必须要在代码中包含`pthread.h`头文件。在链接行，你也必须要显示链接pthreads库，通过添加`-pthread`标志。

例如，为了编译一个简单的多线程程序，所有你需要做的如下：
```sh
prompt> gcc -o main main.c -Wall -pthread
```
只要`main.c`包含了pthreads头，现在你就成功的编译了一个并发程序。无论它工作与否，就想平常一样，就完全是另外的事了。
###27.6 总结
我们已经介绍了pthread库的基本情况，包括线程创建，构建通过锁构建互斥，通过条件变量signaling和waiting。你不需要其它更多的东西就可以编写健壮和高效的多线程代码，除了耐心和特别仔细！

我们现在要以一些在你编写多线程代码(在下面的附注里面)时可能对你有用的tips来结束本章。他们是这些API其它有趣的方面；如果你想要更多信息，在Linux系统命令行输入`man -k pthread`，可以看到超过一百个用来构建这个接口的API。然而，这里基本的讨论应该可以让你构建优雅(希望还是正确和高性能)的多线程程序。关于线程困难的部分不是API，而是关于你应该如何构建正确的并发程序的机智逻辑。请继续阅读学习更多相关内容。
>###附注：线程API指引
>当你使用POSIX线程库(或者，任意线程库)时，这里有一些短小重要的建议用以构建一个多线程程序。它们是：
>* __保持简单__。在其他之上，任何代码用来在锁定或者在线程之间发送信号(signal)都应该尽可能简单。复杂的线程交互导致bugs
>* __最小化线程交互__。试着保持线程交互方式的数量最小。每一中交互方式都应该是经过仔细的思考并且被按照真正的方式(我们将在接下来的章节中学到大多数的方法)可靠的构造。
>* __初始化锁和条件变量__。如果不这样做将会导致代码有时候可以工作有时候以非常奇怪的方式不可以工作。
>* __检查你的返回码__。当然，你编写的任意C和UNIX程序你都应该检车每一个返回码，这里也是真理。不这样做将会导致奇怪且难于理解额行为，很可能让你a)尖叫，b)拔你的头发，或者c)两者都会。
>* __注意你传递参数的方式，以及从线程返回值的方式__。特别的，任何时候你在传递一个分配在栈上面的引用，你都可能在做错误的事情。
>* __每个线程都有自己的调用栈__。这个上一点相关。请记住每个线程都有自己的调用栈。因此，如果你已经在一个线程正在执行的某个函数中有了本地分配的变量，他本质上对这个线程来说就是 _私有的_ 了；其它线程不能(_轻易的_)访问它。为了在线程间共享数据，这个值要么在 __堆__ 上，要么位于某些可以全局访问的地方。
>* __总是使用条件变量在线程直接发送信号(singal)__。总是会冒着风险尝试使用简单的flag，不要这样做。
>* __使用手册__。在Linux上，具体来说，pthread的man手册有很高的教育意义并且讨论很多这里讲的东西的细微差别，通常更加具体。仔细读它们。


[<sup id="1">1</sup>](#content1) 记住，这里我们使用了包装函数；具体来说，我们调用`Malloc()`，`Pthread_join()`和`Pthread_create()`，它们调用了类似名称的小写版本并确保这些例程不会返回任何异常。
[<sup id= "2">2</sup>](#content2)幸运的是，当你编写的代码是这个样子的时候`gcc`编译器会抱怨，这也是另一个要注意编译器警告的理由。
[<sup id = "3">3</sup>](#content3)注意，还可以使用`pthread_cond_init()`(和对应的`pthread_cond_destroy()`调用)而不是静态初始化器`PTHREAD_COND_INITIALZER`。看起来要做更多的工作？是的。