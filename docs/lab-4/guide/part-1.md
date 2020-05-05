## 线程和进程

### 基本概念

从**源代码**经过编译器一系列处理（编译、链接、优化等）得到的可执行文件，我们称为**程序**。而简单地说，**进程（Process）**就是使用正在运行并使用资源的程序，与放在磁盘中一动不动的程序不同：首先，进程得到了操作系统的**资源**支持：程序的代码、数据段被加载到**内存**中，程序所需的虚拟内存空间被真正构建出来。同时操作系统还给进程分配了程序所要求的各种**其他资源**，如我们上面几个章节中提到过的页表、文件的资源。

然而如果仅此而已，进程还尚未体现出其“**正在运行**”的特性。而正在运行意味着 **CPU** 要去执行程序代码段中的代码，为了能够进行函数调用，我们还需要**运行栈（Stack）**。

出于种种目的，我们通常将“正在运行”的特性从进程中剥离出来，这样的一个借助 CPU 和栈的执行流，我们称之为**线程 (Thread)** 。一个进程可以有多个线程，也可以如传统进程一样只有一个线程。

这样，进程虽然仍是代表一个正在运行的程序，但是其主要功能是作为**资源的分配单位**，管理页表、文件、网络等资源。而一个进程的多个线程则共享这些资源，专注于执行，从而作为**执行的调度单位**。举一个例子，为了分配给进程一段内存，我们把一整个页表交给进程，而出于某些目的（比如为了加速需要两个线程放在两个 CPU 的核上），我们需要线程的概念来进一步细化执行的方式，这时进程内部的全部这些线程看到的就是同样的页表，看到的也是相同的地址。但是需要注意的是，这些线程为了可以独立运行，有自己的栈（会放在相同地址空间的不同位置），CPU 也会以它们这些线程为一个基本调度单位。

### 线程的表示

想想一个线程何以区别于其他线程。由于线程是负责“执行”，因此我们要通过线程当前的执行状态（也称线程上下文）来描述线程的当前执行情况（也称执行现场），其中包括：CPU 各个寄存器的状态（如程序计数器 `pc` 、栈指针 `sp` 和其他用于计算等等的通用寄存器）；线程自己的运行栈的内容。

而思考一下，CPU 的各个寄存器的状态在哪里出现过呢？记不记得，当我们实现中断的时候，有一个结构叫做 `TrapFrame`，而恰好的是，其中的结构正好是我们需要表示线程的 CPU 状态的结构，这是因为「处理中断再返回」这个过程本身已经和「切换线程」这个概念有些神似了。

另外，我们也同时将实现进程，这里也会同时记录下每个线程属于哪个进程：

{% label %}os/src/process/thread.rs{% endlabel %}
```rust
/// 线程的信息
pub struct Thread {
    /// 线程的栈
    pub stack: Stack,
    /// 线程执行上下文
    ///
    /// 当且仅当线程被暂停执行时，`trap_frame` 为 `Some`
    pub trap_frame: Mutex<Option<TrapFrame>>,
    /// 所属的进程
    pub process: Arc<RwLock<Process>>,
}
```

下面，我们依次来实现 `Stack` 和 `Process`，以及对应的 `Thread` 中的函数。

### 运行栈 Stack

为了简化操作系统的逻辑，我们在使用 `Stack` 的时候会先分配一段比较大的空间（其实就是一个内存段 `Segment`），同时根据是用户线程用的栈还是内核线程用的栈来写入对应的权限。这一段比较简单：

{% label %}os/src/process/stack.rs{% endlabel %}
```rust
/// 栈是一片内存区域，其空间分配在 `Mapping` 中完成
pub struct Stack {
    range: Range<VirtualAddress>,
    is_user: bool,
}

impl Stack {
    pub fn new(range: Range<VirtualAddress>, is_user: bool) -> Self {
        Self { range, is_user }
    }

    /// 生成对应的 [`Segment`]，其权限为 rw-
    pub fn get_segment(&self) -> Segment {
        Segment {
            map_type: MapType::Framed,
            page_range: Range::from(
                VirtualPageNumber::floor(self.range.start)..VirtualPageNumber::ceil(self.range.end),
            ),
            flags: if self.is_user {
                Flags::READABLE | Flags::WRITABLE | Flags::USER
            } else {
                Flags::READABLE | Flags::WRITABLE
            },
        }
    }

    /// 返回栈顶地址
    pub fn top(&self) -> VirtualAddress {
        self.range.end
    }
}
```

但是，仅仅一个线程一个运行栈就够了吗？答案是否定的，如果线程在运行时发生了栈溢出，访问到了外面的地址产生了异常怎么办呢？这个时候，中断的处理也是需要栈的，但是现在原来的栈根本用不了了，而且更进一步的是，如果是用户线程出现了中断或异常，也不可能把内核态的东西放在用户栈上执行。所以，我们需要一个内核栈，同时需要注意到并不需要每个用户线程都来一个内核栈，我们只需要全部用户线程共用一个内核栈，在发生中断的时候用它来进行处理就好了。而如何告诉 CPU 在中断时换成内核栈呢？这时候就有了 `sscratch` 寄存器，在我们的设计中，当用户态中断时，`sscratch` 的值会代替 `sp`；而当内核态中断时，`sscratch` 的值为 0，不会代替 `sp`，这也意味着内核线程一旦发生了致命的错误，将无法通过中断异常的机制来恢复。

在下面的代码中，我们写了大量的注释，其中部分内容涉及了后面的设计，你可以结合后文再过来理解这个问题，这一个章节因为 RISC-V 的设计本身就比较复杂，需要前后反复联系多看几次。

{% label %}os/src/process/stack.rs{% endlabel %}
```rust
/// 内核栈
///
/// 用户态的线程出现中断时，因为用户栈无法保证可用性，中断处理流程必须在内核栈上进行。
/// 所以我们创建一个公用的内核栈，即当发生中断时，会将 TrapFrame 写到内核栈顶。
///
/// ### 用户线程和内核线程的区别
/// 注意到，在修改后的 `interrupt.asm` 中，添加了一些关于 `sscratch` 的判断。
/// - `sscratch` 存储什么？
///   对于用户线程，`sscratch` 的值为内核栈地址；而对于内核线程，`sscratch` 的值为 0
/// - 为什么要用 `sscratch` 存储内核栈地址？
///   为了保证中断处理流程有可用的栈，用户态发生中断时会将 `sscratch` 的值替换 `sp`。
/// - `sscratch` 是在哪里被保存的？
///   调用 [`Thread::run()`] 时，将内核栈的地址写到了 `sp`，
///   然后在 `__restore` 的流程中被存放至 `sscratch`。之所以没有直接写入，
///   是为了和正常中断恢复的流程相兼容
/// - 内核线程的 `sscratch` 为 0，那么如何找到内核栈？
///   内核线程发生中断时，检测到 `sscratch` 为 0，会直接使用当前线程的栈 `sp` 进行中断处理。
///   也就是说我们编写的内核线程如果出现 bug 会导致整个操作系统崩盘
/// - 为什么内核线程要这么做？
///   用户线程发生中断时就会进入内核态，而内核态可能发生中断的嵌套。此时，
///   内核栈已经在中断处理流程中被使用，所以应当继续使用 `sp` 作为栈顶地址
///
/// ### 用户线程 [`TrapFrame`] 的存放
/// > 1. 线程初始化时，一个 `TrapFrame` 放置在内核栈顶，`sp` 指向 `TrapFrame` 的位置
/// >   （即栈顶 - `size_of::<TrapFrame>()`）
/// > 2. 切换到线程，执行 `__restore` 时，将 `TrapFrame` 的数据恢复到寄存器中后，
/// >   会将 `TrapFrame` 出栈（即 `sp += size_of::<TrapFrame>()`），
/// >   然后保存 `sp` 至 `sscratch`（此时 `sscratch` 即为内核栈顶）
/// > 3. 发生中断时，将 `sscratch` 和 `sp` 互换，入栈一个 `TrapFrame` 并保存数据
///
/// 容易发现，用户线程的 `TrapFrame` 一定保存在内核栈顶。因此，当线程需要运行时，
/// 从 [`Thread`] 中取出 `TrapFrame` 然后置于内核栈顶即可
///
/// ### 内核线程 [`TrapFrame`] 的存放
/// > 1. 线程初始化时，一个 `TrapFrame` 放置在内核栈顶，`sp` 指向 `TrapFrame` 的位置
/// >   （即栈顶 - `size_of::<TrapFrame>()`）
/// > 2. 切换到线程，执行 `__restore` 时，将 `TrapFrame` 的数据恢复到寄存器中后，
/// >   内核栈便不再被内核线程所使用
/// > 3. 发生中断时，直接在 `sp` 上入栈一个 `TrapFrame`
/// > 4. 从中断恢复时，内核线程已经从 `TrapFrame` 中恢复了 `sp`，相当于自动释放了 `TrapFrame`
/// >   和中断处理流程所涉及的栈空间
#[repr(align(16))]
#[repr(C)]
pub struct KernelStack([u8; KERNEL_STACK_SIZE]);

lazy_static! {
    /// 公用的内核栈
    pub static ref KERNEL_STACK: KernelStack = KernelStack([0; STACK_SIZE]);
}

impl KernelStack {
    /// 在栈顶加入 TrapFrame 并且返回 sp
    pub fn push_trap_frame(&self, trap_frame: TrapFrame) -> usize {
        let push_address = &self as *const _ as usize + STACK_SIZE - size_of::<TrapFrame>();
        unsafe {
            *(push_address as *mut TrapFrame) = trap_frame;
        }
        push_address
    }
}
```

另外，上面提到的 `STACK_SIZE` 和 `KERNEL_STACK_SIZE` 都是 512 KB：

{% label %}os/src/process/config.rs{% endlabel %}
```rust
/// 每个线程的运行栈大小 512 KB
pub const STACK_SIZE: usize = 0x80000;

/// 共用的内核栈大小 512 KB
pub const KERNEL_STACK_SIZE: usize = 0x80000;
```

### 进程
有了上面大致的线程的概念，现在我们来画进程的饼，所谓的进程无非就是一些线程的集合，同时还有一个公用的内存空间（相同的页表和映射）。同时，我们也会进一步区分用户态进程和内核态进程的概念：

{% label %}os/src/process/process.rs{% endlabel %}
```rust
/// 进程的信息
pub struct Process {
    /// 是否属于用户态
    pub is_user: bool,
    /// 进程中的线程公用页表 / 内存映射
    pub memory_set: MemorySet,
    /// 所有线程
    pub threads: Vec<Arc<Thread>>, // 目前没用到
}

impl Process {
    /// 创建一个内核进程
    pub fn new_kernel() -> MemoryResult<Arc<RwLock<Self>>> {
        Ok(Arc::new(RwLock::new(Self {
            is_user: false,
            memory_set: MemorySet::new_kernel()?,
            threads: vec![],
        })))
    }

    /// 添加一个线程
    pub fn push_thread(&mut self, thread: Arc<Thread>) {
        self.threads.push(thread);
    }
}
```

有了上面比较上层的设计，下面我们将介绍本节的重头戏：线程切换。