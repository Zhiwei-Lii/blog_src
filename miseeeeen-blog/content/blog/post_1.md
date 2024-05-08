---
title: "Android处理Move事件时为何要等待vsync"
date: 2020-02-10T13:50:36+08:00
draft: false
---

根据https://www.jianshu.com/p/c2e26c6d4ac1
Android处理down的时候是直接处理的, 但是处理move的时候需要等待vsync.
但是为什么要等待vsync呢? 这个等待时间能不能去掉, 以优化latency?
我的理解是: 不能. Android这么设计是有原因的.

前提假设:
```
1. 一个batch中的move间隔很短. (<16ms)
2. 下一帧的画面更新只能根据一个move事件来update. 
(因为假设app在一帧以内收到了好几个move事件, 它不可能在下一帧同时响应这几个move事件. 一次画面更新只能对应一个事件)
```

所以这两个前提推出一个结论就是:
```
1. 一个batch中的move事件需要进行合并, 合并成一个或两个Motionevent. 
```
那么以什么标准, 怎么样去合并move呢? 
我只能想到以vsync来作为标准. 因为很可能会出现下面这种情况. 一个batch的move跨越了一个vsync. 所以这个batch的move应该根据vsync时间来划分成两个event.
![](/images/post_1/16134950-b63098f758fa83be.png)
所以move的合并过程会依赖于vsync的时间戳. 所以只能等vsync到了, 才能做这个事情.

相关代码: (合并多个move为一个MotionEvent)
```
661 status_t InputConsumer::consumeSamples(InputEventFactoryInterface* factory,
662        Batch& batch, size_t count, uint32_t* outSeq, InputEvent** outEvent, int32_t* displayId) {
663    MotionEvent* motionEvent = factory->createMotionEvent();
664    if (! motionEvent) return NO_MEMORY;
665
666    uint32_t chain = 0;
667    for (size_t i = 0; i < count; i++) { // count指的是batch中第一个晚于vsync的move的index
668        InputMessage& msg = batch.samples.editItemAt(i);
669        updateTouchState(msg);
670        if (i) {
671            SeqChain seqChain;
672            seqChain.seq = msg.body.motion.seq;
673            seqChain.chain = chain;
674            mSeqChains.push(seqChain);
675            addSample(motionEvent, &msg);
676        } else {
677            *displayId = msg.body.motion.displayId;
678            initializeMotionEvent(motionEvent, &msg);
679        }
680        chain = msg.body.motion.seq;
681    }
682    batch.samples.removeItemsAt(0, count);
683
684    *outSeq = chain;
685    *outEvent = motionEvent;
686    return OK;
687 }
```

