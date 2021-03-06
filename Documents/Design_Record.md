## 关于MOE
- 简单来说，MOE就是“Minds Of Embedded”
- 简单来说，MOE就是一个多任务事件驱动型的嵌入式操作系统。
- 简单来说，MOE就是一个目标useful & helpful & handy的嵌入式操作系统。 
- 简单来说，MOE就是以我大女儿“萌萌（MOE）”命名的嵌入式操作系统。如果一定要一个中文名，那就是“墨意” :smile:
    
欢迎关注我的微信公众号：**墨意MOE**    
![](https://github.com/ianhom/Note-of-all/blob/master/Pic/Misc/qrcode_for_gh_a64f54357afb_258.jpg?raw=true)

## 关于调度
- 调度应该是最有意思的地方，也是因为对调度的兴趣，才促成了对MOE的开发。   
- 我们常说的裸奔，就是一个while(1)不停的进行if或switch的判断，来实现程序不同分支的执行。当分支越来越复杂，我们就可以把这个分支看做一个独立的任务，只关心任务本身想实现的功能。当任务较多的时候，就要考虑如何让CPU运行这些任务了，这就是调度的方式。   
- 调度的方式有很多，可以让每个任务依次执行（裸奔实际上就是这样处理的），也可以让有事件发生的任务投入执行（事件驱动机制），还可以增加任务优先级。不管哪种方式，调度的方式决定了系统的运行模式，甚至性能。一旦调度方式确定了，就可以围绕这个调度（scheduler）丰富系统的其他模块和接口，逐步形成一个可用的调度系统。   
- 了解TI ZigBee协议栈z-stack的朋友应该知道，它的osal调度是个有趣巧妙的事件驱动调度方式。每个任务都有静态的优先级，当执行到调度部分时，会根据优先级依次检查每个任务是否有事件发生，若无事件则检查下一个次优先级的任务；若有事件发生，则调用该任务的处理部分，同时放弃后续任务的事件检查，再重新检查最高级的任务...这样的好处就是，高优先级的任务总能得到及时的调度，缺点就是所谓的低优先级任务总是被排在最后。对于无明显任务优先级区分的应用场合下，被排在后面的任务只能躺枪等其它任务完成以后再获得执行权，甚至永远得不到响应。   
- contiki的调度方式则不同，它调度的基本原则（更为复杂的调度方式后续讨论）是一个事件FIFO---哪个任务的事件先发生，就执行哪个任务。这样对于各任务优先级差别不大的应用场合是合理的。   
- 起初MOE叫OSAL-Like，是想抽离z-stack中的osal部分。但在后来发现，个人偏好事件FIFO，加之z-stack声明了些使用限制，于是就修改了调度的方式，并更名为MOE，因为在编写的过程中，学习、练习了很多嵌入式思想，所以就起名“Minds Of Embedded”。   
- MOE使用了事件FIFO或事件队列的方式，先发生事件的任务先处理。同时也提供了插队的机制来确保一些高优先级的事件，这样能提高一定实时性。如果想更进一步提高实时性，可以考虑在任务过程中调度其他任务的方式（参考contiki或QP的Qk内核），等考虑清楚之后再添加。   
- 添加一份总结，目前的MOE的调度属于协作型调度。抢占型的调度还需努力。

## 再关于调度
- 目前MOE的调度方式是无优先级的，哪个任务有事件就执行哪个任务，后续的事件根据发生的时间先后排队。这样的方式比较适合一些没有明确优先级的应用中，不必担心所谓的高优先级会不停霸占CPU而使得低优先级任务不得以执行。
对于有实时性要求，有明确优先级的应用中，设想下列三种调度方式：
- 1.非抢占优先级调度
- 2.抢占式统一栈调度
- 3.抢占式独立栈调度


## 关于事件驱动
- 相信事件驱动应该是个很熟悉的概念，单片机系统总是对内外部所发生的事件作出相应的响应---按键按下、定时结束、通讯数据到来....这些在以往的裸奔系统中如果都用查询的方式实现，将会比较低效（至少CPU没有机会休眠），所以为了高效可以采用中断的方式，实际上每个中断都对应着事件。在事件驱动的系统中，每个任务是否得到执行，基本上是取决于是否有相关的事件发生，这样任务其实挺像中断处理程序的。但不同的是，每个任务可以响应各种不同的事件。这里的事件类似于一种软件中断，它不会像硬件中断那样，在发生的那一刻进着手准备响应的中断处理，而是在产生事件的任务（或中断）运行结束之后，程序运行至调度处理部分之后，再根据事件所属的任务进行任务调度，在任务内部，会根据不同的事件作出不同的响应。
- 或许我将之前的调度和这里的事件驱动分开描述并不合适，因为这两者关系非常紧密。正如之前所说，MOE初期是类osal的系统，后来更换了调度方式，发现事件的产生和处理也必须得相应更改。
- OSAL在编译时为每个任务申请了一个2字节的变量用于保存改任务所发生的事件，因为每个任务仅有这一个16bit的变量保存事件，为了应付同一任务有多种多个事件的情况，osal通过事件变量中的每一位来表示一个事件，即一个2字节的事件变量可以同时保存16种事件---对应位为0表示该事件未发生，对应位为1表示该事件发生。说到这要注意两个问题：第一个问题就是只能有16个事件类别，不能每个按键或SPI、uart、i2c都设置个事件，16个事件明显不够用。这时可以进行归类，比如上述事件都可以归于系统事件，而在产生系统事件的同时产生一个消息用于标记具体是哪个事件，通过这个方法就可以扩展事件的数量。第二个问题就是对于同一个任务同一个事件的多次发生，比如两个按键同时按下，产生了系统事件，同时产生两个消息，但任务的2字节事件变量只有一位来标记这两个事件。为了解决这个问题，在处理消息的时候就需要做额外工作，当搜索到一个消息的之后，要继续搜索是否还有消息，如果有就继续产生改任务的系统事件，以便该任务能有机会再次被调度并处理下一个消息。
- Contiki采用的是事件队列的方式，同一个任务的同一个事件或不同事件都可以依次记录上事件队列上，只要队列空间足够，每个事件都会得到处理。对于事件类型的数量，完全取决于记录类型的变量size，如果是一个字节就可以枚举256个事件类别。
- 正如前述，调度和事件的方式是紧密相关的，MOE采用了事件队列的方式，作出了两方面的修改：1、事件可以插队到第一个，即刚产生的事件可以在其他已产生的事件之前进行处理；2、可自动扩展事件队列长度，采用队列（一般数组形式实现）的一个缺点是队列长度固定，如果突发的事件群过多，而系统又没有来得及消化这些事件，那事件队列将累积至满，造成后续事件的丢失。而另一种数据结构可以有效解决长度的问题----链表。如果事件通过链表的方式来记录，在RAM资源充足的情况下可以不断扩展长度，且在无事件的时候释放多余的RAM，事件插队操作在链表上也更容易实现。但链表有个缺点，生产或释放节点时有一定的开销（malloc & free），同时使用链表所产生的额外RAM消耗也不可小觑，举例来说，如果一个事件产生，必要的信息为事件类别和所说任务，一个2个字节，而为了这两个字节，假设在32位的MCU上，链表节点需要额外4个字节记录下一个节点的地址，malloc节点的时候还需要head信息假设大约8个字节，在假设malloc以8字节为块分配HEAP，那记录一个事件就需要((4+2)/8+1)\*8+8 = 16个字节，有效数据占比2/16=12.5%，所以用链表也很消耗RAM。MOE为了解决这个问题，将数组循环队列和链表结合起来，例如先生成一个长度为10的队列，当队列满的时候，再动态分配一个长度为10的队列，并接到之前的队列上去，通过链表的方式将两个队列关联起来，当事件消耗到一定数量，且额外增加的队列不在需要时，在释放该队列，这样就可以解决数组队列长度受限的问题，同时解决的链表有效数据低、频繁申请&释放的开销。
- 对于事件还有一个方面需要考虑，就是事件的定义。MOE可以定义256个事件类别，系统已经定义了一些如定时器事件、消息事件、测试事件等等事件类别，但对于不同的任务可能会需要定义新的事件类别，同时这个事件类别很可能会别其他任务所使用（任务间的通讯），所以需要考虑清楚采用何种方式定义事件。这里还要提出一点，就是MOE采用的模块化编程，有一个基本原则就是:“完成、并通过测试软件模块，不得进行任何修改，除非功能有误”。更多关于模块化的思路后续再记录，这里需要考虑一个好的方法，既可以让所有模块（内核和应用）知道事件类型定义，又不需要修改模块的代码（.C和.H）。这个问题留给我慢慢考虑吧
- 学习了下QP的事件驱动机制，有如下两种方式：1、直接将事件放置到主动对象（任务）的事件队列中；2、将事件发行到框架中，然后由框架发行给订阅改事件的主动对象。回头再看MOE的机制，更像是第一种方式，但有所不同，任务设置事件时就设置了接收改事件的任务，这点和第一种很类似，但这个事件并没有设置到目标任务的队列中，而是放到了内核的事件队列中，内核再根据事件队列中的事件来调度响应的任务，这点和第二点很像。
- 如果采用优先级调度，可以为每一个任务设置事件队列，当事件发生时，放入对应的任务的事件队列中，在调度时，首先调度优先级最高的拥有非空事件队列的任务。
- 关于事件的定义，除了通用事件，例如定时器，按键等，不同的应用或任务都可能会定义自己的事件，每个事件对应自己的事件值，这里就会涉及到事件类型管理的问题，为了简化操作，避免事件值重复，可以采用枚举类型定义，同时将事件类型的定义放在配置文件中，避免内核文件被频繁修改。通用型的事件类型定义在内核文件中，避免被意外修改；而不同任务的自定义的事件类型则统一保存在config.h中或分布在各自的头文件中（要考虑下类型重复的影响）。

## 关于定时器
- 完成调度和事件部分之后，基本的core就完成了，剩下的就是围绕这个core进行重要模块的添加。其中一个很重要的模块就是软件定时器模块，因为不仅仅外部模块需要计时功能，core也可能需要利用时间信息完成一些工作。
- 软件定时器的基本原理: 记录当前系统ms时钟，设定定时ms数，然后回到任务调度之前（因为任务是在有事件发生后才会被调度的，所以对计时结束的检查是放在调度之外进行轮询的），检查当“前系统ms时钟”与“之前记录的ms时钟”的差值是否大于设定的ms数，即可知道定时器是否计时结束，多个计时器通过链表连维护，当计时次数为0时，将会把该计时器节点从链表上删除。当计时结束后，将会产生事先设定的事件，此刻对应任务就可以做响应处理。例如任务中需要周期控制LED的亮灭，则该任务可以设置一个针对自己的周期定时器，设定事件为“周期定时事件”，然后把需要周期处理的LED操作放到对改事件的处理部分中即可。
- 在回到模块化设计方向，该软件定时器几乎全靠软件实现，唯一需要的硬件支持就是“获取系统ms时钟的函数”，该函数会根据不同的硬件所有不同，为了能兼容不同的MCU和硬件时钟获取方案，采用了函数注册的方式，将外部的“获取系统ms时钟的函数”注册到core的timer中即可。定时的精度基本取决于“获取系统ms时钟的函数”的精度。

## 再关于定时器   
- 对定时器模块进行了升级改造，原有的方式问题在于，每次处理各个定时器时，需要更新所有的剩余时间，大部分操作是重复的。 
- 经过改造，所有定时器节点只在下列情况下更新剩余时间:
    - 新定时节点加入之前，大家需要统一剩余时间的起始时刻（考虑修改此条，思路：把新增的节点增加已过去的时间）；
    - 已有计时器节点重计时之前，相当于新定时节点加入计时，大家需要统一剩余时间的起始时刻；
    - 有计时节点计时结束，找出下一个即将到时间的节点，大家再次统一下时间。    
   
## 再再谈定时器
- 之前说到一次定时器优化，优化的重点就是把"每个定时器节点实时更新"修改为"最先到的时间节点结束时一起更新"。这里有个细节就是，在新定时器节点加入后，需要所有节点同步一下时间，在这里尝试另一种方案，就是新节点加入不再更新其它节点的时间，而是把新节点的时间加上已逝去的时间（关注计数溢出问题），这样可以减少对所有节点的操作。

## 再再再谈定时器
- 在写冒泡排序时面临一个问题，那就是定时事件本身是无法告知任务之前定的时间。这其实并不是bug，因为任务也不关心，但确实任务仅仅只能通过事件类型来获得定时器相关事件的区别。所以在定时器节点中，增加了一个void \*的任务参数，便于在定时器相关的事件中可以传递参数。

## 再谈消息机制
- 目前MOE的消息机制是由源任务或中断成产消息，然后把消息放到链表的末端。目标任务收到MSG时间后，会从消息链表的头端开始查看是否有目标任务为自己的消息，如果有就获取消息。
- 下一步考虑将消息的地址直接交给目标任务，目标任务在接收到MSG事件时，可以根据附带的消息地址得到消息内容，而不用历遍整个链表。为了实现该效果，需要修改事件结构体，以便携带更多信息。   
- 20161001 消息模块已经完成了改造，一个事件将包含“事件类型”，“事件目标任务编号”，“void指针用于传递参数”，而消息实体怎通过该指针直接传递给任务。


## 关于任务中的硬件操作
- MOE十分注重模块化，每个任务也在此原则下进行设计，目的是为了通过几个既有任务的组合，快速实现新功能新产品的开发。
- 简单概括下有如下几种方式在任务中操作硬件:    
  * 宏定义关联硬件操作函数。
  * 调用标准HAL函数
  * 调用注册的驱动接口
- **目前硬件操作将可能使用时通用型的HAL---HALO（Hardware Abstract Layer Open）**    

  
## 关于外部函数
- 这里的外部函数是指内核需要的，但对不同的平台有变化的函数，比如系统ms时钟函数，每个平台的实现方式不一样，但对于内核而言，它只需要改函数返回ms时钟即可。加上有模块化的要求，内核中改函数不能随意更改。这样有几种方法解决：   

1. 统一函数名称，比如在内核中调用的函数名为MOE_Core_Sys_Tm(),那外部就必须用MOE_Core_Sys_Tm()函数名实现该函数。优点是简单直接。缺点是外部的函数也可能属于其他模块，函数名不适合更改等等。
2. 动态注册的方式，定义一个static的函数指针变量，用于保存需要调用的函数的指针，这样可以动态保存函数指针，即使封装成库也没有问题，缺点是需要RAM记录函数指针，如果改指针的值意外被修改，调用时会发生意外。
3. 静态函数指针的方式，在rom中定义函数指针并在初始化的时候赋值，这样不消耗宝贵的ram，也不易出错。

## 关于任务注册
- 完成任务的实现后，需要建立任务与内核的关联，就是要让内核知道如何找到任务并在以后调度它。z-stack是通过一个静态的任务数组来实现，还有一个对应的事件数组，对应位置的事件发生，就可以调用对应位置的任务。contiki则是没有静态的任务表，而是将任务处理函数指针包含在事件信息中，处理事件时自然就调度了任务处理函数。
- MOE为了减少事件信息对RAM的开销，采用了ROM静态任务表的方式。方式同z-stack，任务处理函数罗列在任务数组中，在内核进行初始化时，将依次调用任务处理函数（这里与z-stack不同的是MOE的任务是没有初始化函数，而是将初始化作为一个事件，放到处理函数中。考虑到对事件检索的效率，在任务中初始化函数放到最后一个处理，确保在正常运行过程中其他事件能得到即使响应），让每个任务获取任务编号，以后事件发生后即可通过该任务编号找到对应的任务进行处理。
- 在z-stack中，注册任务需要填写三部分任务相关信息：1、初始化函数；2、处理函数；3函数声明。也就是说，如果增加或减少一个任务，就需要同时修改这三个地方，工作量稍多且重复，亦容易出现失误（初始化和处理不对应）。MOE目标是在Project文件中完成所有的项目配置信息，包括函数的注册，同时希望能更简洁地注册，减少工作量和失误的机会。上文中提到MOE任务没有初始化函数，故只需要填写任务的处理函数和函数申明两个内容即可。但仍不满足“一行有效信息即可”的目标。中途试过很多方法和技巧均未成功（宏定义变参比较接近但还是失败），最终在快放弃之际参考了X MACRO的方法，实现了只需要一个有效信息-函数名，就可以实现函数申明和数组罗列。使用的体验感觉不错，增加或减少注册的任务非常便捷，提高了调试效率。
- 注册任务同时还会提供另外一个信息，就是总任务数量。使用宏定义是个很方便的办法，但是在罗列所有任务的时候，其实任务总数量就已经得到了，所以这里采用sizeof（数组）/sizeof（任务）的方式得到任务总数，无需额外清点任务数量，也能避免失误。但在这里遇到另一个问题，如果使用常量宏定义，在其他需要用到任务总数量的模块中，就需要外部引用任务数组，而且这时将丢失数组长度信息，即sizeof（数组）/sizeof（任务）一直等于1。为了解决该问题，同时实现所有配置全部放到一个文件中的目标，MOE的任务数组和任务总数量是放在Project_Config.h头文件中（该头文件机会每个文件都会包含）。一般来讲，一个数组实例不应该放在头文件中，因为当这个文件被include第二次的时候，就会生产第二个同名实例，导致无法编译通过。为了解决这个问题，MOE的任务数组定义为static类型，这样即使出现同名也报错。但这样另一个问题又出现了，就是多个相同的实体浪费内存，对此将数组通过const的修饰定义在了ROM中，本来这类静态的信息没有必要放到宝贵的RAM中，这样就会浪费RAM资源。有人会说虽然不浪费RAM，但是会浪费ROM。对于这个问题，MOE利用了编译的一点特性：值相同的const类型数据将自动合并，即在ROM中只有一个实例，所有模块虽然拥有各自static的任务数组，但是他们指向的实例都是同一个，故这样，每个文件都可见任务数组的“完整”内容，自然就可以通过sizeof（数组）/sizeof（任务）来得到任务总数量。在z-stack中任务总数是通过一个const常量来保存的，所以不会遇到extern不完整的情况。只是这个任务总数经常用于循环判断，如果是保存于ROM或RAM中，不做特别优化的环境中，每次都有读取的操作。
- 其实MOE的这个做法有点冒险，毕竟这样就会暴露任务数组，即任何一个任务都可以乱调用其他任务的处理函数。但这里我还是想保留，一来我考虑其他方法让任务函数看不见这些任务数组；二来日后或许会改变调用机制，实现任务中调用其他任务，如果任务数组对任务可见，那可以更直接调用，获取一定实时性，当然考虑安全性进行下权衡，这点留着以后慢慢考虑吧。

## 关于原型线程
- 原型线程(Protothread,简称PT)是一种形式上类似线程的编程方式。
- 线程/任务间切换需要更新上下文，所以每个线程都有各自独立的栈来保存上下文信息，这些信息包括:
    * **通用寄存器值**：每个任务虽然有独立的栈，但用于计算的通用寄存是所有任务共用的，切换会发生在任何一句C语句之中的汇编语句之间，任务切换前的通用寄存器值需要保存到各自的栈中，类似中断压栈。
    * **栈顶指针SP**：用于恢复任务上下文。
    * **指令指针PC**:用于恢复代码指令。
- 任务在被抢占时，需要保存以上三个部分的内容，才能在下一次恢复执行时，在原有的上下文中继续运行。那我们再来看看PT在任务恢复的时候，是如何恢复上下文的
- 首先PT是没有独立的栈的，所以不需要恢复SP值，其次在PT应用中不会在一个临时变量无效之前Yeild退出（如有则改用static修饰），同时PT不会在某一个C语句之中退出任务，而是在某句C语句执行完之后才显性地明确地退出，所以不需要保存通用寄存器的值，这样就只剩下一个PC需要恢复。如上所说，PT应用只会在具体的C语句完成之后退出，之后肯定有与之对应的行号，那只要“回到”对应的行号，就能在原来退出的地方继续运行。所以对于PT而言，每个任务退出时需要保存的现场，只有退出时的行号这一个元素，这也就是为什么说PT只需要2个字节来保存上下文信息（2字节的行号--65535行代码）。
- 这里我们说一下PT在任务重新运行后，是如何从上一次的退出点继续运行的。首先我们需要知道，在C语言中有哪些会影响运行路径的语句----if/else, switch, goto。if的方式只有两个分支，如果需要多种分支的情况，则需要多个if/else的结构和判断语句，显示不适合断点续行的跳转。goto虽然很受争议，但确实是比较方便的方式跳转到特定的位置，但是这个有个问题，goto必须使用label，这个label仅仅是个标签，记录下goto的位置，而且这个位置在编译阶段就确定了，所以无法在代码运行时动态地调整goto的位置，这也不能满足要求。所以最终只剩下switch的方法了，switch可以通过值的方式选择分支，只要不同的分支对应不同的值，switch就能顺利跳转。现在的问题就是如何方便的设置这些“值”。因为退出点可以出现在不同代码行上，不会出现两个退出点在同一行代码中（一般也不会这样写），所以行号__LINE__这个宏就变得非常合适与定义分支的标记。在退出点记录下当前C语言代码的行号，记录到分支跳转的变量中，然后退出，当下次重新运行任务的时候，首先switch这个分支跳转的变量，就可以跳转到之前的退出点继续运行了。
- 但是代码中出现很多的__LINE__也会降低代码的可读性，所以需要对这些switch，case __LINE__的语句进行巧妙的封装，让程序员只关注应用的编写，而不用烦心这种框架的实现。
- 这里也有的两个明显的限制，就是在switch中再嵌套switch并在内层switch中yeild，将无法实现预期效果；同样也不能在子函数中yeild，因为此时退出前所记录的行号，无法通过protothread的switch中找到对应的返回点。
 
## 关于应用编写风格
- 目前应用程序支持三种应用风格：
    1. 针对事件处理的风格，类似于中断程序，当事件发生时，调用针对该事件处理的函数。该方法可以将状态机融合进应用。
    2. PT应用风格，线性自上而下的行文，多线程假象
    3. 上述两种风格的融合，PT风格做传统意义上线程的main，而事件处理则被当做传统意义上的ISR
- 对于有callback的模块（如定时器模块），可以使用callback直接调用事件的处理，不过这个就是在任务之外的处理了，即在事件产生之后和在任务处理之前的时间段内。

## 关于PT协程应用
- PT协程应用可以使用PT宏定义实现中途退出和断点返回，但在比较大的应用程序中不可避免需要调用写子函数，这时PT宏就无法在这些子函数中使用，如果必须使用的话，只能将该子函数写成宏函数，简单的子函数可以尝试inline并在编译器中开启inline相关选项，但inline不是必然会内联，除非充分了解熟悉。
- PT协程应用中可以使用while(1)，但在while中一定要有return或含有YEILD语句，确保任务能在合适的时候释放CPU，对于需要一直运行的while(1)，建议在最后增加一句PT_WAIT(1)，这样确保循环内的语句能得到尽快的执行，也能让其他任务得以运行。

## 关于事件响应风格应用程序
- 该风格的应用程序主体并不像以往的程序那样“面向过程”，程序并不描述一个时序或流程，而是将过程中对应的处理动作划归为对事件的响应。这样虽然结构零散，但思路反而更清晰。对于无真正多线程的机制下，这种风格的应用恰恰是多应用运行的真相，此问题后续讨论。
- MOE中，“初始化”也可以认为是一个事件，可以拥有对应事件处代码。但这里需要注意一点，如果采用switch来找对应的事件处理代码，不建议像习惯那样把初始化代码放在最前面，反而应该放在最后，这是考虑到switch分支的特性，将执行最不频繁的分支放到最后，有利于运行时的效率。

## 关于PT+事件响应风格
- 如上所述，PT风格更接近于入门时编写的C程序---一个自上而下的风格。而事件响应则没有行文顺序，来什么事件，执行相应的代码，更像是以往的中断。那对于“主程序+中断”的传统风格，我们现在也有了“PT+事件响应”。
- 这里举一个例子：应用中周期亮灭LED_1，这个过程中使用PT_DELAY来实现，整个主程序中周而复始。然后可以同时用一个按键，产生一个按下事件，调度任务后并不在延时处继续运行，而是跳转到事件响应处理函数，在这里可以点亮LED_2。而在延时结束后，继续回到主应用中Toggle LED_1。这一个的过程就和传统的“主程序+中断”对应上了。
- 但这里有地方需要注意：实现响应方式的事件，不能与PT应用中的某阻塞事件相同，否者将破坏PT原有的运行过程。例如PT应用中等待某事件才能继续运行，而对此事件有了对应的事件响应处理。则在进入该task的时候，就会直接跳入事件响应程序，而PT应用得不到进一步运行（目前的设计）。就好比在传统的主程序中不停判断某个中断标志是否置位，而在中断发生时，就会进入中断处理函数，处理完以后，中断标志也随即清除，再回到主程序中，就无法再看到该标志的变化。
    
## 关于硬件驱动
- 为了支持更多的平台，有更好的移植性，MOE通过HAL（Hardware Abstract Layer硬件抽象层）实现应用于硬件的屏蔽。MOE规范硬件驱动接口，不同的平台的外设只要按照定义的驱动接口提供基本功能即可，MOE将会将驱动的功能通过API提供给应用task，这样即使更换了硬件平台，只要驱动实现功能一致性有保证，那应用部分完全可以做到无修改。
- 除了通过硬件驱动注册的方式，还提供了一种硬件直接到task的参考--静态函数指针。硬件的操作函数可以在编译之前建立与task的关联，实现task到硬件的直接操作。好在对事件的定义有65535之多，可以满足该方法的正常工作。

## 关于硬件驱动接口
- 硬件驱动接口设计为如下6个：   
   1. 初始化  ：负责使能硬件，配置初始参数，传入回调函数，配置硬件参数。
   2. 控制命令：硬件控制命令，例如使能/非能硬件功能或中断。
   3. 读取操作：如果可用，可以读取硬件数据。
   4. 写入操作：如果可用，可以写入数据到硬件。
   5. 中断处理：调用上层（MOE）传入的回调函数。
- 暂且按这种粗扩的方式划分，考虑到接口size优化，将配置和初始化放到一起。    
- 配置驱动时的参数结构应包含如下信息：
   1. 回调函数接口，用于中断时执行预设的回调函数
   2. 配置参数的类型或命令
   3. 配置参数的长度
   4. 配置参数区的地址

## 关于中断操作
- 对于非RTOS的MOE而言，有硬实时性要求的操作可以通过中断来实现。在模块化设计的MOE中，硬件驱动针对某一平台是标准的，任何应用对该驱动的使用时不需要修改驱动源码的，这样就使得中断的操作也是标准的。但对于不同的应用，对在中断中的操作有不同的要求，所以在MOE在中断中保留了一个callback的接口。通过该接口，可以实现任何应用相关的操作。对于实时性要求高且短小的操作，可以在callback中直接执行；对于实时性一般的操作，则可以选择在callback中产生事件，然后由对应的任务处理。但有一点要注意的就是，中断与任务之间的共享资源冲突问题。 
- 一般MCU的硬件外设数量不止一个，比如有的STM32就有5个UART，对于这5个UART的配置控制，可以使用统一的函数，通过编号入参作区分。但对于终端处理程序，不同UART的中断就不能使用同一个函数了，毕竟在中断向量表中也会是不同的处理函数。为了不重复“编写”中断处理程序，可以使用X宏定义来批量产生不同编号的中断处理程序。   

   
## 关于堆
- 在过去的传统单片机开发中，很少使用到堆，有“全局变量”这么方便的东西，有时候连栈都不用。但引入多任务之后，用于动态分配的堆就变得十分有用。因为链表是一种物理空间上不一定连续，但逻辑上连续的数据结构，特别适合用在heap上，同时链表有“自由伸缩”的特性，很适合定时器，消息这类在运行过程中才产生的对象，所以说在MOE中，定时器和消息都是建立在heap上，对堆的依赖还是比较深的。

## 关于X宏的应用
- X宏（X macro）是利用宏定义来简单复制类似代码的一种运用。   
举个例子，在一个c源文件中，想建立一个函数指针数组，并建立一个包含上述函数名的字符串数组（可用于显示有哪些函数，某个函数的详细信息），如果使用传统的方法，需要如下步骤：   
```c
typedef void (*PF_FUNC)(int a);
//1、先声明函数：
void Function_1(int a); 
void Function_2(int a); 
void Function_3(int a);  

//2、定义函数指针数组：
const PF_FUNC cg_apfFuncTable[] = 
{
    Function_1,
    Function_2,
    Function_3
};

//3、定义函数名字符串数组
const char* cg_apcFuncName[] = 
{
    "Function_1",
    "Function_2",
    "Function_3"
};
```

- 从上面明显看出，Function_X被输入了3次，如果再1~3之间在增加一个函数，还要注意这3个地方对应位置不能出错。增加了工作量，也容易出错。   
为了解决这个问题，来看看X宏的解决方法：   
```c
//1、首先宏定义LIST
#define LIST \
             X(Function_1)\
             X(Function_2)\
             X(Function_3)

//2、函数声明
#define X(name) void name(int a);    //重定义X宏定义，指明X定义为函数声明
LIST                                 //根据上一行宏定义生成3个函数的函数声明代码
#undef X                             //取消X宏定义的内容

//3、定义函数指针数组：
const PF_FUNC cg_apfFuncTable[] = 
{
#define X(name) name,                //重定义X宏定义为函数名
LIST                                 //根据上一行宏定义生成3个函数名作为函数指针数组的成员
#undef X                             //取消X宏定义的内容
};

//4、定义函数名字符串数组
const char* cg_apcFuncName[] = 
{
#define X(name) #name,               //重定义X宏定义为函数名的字符串
LIST                                 //根据上一行宏定义生成3个函数名字符串
#undef X                             //取消X宏定义的内容
};
 ```
 
- 由此可以看出，只需要在定义LIST时书写一次Function_X，通过对X的定义转变，来自动生成对应的代码，减少编码冗余和笔误的机会。   
有了这个基础，可以很容易追加相关代码块，例如为每个函数定义一个变量计算函数调用次数。   
```c
#define X(name) unsigned int name##_CallCnt = 0;
LIST                                 //将生成代码为：Function_X_CallCnt = 0;
#undef X
```

## 关于移植
- 作为OS，可移植性也是一项重要考虑，毕竟OS会面临各种各样的硬件平台，让OS在不同的芯片上实现相同的效果，这才能体现OS的一大优点---复用现有的应用。对于RTOS而言，移植的重点存在于底层，因为RTOS会在任务执行中过程进行任务切换，这时涉及到通用寄存器（Rx）的保存和栈顶指针（SP）的切换，这些并不能都用过C语言直接操作，加之对任务的切换需要高效，所以需要对这种系统操作使用汇编编写。而不同的芯片一般有不同的汇编，所以RTOS的移植需要做一些底层汇编工作。   
- 如果OS的整体实现不依赖于任何汇编，而是使用纯c编写，那就不需要考虑汇编层面的工作了。
- 如果函数运行位置能得到保存，函数本身在切换任务之前没未使用完成的局部变量，或函数使用静态局部变量，则就无需保存pc值和通用寄存器的值。而pc和通用寄存器一般是没有地址的，一般都是通过汇编来操作的。
- MOE的应用程序有两种风格，一种是事件驱动，另一种为protothread。前者的任务代码对发生的事件进行响应，代码简短紧凑，在响应事件后即刻完成对事件的处理，即使未完成也会通过另一个事件继续运行下去;后者的protothread则通过一个两bytes的变量保存任务对我运行位置，在下一次调用是恢复现场。以上两种方式均不需要使用任何汇编语句，通过纯C即可实现，所以移植起来非常便捷。
- 一个纯粹的调度内核可以很简单，仅仅根据条件进行调度即可。而这里的条件则由外围模块进行处理，定时器模块，消息模块等都可以创造事件来驱动任务的调度。定时器模块比较特殊，因为要根据硬件时钟才能对应到真实世界的时间，所以要想让系统对真实世界的时间进行响应，必须告知一个标准的时钟，这也是移植的一个重要环节。   
- 可能最好的移植就是什么定制都不需要，仅需调用系统的初始化和运行即可。但不幸的是，如果想让系统有较好的实用性，至少需要告知系统一个标准时间，以确保系统时间相关的功能可以正常运行。对于MOE而言，一个标准的ms时钟（free running clock）是必须的。

## 关于调试选项   
- 便捷的调试方式能大幅提高开发效率，即使在MOE的内核开发中，因为少有硬件资源操作，所以MOE模块的工作情况都是通过串口printf来实现的。printf调试方式简单实用，不过任然有三个问题：
      1. 在开发过程中需要打印调试信息，而在发布时不需要进行打印（打印也没有输出接口），这是需要将代码中的打印语句删除，工作量大且容易遗落。
      2. 不同等级调试信息，包含打印的信息，所在的行号、函数名及文件名等。
      3. 不同文件有各自不同的调试等级要求。
- MOE提供了一个灵活实用的debug头文件，可以有效解决上述三个问题。DBG_PRINT可以实现“无信息”，“仅信息”，“信息+文件名+行号”，“信息+函数名+行号”及“信息+文件名+函数名+行号”，且每个文件的打印选项均可不同，详见project_config.h文件。
- 此外对于不喜欢防御性代码那种只把问题抛出而不一定能等到及时处理的风格的同学，MOE也提供了assert方法。

## 关于演化方向
- 增加RTOS实时内核，虽然MOE的方向并不是追求实时性，但是实时性依旧是我想学的一个方向，可以为MOE提供可选的实时内核，而其他的模块和接口做到复用。目前已经接触了ucos, FreeRTOS，华为的LiteOS，各有特点，需要取长补短，然后设计MOE的RT内核。
- 增加脚本语言解释器（Python，JavaScript，Lua...）一直想给MOE增加交互，使用现有的脚本语言是个不错的选择，当然上了解释器之后，对MCU的要求也会大大提高
- 增加cortex-M内核调试库的支持，能在发生错误时快速定位问题的成因及位置。
- 增加自动化的单元测试和集成测试，脚本自动执行测试用例，提高开发效率和可靠性。MOE希望提供的不仅仅是代码框架，还有可靠的测试系统。
- 以上的每个方向都不是一蹴而就的，需要不断积累学习

## 关于编译
- 目前的MOE是在IAR和Keil这样的IDE下开发的，所以对整个工程的编译，需要将两个部分的代码添加到工程中
    1. MOE项目公共部分：内核、调试组件、复用的应用模块
    2. 项目定制化的部分：MCU启动文件、驱动等    
- 后续将提供IDE工程方便移植。
- 目前MOE还未提供makefile文件以便在gcc类开源编译器进行项目编译，后面会适时完善 

## 关于配置
- MOE的很多功能，如调试选项、函数注册等都写在一个config文件中。为了方便配置，将提供PC机软件工具快速便捷地进行工程配置，实现MOE的快速移植。
- 

## 关于文件布局   

   文件夹          |   说明   
:-----        | :------------   
   [**App/**](https://github.com/ianhom/MOE/tree/master/App)             | 应用任务模块，与具体工程无关，新工程可复用该文件夹下模块或根据需求添加模块
   [**Core/**](https://github.com/ianhom/MOE/tree/master/Core)           | 内核文件，包含调度、事件驱动处理、定时器、消息处理
   [**Cpu/**](https://github.com/ianhom/MOE/tree/master/Cpu)             | MCU芯片内核、时钟、启动相关文件
   [**Driver/**](https://github.com/ianhom/MOE/tree/master/Driver)       | 驱动文件，包含MCU外设驱动、扩展设备驱动（RF模块，传感器等）
   [**Pub/**](https://github.com/ianhom/MOE/tree/master/Pub)             | 项目公共文件，包含公共头文件、宏定义、调试文件
   [**Utility/**](https://github.com/ianhom/MOE/tree/master/Utility)     | 常用功能模块，包含队列、链表、printf等
   [**project/**](https://github.com/ianhom/MOE/tree/master/project)     | 具体工程相关文件，包含工程配置文件，硬件配置配件和main文件
   [**Documents/**](https://github.com/ianhom/MOE/tree/master/Documents) | 说明性文档，包含设计说明，API说明、图片   
   [**Tools/**](https://github.com/ianhom/MOE/tree/master/Tools)         | 配置、编译、调试、分析等实用工具
   
## 关于开发板
- MOE目前已经移植了多款开发板（Nucleo系列、FRDM系列），当然，为了更好展现MOE，也在为MOE定制一款开发板。MOE开发板将采用大容量MCU，用于支持复杂的应用及脚本解释器。   
- 同时MOE支持useful的调试功能，便捷的仿真器和串口调试也是必不可少的。
- 板载仿真器，便于调试，也为脚本解释器的固件更新提供便捷     
- 对于多任务的系统，外围资源或接口也非常重要，各类输入（按键，传感器），各类输出（继电器，电机），各类通讯等等。
- 丰富的外围扩展，兼容Arduino接口，可以拥有更多通用扩展板。

## 关于内存管理
- 内存的分配大致可以分为编译器分配和动态分配（人工分配），前者在编译阶段就完成的分配，而后者则是在代码的执行阶段由代码控制分配。
- 动态分配最常见的就是malloc函数，在典型的内存结构中有一个叫堆的区域，malloc就是从堆中申请一片内存，然后由应用程序使用，用完后通过free函数释放该段内存归还到堆中。但在没有MMU的环境下，malloc有个一个弊端就是碎片问题，因为动态申请和释放的内存大小不一，很容造成已申请的内存块之间纯在很多“可用”小内存块，但却分配给一个较大申请来使用。
- 为了解决这个问题，出现了一种叫内存池（memory pool）的方式，将内存池中的内存分成大小固定的内存块，每次申请和释放都得到同样的大小，这样两个内存块之间的“碎片”的大小也就满足可申请的大小，从而不会变成不可用的区域。但这样的方式的不足也是很明显的，就是无论申请多大的内存，都只能申请到相同大小（假设块大小满足最大申请的要求），这样的话其实也是一种浪费（少量的字节内存申请还是需要一个大的内存块）。为了解决这个问题，采用多种尺寸的内存块，例如分为大中小三种，根据申请的大小，自动申请对应的块。然而这样的方式，在内存池形成的初期，就必须评估好各种块的数量，不然还是会造成浪费。
- MOE目前动态内存使用还是基于malloc，随着应用的复杂度增加，对malloc使用地频繁，需要考虑碎片化的问题，后期考虑增加memory pool的方式。

## 关于临界区
- 临界区是为了防止线程访问共享资源冲突而设置的一段代码区，在这段区域内，活动线程不会被其他线程打断，或活动线程所访问的资源不会被抢占的线程所冲突访问。MOE目前的内核是无优先级不可抢占的，所以线程之间的冲突只会发生在中断与活动任务之间，对于这样的情况，在任务操作一个全局资源的时候，可以考虑关闭中断，避免中断的访问冲突。

## 关于通讯协议栈
- 一个操作系统的流行，除了靠出色的性能及易用性，很大一定程度还看系统所拥有的外围功能---丰富的外设驱动、通讯协议栈、shell等等。
- 通讯协议栈是非常大的亮点，以contiki为例，众多Zigbee和6LoWPAN玩家为了了解及使用6LoWPAN而多contiki操作系统产生了浓厚的兴趣（也使得protothread的方式得到普及）。
- 所以对于MOE，一个前沿或可靠通讯协议栈是有必要的。对于我开启了LEON（Lite Engine Of Network）项目（以我儿子命名），LEON既可以独立使用，与MOE更是天然合拍（姐弟同心，其利断金）。
- LEON将是一个通讯协议栈集合（TCP/IP, 6LoWPAN, LoRaWan, ZigBee, BLE...）。LEON刚刚建立，还没有实质内容，但随着LEON的成长，MOE也将成熟。

## 关于实用性
- 虽然MOE初期是为了学习而开启的项目，但庆幸的是这并没有影响她的独立成长，并且渐渐拥有的可以实战的实用性。目前阶段我不希望MOE只是个简单的内核和几个小模块的扩展，我更乐意主动去完成在各个平台上的实现，发现不足，不统一的地方，进而有机会使MOE更完善，并会开发实用的上层应用，在实践中验证设计。
- 因个人爱好，收集了很多不同内核的多种开发板，这使得我有机会在不同的平台上探索MOE的移植性和实用性。如果顺利的话，经历过各种平台的MOE将会很轻松应用到新的项目中，用户只需要关心实际应用功能即可，上手即用。

## 关于写调度/OS
- 相信与MOE类似的OS应该有很多了，主流的OS亦是不少，那为何还要重造“轮子”呢？
- 重复造轮子不完全是一件浪费精力的事，技能和思路需要锻炼，即使所锻炼的内容不会使用到。一个简单的例子：去健身房挥霍体力也不是为了更好做体力活，而是为了有个更健康的身体。重复造轮子也是为了“健康”。
- 对我而言，我很庆幸自己有机会写了MOE。在编写的过程不断强化嵌入式思想，正如MOE的全称含义“Minds Of Embedded”。个人能力和自信都在默默增加，编写MOE所带来的益处远远超过了MOE本身的实用性。同时，自制调度/OS的小伙伴也越来越多，能有更多的人一起交流，是件很开心的事，学习他山之石，也是受益匪浅。
- 还有一点感觉就是积少成多。MOE并没有花很多的时间编写，只是在平日空闲在GitHub上少许添加及改善，日积月累也就有了现在的模样。这样的方式还有另一个好处，就是有更多的时间思考。思考快于行动总能减少“欠考虑”的输出。

## 关于同一任务多实例
- 在RTOS中，任务是可以被创建和删除的，这样就会存在同一个任务被多次创建的可能，这种情况就是同一任务被多次实例化，每个任务都拥有自己的堆栈。但如果任务中使用了全局/静态变量，则所有任务实例都将共享该变量而无法与其它实例相互独立。MOE中因为任务没有独立的堆栈，所以单个任务无法多实例化，但MOE通过参数数组的方式来解决。比如一个任务是继电器的状态机，控制一路继电器，如果一个项目中有多路继电器，那就需要多个这样的任务。如果这个任务中的参数通过一个数组来组织，那只需要对该任务提供一个“编号”，即可知道控制哪一路的继电器，从而实现多实例化。
- 以继电器驱动软件模块举例，设计如下的数据结构来表征单个继电器的属性，然后通过数组的方式创建多个实例：
```c
typedef struct _T_RELAY
{
    uint8 u8Id;        /* ID for relays        */
    uint8 u8State;     /* Current state of SM  */
    uint8 u8Contact;   /* Open or close status */
}T_RELAY;

#deinfe MAX_RELAY_CHANNEL_NUM        (5)

T_RELAY tRelay[MAX_RELAY_CHANNEL_NUM] = {0};  /* Create 5 relay objects */
```

## 关于MOE与RTOS的区别与共同点
- 虽然MOE支持多任务，但并不是一个传统的RTOS系统，最大的区别就是MOE不支持抢占式的任务调度。不支持抢占的一个明显缺点就是，如果一个任务始终不放弃执行权，那系统将被“卡死”。解决这个问题的方法就是在应用中，显性地使用状态机的方式对应用“切片”保证任务不会持久独占CPU,或使用既有框架（Protothread）的方式隐性地放弃执行权，让其他任务得以运行。虽然这两种方法没有RTOS中通过时间片强制调度那样方便可靠，但在嵌入式软件中，把任务写成独占CPU的方式本身就不是值得推荐的事。如果有一个唯一最高优先级的任务，并在任务中做了一个while(1)操作，那一样将会“卡死”系统，这里同样需要开发人员花精力在编写应用的过程中关注CPU的共享问题。   
- 在RTOS中，任务一般有“运行态”、“就绪态”和“阻塞态”几个状态，在MOE中也有对应的表现形式，当MOE任务正在运行时，MOE任务处于“运行态”；当对应到某个MOE任务的事件处在事件队列时，该MOE任务处于“就绪态”；如果事件队列中没有事件指向某个MOE任务，那该MOE任务可以认为是“阻塞态”；此外RTOS中会有删除任务或叫“退出态”，这里删除任务不是把任务的代码从ROM中删除，而是删除任务的实例---释放为任务分配的资源。而对于MOE中的任务仅占两个字节的上下文信息，所以不需要释放。   
- MOE中的任务件通讯机制有事件和消息，这点和RTOS类似。关于这点，事件和消息模块会设计成兼容目前的协作型内核和后续实现的抢占型内核。至于RTOS中的互斥量和信号量，因为协作型任务之间不涉及共享资源的访问冲突，所以暂时使用不到。   
 

## 关于模块化
- 之前已经提及多次，MOE的设计遵循模块化的思路。任何的模块都尽可能地相互独立，留出明确、充分的接口。在MOE中对模块的理解姑且可以认为是一个.c和.h的组合。
- 在划分模块化之前，先对整体进行层的划分：应用层、系统层、HAL和底层实现。独立的功能作为独立的模块工作在对应的层级上。比如系统层中拥有调度模块，定时器模块和消息机制模块，模块和模块之间有明确的接口，只要实现接口，模块可以任意修改，比如更换其他的调度模块实现具有优先级的调度策略。   
- 对应用的部分，也是遵循模块化思路。一个独立的功能，通过一个task来实现，同样留出明确的接口，便于其它task的互操作。关于task之间有一个非常重要的信息需要相互了解，这就是task本身的识别信息---task ID或task 地址等。例如在MOE中消息传递时就需要知道目标task的ID，以便目标task关心并接收该消息。
- 对于应用的部分，还存在相互调用的关系，应用层中也分层的关系，因此每个task可能用到其它的模块，此处也需要做好接口定义，减少耦合。
- 驱动也采用模块化的设计，区别之处在于驱动模块到系统层使用HAL接口。应用层使用系统层提供的底层接口。

## 关于脚本解释器
- 嵌入式软件工程师使用的主流语言还是编译型的C语言，但随着MCU资源越来越丰富，高级的脚本语言也可以在MCU上直接编程应用
- 设想之一，为脚本解释器专门写一个task，可以选择周期调用（适合脚本模式），或中断触发调用（适合交互模式）。
- 在脚本模式中需要实现一些底层操作函数的支持，比如python解释器中作为一个模块来调用，直接操作IO口的电平变化。同时对时间的管理，如delay可以直接用MOE来做延时，延时结束后再调用脚本解释器（仅限单线程）。同时需要建立简单的文件系统，用于保存脚本文件。
- 在交互模式下，就在上述的脚本模式基础之上，提供交互的接口，将录入的字符保存在临时脚本文件中，然后在语句结束后进行处理。对于if、while等需要更多指令的语句，则在换行后区别对待。
- 对于很多脚本语言，也是支持多线程的。如果解释器也要支持多线程，那底层的OS必须也要支持抢占式的调度内核，并且需要OS可以在运行时自由创建任务线程。

## 关于可抢占内核
- 目前MOE的特点是小巧，她是非抢占内核。之前也提到过，抢占式内核也是我学习的方向之一，随后将对MOE增加抢占式内核。
- 目前已经接触过的RTOS有FreeRTOS、Trochili和LiteOS，原理基本相同，但各模块的具体策略各有特色。MOE喜欢在完全掌握各家系统的特点之后，结合MOE的喜好，发展自己风格的RT内核。
- 增加可抢占内核后，原有的应用可以不需要修改，原应用中的与非抢占内核相关的部分（各种形式的return），计划通过宏等方式排除差异。争取做到应用的通用。对于系统中内核以外的模块（定时器，消息等）也需要进行调整，来兼容适应非抢占及抢占式的内核。
- 为了和现有的非抢占内核有所区分，将对MOE的命名有所细化，根据对资源需求的大小来区分：    

  内核类型 | 命名 | 简称 | 说明    
  --------|-----|------|----------    
  非抢占式内核 | MOE-bit  | 小b | 适用于资源受限的平台，低功耗应用    
  抢占式内核   | MOE-BYTE | 大B | 适用于资源充足，注重性能的应用    

- MOE-bit即使目前的协作型OS
- MOE-BYTE将会实现主流RTOS的特性，并在此基础上尝试新的思路，如提供python作为交互环境，便于调试和使用。

## 关于OS在物联网中的应用
- 首先，os的最初的作用是提供资源，如果能提供线程资源那就是os，如果提供的事部分服务，那更像是框架，如果提供的通讯能力，那更像协议栈，总之，这一切都是为了给的简单应用赋能，实现更完整的效果。
