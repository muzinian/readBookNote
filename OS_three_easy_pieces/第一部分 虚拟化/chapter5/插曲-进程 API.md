>#### 旁注：插叙
>插叙会更多的覆盖系统的实践方面，包括对操作系统 API 以及如何使用这些 API 特别的关注。如果你不喜欢实践的东西，你可以跳过这些插叙。但是你应该喜欢这些，因为，它们在现实中很有用；举个例子，公司不会雇佣你，仅仅是你为了那些没有实践过的技能。

在这篇插叙中，我们会讨论 UNIX 系统中进程的创建。UNIX 通过使用`fork()`和`exec()`这一对系统调用创建进程，这是创建一个新进程的最奇妙的方式之一。一个进程可以使用`wait()`来等待它创建的进程完成。我们现在会展现这些接口更多细节，通过一些简单的例子来刺激我们。我们的问题是：
>#### 症结：如何创建并控制进程
>OS 需要为创建和控制进程展现什么接口？这些接口如何设计才能兼顾易用和高效？

# 5.1 `fork()`系统调用

`fork()`系统调用用来创建新进程。然而先提前告知下：他应该是你会调用的最奇怪[<sup id='content1'>1</sup>](#1)的例程了。进一步说，你有一个代码如下的正在运行的程序，检查一下代码(p1.c)，或者更好的，键入并执行下！
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main(int argc,char *argv[])
{
    printf("hello world (pid:%d)\n",(int)getpid());
    int rc = fork();
    if(rc<0){//fork failed;exit
        fprintf(stderr,"fork failed\n");
        exit(1);
    }else if(rc == 0){//child(new process)
        printf("hello, I am child (pid:%d)\n",(int)getpid());
    }else{
        printf("hello , I am parent of %d (pid:%d)\n",rc,(int)getpid());
    }
    return 0;
}
```
当你运行这个程序，你可以看到如下结果：
```Sh
$ ./callFork
hello world (pid:9083)
hello , I am parent of 9084 (pid:9083)
hello, I am child (pid:9084)

```
现在，让我们看看在 p1.c 的细节。当它开始运行时，进程答应一个 hello world 消息；这个信息还包含了这个进程的 PID，在这里，PID 是9083，在 UNIX 系统中，如果有人想用进程做些事情，例如停止这个进程，就需要用 PID 命名一个进程，然后操作这个进程。到现在，都还好。

现在，有意思的事情开始了。进程调用`fork()`系统调用，OS 通过它来创建一个新进程。奇怪的部分是：通过`fork()`创建的进程基本上是调用者的完全拷贝。这意味着，对于 OS 来说，现在有 p1 的两份拷贝在运行，它们都要从`fork()`系统调用返回。新建的进程(相对于创建它的进程，叫做子进程，而创建者，叫做父进程)并没有从`main()`函数开始运行，就像你可能期望的(记住，"hello world"信息只打印了一次);相反，他就好像是已经执行过了`fork()`函数。

你可能注意到了：子进程并不是“完全的”拷贝。进一步，尽管它现在有了自己的地址空间副本(例如，它自己私有的内存)，它自己的寄存器，它自己的程序计数器(PC)等等，但是从`fork()`函数返回的值是不同的。特别的，当父进程接收到新建的子进程的 PID 时，子进程接收到的是0。这个区别很有用，因为当写那些需要控制不同情况的代码时(就想上面那样)，这样做的话就会很简单。(注：这里，fork()函数的返回值有两个，对于子进程，就是0，对于父进程，返回的是子进程的 PID)。

你可能还注意到了：`p1.c`的输出是不确定的。当子进程被创建，系统中就有两个活跃的进程是我们关心的，父进程和子进程。为了简化问题，假设我们系统运行在单个CPU上，那么，在这个时刻，父进程和子进程中的一个可能在运行。在我们的例子里，父进程先运行，然后打印出信息。在其他例子中，可能会是相反的情况，结果可能如下：
```Sh
prompt> ./p1
hello world (pid:29146)
hello, I am child (pid:29147)
hello, I am parent of 29147 (pid:29146)
prompt>

```
CPU**调度器(scheduler)** (很快我们就会仔细的讨论的这个主题)决定在给定的某个时刻哪个进程运行；由于调度器很复杂，我们通常不能假设他会选择哪个进程执行，也就无从知道那个进程会先执行。正如它显示的样子，这种不确定性会导致一些有趣的问题出现，特别是在**多线程程序中(multi-threaded programs)**，因此，在我们学习本书第二个部分**并发(concurrency)** 时，我们会看到很多更加不确定的事情。

# 5.2 `wait()`系统调用

到目前为止，我们还没有做很多事：只是创建了一个打印一个消息然后退出的子进程。有时，让一个父进程等待子进程完成然后再完成它(这里应该指的是父进程)已经开始做的事情。这个任务通过`wait()`系统调用可以完成(或者它的更完整的兄弟`waitpid()`)；下面的程序(`p2.c`)显示了具体的细节。
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) { // fork failed; exit
    fprintf(stderr, "fork failed\n");
    exit(1);
    } 
    else if (rc == 0) { // child (new process)
    printf("hello, I am child (pid:%d)\n", (int) getpid());
    }
     else {  // parent goes down this path (main)
     int wc = wait(NULL);
     printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",rc, wc, (int) getpid());
     }
      return 0;
}
```
在这个例子中(`p2.c`)，父进程调用了`wait()`来延迟它的执行知道子进程完成执行。一旦子进程完成，`wait()`返回到父进程中。通过在代码中加入了`wait()`调用使得代码的输出是确定的了。结果输出如下：

```Sh
prompt> ./p2
hello world (pid:29266)
hello, I am child (pid:29267)
hello, I am parent of 29267 (wc:29267) (pid:29266)
prompt>
```
使用这个代码，我们现在就可以知道子进程总是会先打印输出。为什么我们会知道呢?有可能就是子进程就是先执行的进程，像之前那样，在父进程之前先打印输出。然而，如果是父进程碰巧先运行，他会立刻调用`wait()`，这个系统调用直到子进程运行结束并退出后[<sup id='content2'>1</sup>](#2) 才会返回。因此，即使是父进程先运行，它也会等待子进程我弄成运行，然后`wait()`返回，然后父进程再打印自己的消息。

# 5.3 最后，`exec()`系统调用

进程创建API中最后一个重要的部分就是`exec()`系统调用[<sup id='content3'>3</sup>](#3)。但你想要运行一个和调用(calling)程序不同的程序时，这个系统调用很有用。举个例子，在`p2.c`中，只有你希望保持运行同样程序的副本时调用`fork()`才有用。然而，常常是你希望运行不同的程序。`exec()`就是用来做这个的。来看下面的例子(`p3.c`)
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) { // fork failed; exit
    fprintf(stderr, "fork failed\n");
    exit(1);
    } else if (rc == 0) { // child (new process)
    printf("hello, I am child (pid:%d)\n", (int) getpid());
    char *myargs[3];
    myargs[0] = strdup("wc"); // program: "wc" (word count)
    myargs[1] = strdup("p3.c"); // argument: file to count
    myargs[2] = NULL; // marks end of array
    execvp(myargs[0], myargs); // runs word count
    printf("this shouldn’t print out");
    } else { // parent goes down this path (main)
    int wc = wait(NULL);
    printf("hello, I am parent of %d (wc:%d) (pid:%d)\n",rc, wc, (int) getpid());
    }
    return 0;
}
```
在这个例子中，子进程调用了`execvp()`，用来运行统计词语个数的程序`wc`。事实上，它在`p3.c`中运行了`wc`，它告诉了文件中有多少行，多少个单词以及字节。输出如下：
```sh
prompt> ./p3
hello world (pid:29383)
hello, I am child (pid:29384)
29 107 1030 p3.c
hello, I am parent of 29384 (wc:29384) (pid:29383)
prompt>
```
`fork()`调用很奇怪，它的同伙，`exec()`也不遑多让。这里`exec()`做的是：给定一个可执行程序(例如`wc`)和一些参数(例如`p3.c`)，它就从可执行程序加载代码和静态数据，然后覆写(overwrite)它自己当前的代码段(也包括当前的静态数据)，程序的堆栈和其它内存空间被重新初始化。然后OS就只需运行那个程序，并将传入的任何参数当作那个进程的`argv`。它不会创建新的进程，相反，它转换当前运行的程序(之前的p3)为另外的运行的程序(wc)。在子进程中的`exec()`后，就好像从来`p3.c`从来没有运行过，对于一个成功的`exec()`调用，是永远不会返回的。

# 5.4 这些API的动机

当然，你可能会有疑问：为什么要为创建一个进程这么简单的操作设计如此奇怪的接口？事实证明，在构建 UNIX shell时，把`fork()`和`exec()`分离是很有效的，这样就让`shell`可以在`fork()`之后`exec()`调用之前运行代码；这个代码可以改变即将运行的程序的环境，确保各种有趣的功能能够容易地被构建。

>#提示：让事情正确(LAMPSON's LAW)
>正如Lapson 在他著名的"Hints for Computer Systems Design"中声明的，"**Get it right.** 无论抽象还是简洁都不是让事情正确的替代品。"有时，你只用做正确的事，当你做了，他就比任何替代方法要好。对于创建进程的API设计，有很多方法，然而，`fork()`和`exec()`的组合是简单而非常有力的。这里，UNIX 的设计者只是简单的让事情正确。由于 Lampson 总是能够"got it right"，我们以他的名字命名了这个定律。

`shell`程序也只是一个用户程序。它向你展示了一个**命令提示符(prompt)** 然后等待你的输入。你输入了一个命令(例如，执行程序的名字和任意的参数)，在大多数例子里，`shell`找到执行程序的所在的文件系统，调用`fork()`创建新的子进程来运行命令，调用`exec()`的各种变体懒运行命令，通过`wait()`来的等待命令的完成。当子进程完成了，`shell`从`wait()`返回，然后再次打印命令提示符，等待你的下一个命令。

`fork()`和`exec()`的分离使得`shell`可以很简单的做一大堆有用的事情，例如：
```sh
prompt> wc p3.c > newfile.txt
```
上面的例子里，`wc`的输出被**重定向(redirected)** 到输出文件`newfile.txt`中了(大于符号表示重定向)。`shell`完成这个任务的方式很简单：当子进程被创建，在调用`exec()`之前，`shell`关闭了**标准输出(standard output)** 然后打开了`newfile.txt`文件。通过这样做，任何来自即将执行的`wc`程序的输出都会被发送到这个文件里面，而不是屏幕。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/wait.h>
int main(int argc, char *argv[])
{
    int rc = fork();
    if (rc < 0) { // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } 
    else if (rc == 0) { // child: redirect standard output to a file
        close(STDOUT_FILENO);
        open("./p4.output", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);
        // now exec "wc"...
        char *myargs[3];
        myargs[0] = strdup("wc"); // program: "wc" (word count)
        myargs[1] = strdup("p4.c"); // argument: file to count
        myargs[2] = NULL; // marks end of array
        execvp(myargs[0], myargs); // runs word count
    } else { // parent goes down this path (main)
        int wc = wait(NULL);
    }
    return 0;
}
```
上面的代码做了同样的事。重定向的工作的原因是对操作系统系统如何管理文件描述符做了一个假设。特别的，UNIX 统从零开始查找可用的文件描述符。在这个例子中，STDOUT_FILENO 将会是第一个可用的文件描述符，因此，当`ope()`被调用后就会被赋值。随后，就通过子进程开始向标准输出文件描述符写东西了。以`printf()`例程为例，它会透明的路由到新开的文件而不是屏幕。
这里是运行`p4.c`程序的输出：
```sh
prompt> ./p4
prompt> cat p4.output
32 109 846 p4.c
prompt>
```
你可能注意到了这个输出中(至少)两个有趣的事。第一，当`p4`运行了，他看起来好像什么都没有发生，`shell`只是打印了命令提示符然后立刻准备等待你的下一个命令。然而，这不是事实，`p4`程序确实调用了`fork()`来创建新的子进程，然后通过调用`execvp()`运行了`wc`程序。你没有在屏幕上看到任何输出是因为它被重定向到了`p4.output`。第二，你可以通过`cat`命令看输出文件，所有希望来自`wc`的输出都可以找到。

UNIX 管道(pipe)也是以同样的方式实现的，但是使用的是`pipe()`系统调用。在这个例子里，一个进程的输出被链接到了一个在内核里面的**管道(pipe)** ，另一个进程的输入也被连接到了同样的管道上，然后，一个进程的输出被无缝的用作另一个进程的输入，多个命令可以通过这样的方式连接到一起。举一个简单的例子，考虑在一个文件中找一个单词，然后统计这个单词出现了多少次；通过使用`grep`，`wc`和管道，这很容易做到————只用在命令提示符输入`grep -o foo file | wc -l`然后奇迹就发生了。

最后，我们只是再一个高层次的水平草草地描绘了进程API，关于这些调用还有很多细节等着学习笑话；我们将会在后面学习更多的知识，例如，当我们在本书的第三部分讨论文件系统的时候会学习有关文件描述符的知识。现在，我们就满足于知道`fork()`和`exec()`的组合是创建并操作进程的有力方式就可以了。

>#附注：RTEM——阅读 MAN 手册
>在本书的多次提到，当涉及到具体的系统调用或者库调用，我们会简单地告诉你阅读 **手册(manual pages or man pages)**。Man手册是UNIX系统中文档的原始形式；记住，它们早于web网页而存在。
>
>花些时间阅读man手册是成长为系统程序员的一个关键步骤；隐藏在手册里面有很多由于的有用的知识点。无论你是使用那种`shell`(例如**tcsh** 或**bash**)有些特别有用的手册都应该阅读，这必然包括那些你程序中要使用的系统调用(为了搞清楚返回值是什么和退出的错误条件)。
>
>最后，阅读man手册可以让你不至于手足无措。当你问同时关于`fork()`的复杂问题时，他们可能会简单的说:RTFM。这里你的同事就要求你去读手册，不过这里F有点。。。

# 5.5 其它API
除了`fork()`,`exec()`和`wait()`，还有很多接口可以用来跟UNIX系统的进程交互。例如，`kill()`系统调用被用来给进程发送**信号(signals)**，包括指示进程`sleep`,`die`和其它有用的指令。事实上，整个信号子系统提供了丰富的架构来给进程发送外部的事件，包括接受和处理那些信号。

还有很多有用的命令行工具。例如，使用`ps`命令可以查看正在运行哪些进程，查看手册了解更多关于`ps`的信息。`top`命令显示了系统的进程，也显示了每个进程占用了多少CPU和其它资源。有趣的是，有很多次在你运行这个命令时，`top`都显示它自己是最占用资源的程序，这里可能有点自大。最后，有很多种CPU测量方式供你使用，这样你就可以快速的理解你系统的负载；举个例子，我们使用运行在我们自己的Mac工具箱的MenuMeters(来自 Menace software)，这样我们就可以了解到在任意时刻有多少CPU被使用。总的来说，关于事情走向的消息越多越好。
# 总结
我们介绍了用来创建UNIX进程的部分API:`fork()`,`exec()`和`wait()`。然而，我们只是看到皮毛。更多细节，请阅读Stevens和Rago的UNIX高级编程(Advanced Programming in UNIX Environment)，当然，特别注意关于进程控制，进程关系和信号的章节。那里有很多有用的东西。


<sup id='1'>1</sup>好吧，我们承认我们不知道这是不是真的；天知道在没人看到的情况下，你会调用什么样的例程？但是`fork()`函数确实很奇怪，无论你的例程调用模式多么不同寻常[&crarr;](#content1)。

<sup id='2'>2</sup>有几种可能回导致`wait()`在子进程退出前返回，跟以往一样，参看 man 获取更详细的信息。同时，小心本书做出的任何绝对且不合格的语句。比如，"子进程总是先打印"或者"UNIX 是世界上最好的事情，甚至好于冰淇淋"[&crarr;](#content2)。

<sup id='3'>3</sup>事实上，有六个`exec()`的变种：`execl()`，`execle()`，`execlp()`，`execv()`和`execvp()`。参看 man [&crarr;](#content3)。