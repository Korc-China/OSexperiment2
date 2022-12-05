# 操作系统实验2

## Part1.1
### 猜想运行结果：<br>
先等待5s，而后杀死进程1，再杀死进程2，在两个子进程杀死后结束父进程  
如果按下delete，则不用等待并按顺序直接结束两个子进程，后结束父进程  
### 多次运行截图（未按下delete）：<br>
![image](https://user-images.githubusercontent.com/98074671/205279438-7f5d864b-0efb-4bb9-9afc-e19d0e2d4326.png)
### 多次运行截图（按下delete）：<br>
![image](https://user-images.githubusercontent.com/98074671/205279750-15701fc1-09a8-4155-af24-ddc02c63e233.png)
### 运行结果：
运行结果并不符合猜想：运行程序时会总会直接杀死子进程再等5s  
相同部分：按下delete或quit会直接结束父进程而不用等待5s  
***原因分析：***  
1、为什么会直接杀死子进程：因为我对signal（）函数理解有问题，片面的认为signal收到信号后才会运行后续代码，实际上signal（）函数是执行过后一直在后台，并且会把后续代码执行完毕，如果signal（）函数接收到中断信号，则会调用stop（）（本次实验中handler为stop），所以三个进程同时运行下，可分析得子进程1会早于子进程2结束，同时父进程在sleep（5）（可以分析得出kill在本part中并没有实际作用，在kill之前子进程已经exit了）  
2、为什么按下delete会直接结束父进程：因为进程在睡眠的过程中，如果被中断，那么当中断结束回来再执行该进程的时候，该进程会直接从sleep函数的下一条语句执行；这样的话就不会继续睡眠了。  
3、还有一点要注意的是，程序被中断并执行完中断函数后，也就是在中断函数中返回，那么程序会重新返回到中断前的位置继续执行之前的程序。信号处理函数只能返回void，不能返回指定的参数  
<linux 定时器和sleep,linux中sleep函数的使用和总结>--https://blog.csdn.net/weixin_39865625/article/details/116943755?ops_request_misc=&request_id=&biz_id=102&utm_term=linux%20ctrlc%E5%AF%B9sleep&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-3-116943755.142^v67^js_top,201^v3^add_ask,213^v2^t3_esquery_v2&spm=1018.2226.3001.4187

## Part1.2&3
### 如何修改使得只有接收到对应中断信号再发生跳转：
1、改变为wait_flag[3]，并在各个进程内加入了循环来判断是否接收到对应中断信号，失败  
2、改为三种handler函数，函数内有不同变量，对这三种变量进行判断  
3、在创建完子进程后记录pid，通过if语句在handler函数内判断是哪个进程并改变对应变量（子进程的pid如何区分？）  
***运行出错***<br>
![image](https://user-images.githubusercontent.com/98074671/205431671-c2c5c591-4572-4cf0-8d6a-7db87cf50b25.png)  
***解决方法：***  
上述错误是由于handler函数不能带有参数，若在函数定义时定义了参数，则该参数的值为所接收到的信号值，因此不能使用该种方法  
***运行情况***<br>
图1：未加入lock  
![image](https://user-images.githubusercontent.com/98074671/205540025-3e4d11c9-e39f-4fab-b001-e9d0468968c1.png)   
***解决方法：***  
可以发现：会出现2比1先结束并先printf，这是由于进程的不可预知性  
那么我在代码内部加入了lockf（）函数，实现了运行stop、handler函数时对后续代码lock，而后运行到输出结束后再取消lock，即可保证输出循序的有序  
***运行情况***<br>
图2：加入lock，但没在父进程加入signal（3，stop）  
![image](https://user-images.githubusercontent.com/98074671/205643868-b66e94a7-7875-4812-98dc-53cc887b043a.png)   
***解决方法：***  
可以发现：等5s时，没有再出现顺序错误，但是使用delete时，依旧会出现顺序不定的情况，这是由于***两个子进程复制了signal（2，stop）函数，按下ctrl+c时，所有进程都会调用stop函数，而调用stop的顺序不是有我的代码决定的，此时是由cpu调度决定的，所以出现了顺序不定情况。***  
那么我在父进程中加入了signal（3，stop），这样可以区分于前个监听信号。  
***运行情况***<br>
不可这么用，单独父进程中含有该函数时，quit时会使得两个子进程直接死亡，而父进程继续按原方式执行，因此须在外部设置  
我的想法是利用两个signal，外面的signal来防止子进程死亡（即SIG_IGN），里面的使父进程进入下一步，但是很不幸，虽然解决了子进程死亡的问题，但无法解决顺序问题  
![image](https://user-images.githubusercontent.com/98074671/205650881-ec2e6881-1092-422a-9002-1fa1e1b4f44c.png)  
虽然无法解决问题，但至少有90%情况下1是先于2死亡的(是否需要在每个unlock后加入一个wait_flag=1？)  
***对中断更好的理解***<br>
中断是一种异步的事件处理机制，可以提高系统的并发处理能力  
<硬中断和软中断>-- <https://blog.csdn.net/weixin_43215305/article/details/120027555?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167003506516782395363974%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=167003506516782395363974&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_click~default-2-120027555-null-null.142^v67^js_top,201^v3^add_ask,213^v2^t3_esquery_v2&utm_term=%E8%BD%AF%E4%B8%AD%E6%96%AD&spm=1018.2226.3001.4187>   
<通俗理解软中断>-- <https://blog.csdn.net/weixin_38374686/article/details/119546173?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167003506516782395363974%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=167003506516782395363974&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-119546173-null-null.142^v67^js_top,201^v3^add_ask,213^v2^t3_esquery_v2&utm_term=%E8%BD%AF%E4%B8%AD%E6%96%AD&spm=1018.2226.3001.4187>   






////////// 
***运行出错***<br>
![image](https://user-images.githubusercontent.com/98074671/199521326-3c2d924e-5b96-4599-a1a9-2ef47455647c.png)  
***解决方法：***  
上述错误是由于使用的是cpp文件格式，而程序中一些头文件是c语言的头文件，将文件格式改为c就可以解决  
***运行出错***<br>
![image](https://user-images.githubusercontent.com/98074671/199529996-3fc3328e-64bb-44a9-824a-05057859ebe2.png)  
***解决方法：***  
wait函数所在的c文件中定义了，但是没有在.h文件中声明，此时加入头文件#include <sys/wait.h>即可。  
https://blog.csdn.net/yigedaluobo/article/details/104775533?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-104775533-blog-120277077.pc_relevant_recovery_v2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-104775533-blog-120277077.pc_relevant_recovery_v2&utm_relevant_index=5

### 含wait运行结果
![image](https://user-images.githubusercontent.com/98074671/199627135-1569bd38-344f-4a72-acc2-5bbc9b144b3c.png)
### 不含wait运行结果
![image](https://user-images.githubusercontent.com/98074671/199627262-d0445c64-c842-4010-8b12-804f0ccb8278.png)
### 运行结果：
通过对图片分析，可以得到几个基本结论，即  
子进程中pid = 0;  
子进程pid1 = 父进程pid；  
父进程中pid1 = pid - 1；  
含wait运行结果中，先进行了子进程，而不含wait时，先进行了父进程。  
***原因分析：***  首先：fork会创建一个新进程，这个新进程即为子进程，先在1.1问题中先只讨论运行顺序问题  
进程以单调用了wait就会立即阻塞自己，由wait自动分析该子进程是否已经退出，若未收集到这个已经变成僵尸的子进程，wait就会一直阻塞在那里，直到该僵尸子进程出现（这里我们对子进程如何死掉不在意，使用wait（NULL））那么我们就可以得到与图片相符的分析，即含wait时，子进程先运行完毕并退出，而后父进程退出。 而不含wait时，父子进程退出顺序不确定，不过往往是父进程先退出，因为父进程先于子进程执行。


## Part1.2  
### 1.2.a运行结果  
![image](https://user-images.githubusercontent.com/98074671/199633194-86baaeaa-6397-4e73-84f1-1ff838761de7.png)
***原因分析：***  图中可见var值有变化而location不变，原因是因为该地址只是虚拟地址，而非物理地址，子进程将父进程的虚拟地址复制下来，但基址是不同的，所以他们的物理地址不同而虚拟地址相同
https://blog.csdn.net/pang241/article/details/44343691
### 1.2.b运行结果  
![image](https://user-images.githubusercontent.com/98074671/199633742-a5ca7c72-e63f-4e95-ab63-1f14138e7dfb.png)  
***原因分析：***  图中可见var值均变为3，是由于紧接着父、子进程将var值更改，因此父子进程中var均变为了3  
***运行出错***<br>
![image](https://user-images.githubusercontent.com/98074671/199640822-02e6236f-108f-4e47-99ff-9cae748959d0.png)
***解决方法：***  
system函数所在的c文件中定义了，但是没有在.h文件中声明，此时加入头文件#include <stdlib.h>即可。
### 1.2.c运行结果  
![image](https://user-images.githubusercontent.com/98074671/199641449-031cbeb2-9b26-4371-a402-13486b1c2acd.png)  
![image](https://user-images.githubusercontent.com/98074671/199680538-e0f553f7-30a4-40eb-8586-5e517c439aa5.png)  
***原因分析：***  通过图可见exec族函数PID没变化，因为exec族函数并不产生新进程，进行的PID、PPID等没发生变化，而进程空间变了，即开始执行新程序，旧程序则不执行了，于是后续子进程缓冲区内容便不再输出了，而父进程依然不变。而system函数中先执行了fork，后再执行exec，那么在进程空间变化的同时，PID等也会发生变化，运行完tempprogram便回到子程序中继续执行余下部分，并不会影响父进程的执行。


## Part2.1  
***运行出错***<br> 
![image](https://user-images.githubusercontent.com/98074671/199661320-5e94889d-c02b-40be-a3ae-daf76e856d2c.png)  
***解决方法：***  
由于函数返回的是一个void*指针，因此返回时需要给其赋值NULL以防止空指针乱指  
***运行出错***<br> 
![image](https://user-images.githubusercontent.com/98074671/199662143-233a00f1-3364-43b9-919e-ea3fa80e1117.png)  
***解决方法：***  
因为pthread并不是linux下的默认库，linux没有真正的线程，都是通过进程模拟的，所以需要加入-lpthread以解决问题https://blog.csdn.net/mao_hui_fei/article/details/99831013  
### 2.1运行结果  
![image](https://user-images.githubusercontent.com/98074671/199664840-ec061e04-7a56-47e2-b71b-ca9b0bf9eb55.png)  
***原因分析：***  通过将pthread1内部每一次都输出可以发现在还没到达6000时主线程就已经结束，是因为没有加入pthread_join(),加入后即可以阻塞的方式等待指定线程结束，即可防止主线程结束过快  
![image](https://user-images.githubusercontent.com/98074671/199662728-5914212c-c1c6-4196-bd55-bac316e94690.png)  
***原因分析：***  可以发现每次运行结果都不同，这是由于两个线程同时运行，且线程共享数据，导致var同时由两个线程进行操作，使得结果不确定  
![image](https://user-images.githubusercontent.com/98074671/199675080-a5d9f85f-c551-44c5-8030-a3bf24171ee2.png)  
***原因分析：***  加锁后，可以保证寄存器和内存的值同时刻相同，那么只是pthread1和2先后运行完毕的问题，由图可以得到，var=17265前，两线程交互使用非临界资源，这之后只有2在运行。最终值也可以体现出该程序的正确性18000=6000+12000。  
### 2.2运行结果  
![image](https://user-images.githubusercontent.com/98074671/199686235-1cc8598f-678f-4747-b95d-20ebcc2fd1fe.png)  
***原因分析：***  system创建了一个新的进程，所以其pid通线程中的pid不同，且本身不是同一进程下的线程，所以tid不同  
![image](https://user-images.githubusercontent.com/98074671/199688444-3cf24a1c-dded-48a5-bd4b-24fd5c719ce9.png)
***原因分析：***  exec创建了一个新的进程空间，其pid不变，但进程空间变了，所以线程中后续代码不再执行了  
https://blog.csdn.net/weixin_42250655/article/details/105234980

***附：***  ssh基本用法：https://zhuanlan.zhihu.com/p/21999778  
ssh命令详解：https://www.cnblogs.com/ftl1012/p/ssh.html  
github Readme编写方法：https://blog.csdn.net/Strive_For_Future/article/details/120956016  
线程：https://blog.csdn.net/m0_52640673/article/details/123533809
