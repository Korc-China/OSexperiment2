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

## Part2
### 管道相关关键知识：
1. 父进程调用pipe函数创建管道，得到两个文件描述符fd[0]、fd[1]指向管道的读端和写端。  
2. 父进程调用fork创建子进程，那么子进程也有两个文件描述符指向同一管道。  
3. 父进程关闭管道读端，子进程关闭管道写端。父进程可以向管道中写入数据，子进程将管道中的数据读出。由于管道是利用环形队列实现的，数据从写端流入管道，从读端流出，这样就实现了进程间通信。 
### 猜想运行结果： 
猜想运行结果：先输出2000个1，再输出2000个2.
如何实现同步：管道本身给读写双方提供了同步处理，可以简单处理实现“没写完不能读”，“没有读空缓冲区不能写”。而在我们的程序中wait也可以充当同步这一角色
如何实现互斥：利用lockf实现了护持，在其中一个子进程写时，其他子进程无法写。
***运行出错***<br>
![image](https://user-images.githubusercontent.com/98074671/205800304-3e9ce720-dc62-4ea1-8359-98b703648955.png)  
***解决方法：***  
读写函数的第二个参数需要的是字符串首地址，而原代码中给的是char类型，因此报错，修改成字符串首地址即可。    
### 加锁运行情况<br>
![image](https://user-images.githubusercontent.com/98074671/205804973-286d7745-669e-4e17-b1d7-b7ace3b2e927.png)
### 不加锁运行情况<br>
![image](https://user-images.githubusercontent.com/98074671/205806426-ff0090e4-eb23-4de3-b6bd-9849911ceb52.png)
***原因分析：***  加锁能保证某一子进程在写入2000个数据时，其他子进程不会同时进行写的操作，因此体现为第一张图，前面全为1，后面全为2。而不加锁时，两个子进程之间不会互斥，写的顺序取决于调度顺序和时长，因此体现出了12交叉写入的情况。  
<linux管道pipe详解>--<https://blog.csdn.net/oguro/article/details/53841949?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522167029299016800184152433%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=167029299016800184152433&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~top_positive~default-1-53841949-null-null.142^v67^js_top,201^v4^add_ask,213^v2^t3_esquery_v2&utm_term=%E7%AE%A1%E9%81%93&spm=1018.2226.3001.4187>   

## Part3
***连续分配***：为用户进程分配的必须是一个连续的内存空间。
***非连续分配***：为用户进程分配的可以是一些分散的内存空间。
***分页存储管理的思想***：把内存分为一个个相等的小分区，再按照分区大小把进程拆分成一个个小部分。
***分页存储管理分为***：实分页存储管理和虚分页存储管理
***运行出错***<br>
出现了运行错误的情况。  
***解决方法：***  
通过检查代码发现，我给queue->rear的初值设置为0，而rear是指向最后一个（实际存在的）的项，即队尾，但是初始化时该处并没有实际上赋值，导致每新增一项，rear总是指向空，这就导致了isFull函数在还剩一个空位时认为队列已满，产生错误。
### FIFO
在分配内存页面数（AP）小于进程页面数（PP）时，最先运行的AP个页面放入内存；当内存分配页面被占满时，如果又需要处理新的页面，则将原来放的内存中的AP个页中最先进入的调出（FIFO），再将新页面放入，总是淘汰最先进入内存的页面；所使用的内存页面构成一个队列，效率不高，因为它与进程实际的运行规律不相适应，如常用的全局变量、递归函数或循环体的所在页面都可能被它选为淘汰对象    
***bleady现象原因：***    
FIFO算法的置换特征与进程访问内存的动态特征是非常不一致的，即被置换的页面通常并不是进程不会访问的  
在AP=7，PP=10时  
### 运行情况<br>
![image](https://user-images.githubusercontent.com/98074671/206364521-3f5f45e2-40a3-470e-a884-ed61a782ac34.png)  
### LRU(Least-Recent-Used algorithm)
三种方法：a、计时法，b、堆栈法，c、多位寄存器法。我的代码采用了计数器实现法，新增了pagecontroltimer数组来表示时间域，并为timer作为计数器。每次引用都会递增timer，并更新对应引用项的pttimer。通过FindMintimer来寻找到最近更新时间最小的内存页面下标，通过此下标来更新相关数值。  
在AP=7，PP=10时  
### 运行情况<br>
![image](https://user-images.githubusercontent.com/98074671/206365213-aadeac75-11b4-4e30-947d-12ec9d5dcaa3.png)  
<操作系统之页式管理>-- <https://zhuanlan.zhihu.com/p/391327282>   


***附：***  ssh基本用法：https://zhuanlan.zhihu.com/p/21999778  
ssh命令详解：https://www.cnblogs.com/ftl1012/p/ssh.html  
github Readme编写方法：https://blog.csdn.net/Strive_For_Future/article/details/120956016  
线程：https://blog.csdn.net/m0_52640673/article/details/123533809
