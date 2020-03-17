---
title: GCD捕获self是否会造成内存泄漏？
date: 2020-03-13 10:25:39
tags: 并发
---

### 背景

关于 GCD 的 block 捕获 self 是否造成循环引用的问题，网上是争论不休，在 iOS 的面试中更是频繁出现。我们从 YYKit 里面的一个 [Issue](https://github.com/ibireme/YYKit/issues/41) 出发，来探索一下 GCD 跟 self 之间是否会造成循环引用的问题。

该 Issue 起源于 YYKit 中的一段代码：
```objectivec
- (void)_trimInBackground {
    __weak typeof(self) _self = self;
    dispatch_async(_queue, ^{
        __strong typeof(_self) self = _self;
        /* 此处省略一万字 **/
    });
}
```

可以看到，YY 大神在 GCD 中，为了避免循环引用，使用了 strong-weak dance，但是网友在该 Issue 中提出，苹果的 dispatch_async 函数在 block 任务执行完成后会将该 block 进行 Block_release，并不会造成循环引用，此处用 strong-weak dance 反而可能造成 block 执行前 self 就已经被释放。

而 YY 大神的观点则认为，由于self 持有一个 _queue 变量，而 _queue 会持有该 block，此时在 block 内直接捕获 self 则会造成循环引用。（self->_queue->block->self）

**然而这样真的会造成内存泄漏吗？**

### 探索
我们创建一个简单的 Demo，代码如下：
```objectivec
@interface ViewController2 ()

@property (nonatomic, strong) dispatch_queue_t queue;

@end


@implementation ViewController2

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.queue = dispatch_queue_create("com.mayc.concurrent", DISPATCH_QUEUE_CONCURRENT);
    dispatch_async(self.queue, ^{
        [self test];
    });
}

- (void)test {
    NSLog(@"test");
}

- (void)dealloc {
    NSLog(@"dealloc");
}

@end
```

在 Demo 里，ViewController2 持有一个 queue 变量，dispatch_async 的 block 中捕获了 self。我们打开一个 ViewController2 的页面，然后关掉；如果 dispatch_async 强捕获 self 会造成内存泄漏，那么 ViewController2 的 dealloc 方法必然是不会执行的。 执行结果如下：

```objectivec
2020-03-11 15:36:35.352789+0800 MCDemo[83661:22062265] test
2020-03-11 15:36:36.922477+0800 MCDemo[83661:22062108] dealloc
```

可以看到 ViewController2 被正常释放了，也就是说**并不会造成内存泄漏**。

### 源码分析
源码面前无秘密，我们看一下 dispatch_async 的源码：

```objectivec
void dispatch_async(dispatch_queue_t dq, dispatch_block_t work) {
    dispatch_continuation_t dc = _dispatch_continuation_alloc();
    uintptr_t dc_flags = DC_FLAG_CONSUME;
    dispatch_qos_t qos;
    // 将work（也就是我们传进来的任务 block）封装成 dispatch_continuation_t
    qos = _dispatch_continuation_init(dc, dq, work, 0, dc_flags);
    _dispatch_continuation_async(dq, dc, qos, dc->dc_flags);
}
```

可以看到，dispatch_async 传入的 block 最终会与其他参数封装成 dispatch_continuation_t，我们重点看一下这块封装的代码（以下代码有所精简）：

```objectivec
static inline dispatch_qos_t
_dispatch_continuation_init(dispatch_continuation_t dc,
        dispatch_queue_class_t dqu, dispatch_block_t work,
        dispatch_block_flags_t flags, uintptr_t dc_flags)
{
    // 拷贝block
    void *ctxt = _dispatch_Block_copy(work);

    dc_flags |= DC_FLAG_BLOCK | DC_FLAG_ALLOCATED;

    dispatch_function_t func = _dispatch_Block_invoke(work);
    if (dc_flags & DC_FLAG_CONSUME) {
        // 顾名思义，执行block，然后释放。
        func = _dispatch_call_block_and_release;
    }
    return _dispatch_continuation_init_f(dc, dqu, ctxt, func, flags, dc_flags);
}

// _dispatch_call_block_and_release的源码如下
void _dispatch_call_block_and_release(void *block)
{
    void (^b)(void) = block;
    b();    // 执行
    Block_release(b);    // 释放
}
```

可以看到正如 Apple 文档所说，dispatch_async 会在 block 执行完成后将其释放。因此 _self->_queue->block->self_ 这个循环引用只是暂时的（block 执行完成后被释放，打断了循环引用）。

### 刨根问底
#### dispatch_sync

既然 dispatch_async 的 block 捕获 self 不会造成循环引用，那么换成 dispatch_sync 会怎么样呢？

```objectivec
self.queue = dispatch_queue_create("com.mayc.concurrent", DISPATCH_QUEUE_CONCURRENT);
 dispatch_sync(self.queue, ^{
     [self test];
 });
```

其实 dispatch_sync 也不会有问题。我们把刚刚 Demo 中的 dispatch_async 换成 dispatch_sync ，可以看到也未造成内存泄漏。

```objectivec
2020-03-11 17:05:18.840834+0800 MCDemo[5437:69508] test
2020-03-11 17:05:20.419588+0800 MCDemo[5437:68626] dealloc
```

不过 dispatch_sync 不会造成 _self->_queue->block->self_ 循环引用的原因跟 dispatch_async 有所不同，不是因为执行完成后被 release，我们看一下官方关于 dispatch_sync 的文档有段说明：

> _Unlike with `dispatch_async`, no retain is performed on the target queue. Because calls to this function are synchronous, it "borrows" the reference of the caller. Moreover, no `Block_copy` is performed on the block._


大致意思是说，queue 不会对 block 进行持有，也不会进行 Block_copy 操作。既然 queue -> block 这一层引用不存在，自然也**不会造成循环引用**。

#### dispatch_after 等其他 GCD api
我们在 dispatch_after、dispatch_group_async 的官方文档里面也看可以到和 dispatch_async 类似的话：

> _This function performs a `Block_copy` and `Block_release` on behalf of the caller. _


可以看到这些 GCD api 的方法也都做了 release 处理，因此其他的这些也不会因为捕获 self 而造成循环引用。
### 拓展
既然就算 self 持有 queue 也不会造成 GCD 的循环引用，那如果是 self 直接持有 GCD 的 block 呢？
```objectivec
dispatch_queue_t queue = dispatch_queue_create("com.mayc.concurrent", DISPATCH_QUEUE_CONCURRENT);
self.block = ^{ 
    [self test]; 
};
dispatch_async(queue, self.block);
```

emm...如果非要这样的话，肯定是会内存泄漏的....这是因为 block 被 self 直接持有，同时在 gcd 中进行了一次 Block_copy 操作，引用计数器为 2。block 任务执行完成后进行 Block_release，此时引用计数器为1 ，这种情况下 block 不会被清理。

