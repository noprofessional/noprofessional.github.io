---
layout: post
title:  "chrome webrtc中常用事件驱动的slot机制"
date:   2020-12-24 14:23:48 +0800
categories: webrtc
---

## slot常用格式

```cpp
class A:public sigslot::has_slots<>{
    sigslot::signal<T1, T2, ...> SignalSomeThing;

    A(){
        SignalSomeThing.connect(this, &A::HandleFunc);
    }

    void trigger(){
        SignalSomeThing(t1, t2, ...);
    }

    void HandleFunc(T1 t1, T2 t2, ...){
        //do something
    }
}
```

## 优势

可以看到，这样做就能将确保处理变量的变化，而不会错过去，避免显示调用时，会出现忘记调用的情况。

另一个好处就是，处理函数与值的变化是异步的，就可以实现不同线程的的处理，以提高效率