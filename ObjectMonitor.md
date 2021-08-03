[toc]

## ObjectMonitor源码

### synchronized的使用

synchronized关键字是Java中解决并发问题的一种常用方法，也是最简单的一种方法，其作用有三个:

1. 互斥性：确保线程互斥的访问同步代码
2. 可见性：保证共享变量的修改能够及时可见
3. 有序性：有效解决重排序问题

### synchronized的底层原理

#### 对象头和内置锁(ObjectMonitor)

根据jvm的分区，对象分配在堆内存中，可以用下图标识：

![0](concurrent/images/ObjectMonitor1.png)

##### 对象头(Mark Word)

Hotspot虚拟机的对象头包括两部分信息，第一部分用于储存对象自身的运行时数据，如hashcode，gc分代年龄，锁状态标识，锁指针等，这部分数据在32bit和64bit的虚拟机中大小分别为32bit和64bit，官方称它为"Mark Word"，考虑到虚拟机的空间效率，MarkWord被设计成一个非固定的数据结构以便在极小的空间中存储尽量多的信息，它会根据对象的状态复用自己的存储空间，详细情况如下：

![0](concurrent/images/ObjectMonitor2.png)

对象头的另一部分是类型指针，即对象指向它的元数据指针，如果对象访问定位方式是句柄访问，那么该部分没有，如果是直接访问，该部分保留。句柄访问方式如下：

![0](concurrent/images/ObjectMonitor3.png)

直接访问：

![0](concurrent/images/ObjectMonitor4.png)

##### 内置锁(ObjectMonitor)

通畅所说的内对象的内置锁，是对象头MarkWord中的重量级锁指针指向的monitor对象，该对象是在HotSpot底层C++语言编写的(openjdk里面可以看)

```c++
//结构体如下
ObjectMonitor::ObjectMonitor() {  
  _header       = NULL;  
  _count       = 0;  
  _waiters      = 0,  
  _recursions   = 0;       //线程的重入次数
  _object       = NULL;  
  _owner        = NULL;    //标识拥有该monitor的线程
  _WaitSet      = NULL;    //等待线程组成的双向循环链表，_WaitSet是第一个节点
  _WaitSetLock  = 0 ;  
  _Responsible  = NULL ;  
  _succ         = NULL ;  
  _cxq          = NULL ;    //多线程竞争锁进入时的单向链表
  FreeNext      = NULL ;  
  _EntryList    = NULL ;    //_owner从该双向循环链表中唤醒线程结点，_EntryList是第一个节点
  _SpinFreq     = 0 ;  
  _SpinClock    = 0 ;  
  OwnerIsThread = 0 ;  
} 
```

ObjectMonitor队列之间的关系转换可以用下图表示：

![0](concurrent/images/ObjectMonitor5.png)

既然提到了_waitSet和_EntryList(_cxq队列后面会说)，那就看一下底层的wait和notify方法

wait方法的实现过程：

```c++
//1.调用ObjectSynchronizer::wait方法
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  /*省略 */
  //2.获得Object的monitor对象(即内置锁)
  ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  //3.调用monitor的wait方法
  monitor->wait(millis, true, THREAD);
  /*省略*/
}
  //4.在wait方法中调用addWaiter方法
  inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
  /*省略*/
  if (_WaitSet == NULL) {
    //_WaitSet为null，就初始化_waitSet
    _WaitSet = node;
    node->_prev = node;
    node->_next = node;
  } else {
    //否则就尾插
    ObjectWaiter* head = _WaitSet ;
    ObjectWaiter* tail = head->_prev;
    assert(tail->_next == head, "invariant check");
    tail->_next = node;
    head->_prev = node;
    node->_next = head;
    node->_prev = tail;
  }
}
  //5.然后在ObjectMonitor::exit释放锁，接着 thread_ParkEvent->park  也就是wait
```

总结：通过object获得内置锁(ObjectMonitor)，通过内置锁将Thread封装成ObjectWaiter对象，然后addWaiter将它插入以_waitSet为首节点的等待线程链表中去，最后释放锁。

notify方法的底层实现：

```c++
//1.调用ObjectSynchronizer::notify方法
    void ObjectSynchronizer::notify(Handle obj, TRAPS) {
    /*省略*/
    //2.调用ObjectSynchronizer::inflate方法
    ObjectSynchronizer::inflate(THREAD, obj())->notify(THREAD);
}
    //3.通过inflate方法得到ObjectMonitor对象
    ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
    /*省略*/
     if (mark->has_monitor()) {
          ObjectMonitor * inf = mark->monitor() ;
          assert (inf->header()->is_neutral(), "invariant");
          assert (inf->object() == object, "invariant") ;
          assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is inva;lid");
          return inf 
      }
    /*省略*/ 
      }
    //4.调用ObjectMonitor的notify方法
    void ObjectMonitor::notify(TRAPS) {
    /*省略*/
    //5.调用DequeueWaiter方法移出_waiterSet第一个结点
    ObjectWaiter * iterator = DequeueWaiter() ;
    //6.后面省略是将上面DequeueWaiter尾插入_EntrySet的操作
    /**省略*/
  }
```

总结：通过object获得内置锁(ObjectMonitor)，调用内置锁的notify方法，通过_waitset节点移出等待链表中的首节点，将它置于_EntrySet中去，等待获取锁。

注意：notifyAll根据policy不同可能移入_EntryList或者_cxq队列中，此处不详谈。

### synchronized的优化

在了解了synchronized重量级锁效率特别低之后，jdk自然做了一些优化，出现了偏向锁，轻量级锁，重量级锁，自旋等优化，我们应该改成monitorenter指令就获取对象重量级锁的错误认知。很显然，优化之后，锁的获取判断次序应该是偏向锁->轻量级锁->重量级锁。

#### 偏向锁

```c++
//偏向锁入口
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 //UseBiasedLocking判断是否开启偏向锁
 if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      //获取偏向锁的函数调用
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
 }
 //不能偏向，就获取轻量级锁
 slow_enter (obj, lock, THREAD) ;
}
```

BiasedLocking::revoke_and_rebias调用过程如下：

![0](concurrent/images/偏向锁1.png)

偏向锁的撤销过程如下：

![0](concurrent/images/偏向锁2.png)

#### 轻量级锁

轻量级锁获取源码：

```c++
//轻量级锁入口
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();  //获得Mark Word
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");
  //是否无锁不可偏向，标志001
  if (mark->is_neutral()) {
    //图A步骤1
    lock->set_displaced_header(mark);
    //图A步骤2
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
    // Fall through to inflate() ...
  } else if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) { //如果Mark Word指向本地栈帧，线程重入
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);//header设置为null
    return;
  }
  lock->set_displace
 
  d_header(markOopDesc::unused_mark());
  //轻量级锁膨胀，膨胀完成之后尝试获取重量级锁
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```

轻量级锁获取流程如下：

![0](concurrent/images/轻量级锁1.png)

轻量级锁撤销源码：

```c++
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");
  markOop dhw = lock->displaced_header();
  markOop mark ;
  if (dhw == NULL) {//如果header为null，说明这是线程重入的栈帧，直接返回，不用回写
     mark = object->mark() ;
     assert (!mark->is_neutral(), "invariant") ;
     if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
        assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
     }
     if (mark->has_monitor()) {
        ObjectMonitor * m = mark->monitor() ;
     }
     return ;
  }
 
  mark = object->mark() ;
  if (mark == (markOop) lock) {
     assert (dhw->is_neutral(), "invariant") ;
     //CAS将Mark Word内容写回
     if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
        TEVENT (fast_exit: release stacklock) ;
        return;
     }
  }
  //CAS操作失败，轻量级锁膨胀，为什么在撤销锁的时候会有失败的可能？
   ObjectSynchronizer::inflate(THREAD, object)->exit (THREAD) ;
}
```

轻量级锁撤销流程如下：

![0](concurrent/images/轻量级锁2.png)

轻量级锁膨胀

源码：

```c++
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
  assert (Universe::verify_in_progress() ||
          !SafepointSynchronize::is_at_safepoint(), "invariant") ;
  for (;;) { // 为后面的continue操作提供自旋
      const markOop mark = object->mark() ; //获得Mark Word结构
      assert (!mark->has_bias_pattern(), "invariant") ;
 
      //Mark Word可能有以下几种状态:
      // *  Inflated(膨胀完成)     - just return
      // *  Stack-locked(轻量级锁) - coerce it to inflated
      // *  INFLATING(膨胀中)     - busy wait for conversion to complete
      // *  Neutral(无锁)        - aggressively inflate the object.
      // *  BIASED(偏向锁)       - Illegal.  We should never see this
 
      if (mark->has_monitor()) {//判断是否是重量级锁
          ObjectMonitor * inf = mark->monitor() ;
          assert (inf->header()->is_neutral(), "invariant");
          assert (inf->object() == object, "invariant") ;
          assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
          //Mark->has_monitor()为true，说明已经是重量级锁了，膨胀过程已经完成，返回
          return inf ;
      }
      if (mark == markOopDesc::INFLATING()) { //判断是否在膨胀
         TEVENT (Inflate: spin while INFLATING) ;
         ReadStableMark(object) ;
         continue ; //如果正在膨胀，自旋等待膨胀完成
      }
 
      if (mark->has_locker()) { //如果当前是轻量级锁
          ObjectMonitor * m = omAlloc (Self) ;//返回一个对象的内置ObjectMonitor对象
          m->Recycle();
          m->_Responsible  = NULL ;
          m->OwnerIsThread = 0 ;
          m->_recursions   = 0 ;
          m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;//设置自旋获取重量级锁的次数
          //CAS操作标识Mark Word正在膨胀
          markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
          if (cmp != mark) {
             omRelease (Self, m, true) ;
             continue ;   //如果上述CAS操作失败，自旋等待膨胀完成
          }
          m->set_header(dmw) ;
          m->set_owner(mark->locker());//设置ObjectMonitor的_owner为拥有对象轻量级锁的线程，而不是当前正在inflate的线程
          m->set_object(object);
          /**
          *省略了部分代码
          **/
          return m ;
      }
  }
}
```

轻量级锁膨胀流程图：

![0](concurrent/images/轻量级锁3.png)

**为什么在撤销轻量级锁的时候会有失败的可能？**

假设thread1拥有了轻量级锁，MarkWord指向thread1栈帧，thread2请求锁 的时候，就会膨胀初始化ObjectMonitor对象，将MarkWork更新为指向ObjectMonitor的指针，那么thread1推出的时候，CAS操作会失败，因为MarkWork不再指向thread1的栈帧，这个时候thread1自旋等待infalte完毕，执行重量级锁的退出过程。

#### 重量级锁

重量级锁的获取入口：

```c++
void ATTR ObjectMonitor::enter(TRAPS) {
  Thread * const Self = THREAD ;
  void * cur ;
  cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  if (cur == NULL) {
     assert (_recursions == 0   , "invariant") ;
     assert (_owner      == Self, "invariant") ;
     return ;
  }
 
  if (cur == Self) {
     _recursions ++ ;
     return ;
  }
 
  if (Self->is_lock_owned ((address)cur)) {
    assert (_recursions == 0, "internal state error");
    _recursions = 1 ;
    // Commute owner from a thread-specific on-stack BasicLockObject address to
    // a full-fledged "Thread *".
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
  }
  /**
  *上述部分在前面已经分析过，不再累述
  **/
 
  Self->_Stalled = intptr_t(this) ;
  //TrySpin是一个自旋获取锁的操作，此处就不列出源码了
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
     Self->_Stalled = 0 ;
     return ;
  }
  /*
  *省略部分代码
  */
    for (;;) {
      EnterI (THREAD) ;
      /**
      *省略了部分代码
      **/
  }
}
```

进入Enterl(TRAPS)方法

```c++
void ATTR ObjectMonitor::EnterI (TRAPS) {
    Thread * Self = THREAD ;
    if (TryLock (Self) > 0) {
        //这下不自旋了，我就默默的TryLock一下
        return ;
    }
 
    DeferredInitialize () ;
    //此处又有自旋获取锁的操作
    if (TrySpin (Self) > 0) {
        return ;
    }
    /**
    *到此，自旋终于全失败了，要入队挂起了
    **/
    ObjectWaiter node(Self) ; //将Thread封装成ObjectWaiter结点
    Self->_ParkEvent->reset() ;
    node._prev   = (ObjectWaiter *) 0xBAD ; 
    node.TState  = ObjectWaiter::TS_CXQ ; 
    ObjectWaiter * nxt ;
    for (;;) { //循环，保证将node插入队列
        node._next = nxt = _cxq ;//将node插入到_cxq队列的首部
        //CAS修改_cxq指向node
        if (Atomic::cmpxchg_ptr (&node, &_cxq, nxt) == nxt) break ;
        if (TryLock (Self) > 0) {//我再默默的TryLock一下，真的是不想挂起呀！
            return ;
        }
    }
    if ((SyncFlags & 16) == 0 && nxt == NULL && _EntryList == NULL) {
        // Try to assume the role of responsible thread for the monitor.
        // CONSIDER:  ST vs CAS vs { if (Responsible==null) Responsible=Self }
        Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
    }
    TEVENT (Inflated enter - Contention) ;
    int nWakeups = 0 ;
    int RecheckInterval = 1 ;
 
    for (;;) {
        if (TryLock (Self) > 0) break ;//临死之前，我再TryLock下
 
        if ((SyncFlags & 2) && _Responsible == NULL) {
           Atomic::cmpxchg_ptr (Self, &_Responsible, NULL) ;
        }
        if (_Responsible == Self || (SyncFlags & 1)) {
            TEVENT (Inflated enter - park TIMED) ;
            Self->_ParkEvent->park ((jlong) RecheckInterval) ;
            RecheckInterval *= 8 ;
            if (RecheckInterval > 1000) RecheckInterval = 1000 ;
        } else {
            TEVENT (Inflated enter - park UNTIMED) ;
            Self->_ParkEvent->park() ; //终于挂起了
        }
 
        if (TryLock(Self) > 0) break ;
        /**
        *后面代码省略
        **/
    }
```

TryLock

```c++
int ObjectMonitor::TryLock (Thread * Self) {
   for (;;) {
      void * own = _owner ;
      if (own != NULL) return 0 ;//如果有线程还拥有着重量级锁，退出
      //CAS操作将_owner修改为当前线程，操作成功return>0
      if (Atomic::cmpxchg_ptr (Self, &_owner, NULL) == NULL) {
         return 1 ;
      }
      //CAS更新失败return<0
      if (true) return -1 ;
   }
}
```

重量级锁获取入口流程图：

![0](concurrent/images/重量级锁1.png)

重量级锁出口：

```c++
void ATTR ObjectMonitor::exit(TRAPS) {
   Thread * Self = THREAD ;
   if (THREAD != _owner) {
     if (THREAD->is_lock_owned((address) _owner)) {
       _owner = THREAD ;
       _recursions = 0 ;
       OwnerIsThread = 1 ;
     } else {
       TEVENT (Exit - Throw IMSX) ;
       if (false) {
          THROW(vmSymbols::java_lang_IllegalMonitorStateException());
       }
       return;
     }
   }
   if (_recursions != 0) {
     _recursions--;        // 如果_recursions次数不为0.自减
     TEVENT (Inflated exit - recursive) ;
     return ;
   }
   if ((SyncFlags & 4) == 0) {
      _Responsible = NULL ;
   }
 
   for (;;) {
      if (Knob_ExitPolicy == 0) {
         OrderAccess::release_store_ptr (&_owner, NULL) ;   // drop the lock
         OrderAccess::storeload() ;                         
         if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
            TEVENT (Inflated exit - simple egress) ;
            return ;
         }
         TEVENT (Inflated exit - complex egress) ;
         if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
            return ;
         }
         TEVENT (Exit - Reacquired) ;
      } else {
         if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
            OrderAccess::release_store_ptr (&_owner, NULL) ;  
            OrderAccess::storeload() ;
            if (_cxq == NULL || _succ != NULL) {
                TEVENT (Inflated exit - simple egress) ;
                return ;
            }
            if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
               TEVENT (Inflated exit - reacquired succeeded) ;
               return ;
            }
            TEVENT (Inflated exit - reacquired failed) ;
         } else {
            TEVENT (Inflated exit - complex egress) ;
         }
      }
      ObjectWaiter * w = NULL ;
      int QMode = Knob_QMode ;
      if (QMode == 2 && _cxq != NULL) {
          /**
          *模式2:cxq队列的优先权大于EntryList，直接从cxq队列中取出一个线程结点，准备唤醒
          **/
          w = _cxq ;
          ExitEpilog (Self, w) ;
          return ;
      }
 
      if (QMode == 3 && _cxq != NULL) {
          /**
          *模式3:将cxq队列插入到_EntryList尾部
          **/
          w = _cxq ;
          for (;;) {
             //CAS操作取出cxq队列首结点
             ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
             if (u == w) break ;
             w = u ; //更新w，自旋
          }
          ObjectWaiter * q = NULL ;
          ObjectWaiter * p ;
          for (p = w ; p != NULL ; p = p->_next) {
              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
              p->TState = ObjectWaiter::TS_ENTER ; //改变ObjectWaiter状态
              //下面两句为cxq队列反向构造一条链，即将cxq变成双向链表
              p->_prev = q ;
              q = p ;
          }
          ObjectWaiter * Tail ;
          //获得_EntryList尾结点
          for (Tail = _EntryList ; Tail != NULL && Tail->_next != NULL ; Tail = Tail->_next) ;
          if (Tail == NULL) {
              _EntryList = w ;//_EntryList为空，_EntryList=w
          } else {
              //将w插入_EntryList队列尾部
              Tail->_next = w ;
              w->_prev = Tail ;
          }
   }
 
      if (QMode == 4 && _cxq != NULL) {
         /**
         *模式四：将cxq队列插入到_EntryList头部
         **/
          w = _cxq ;
          for (;;) {
             ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
             if (u == w) break ;
             w = u ;
          }
          ObjectWaiter * q = NULL ;
          ObjectWaiter * p ;
          for (p = w ; p != NULL ; p = p->_next) {
              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
              p->TState = ObjectWaiter::TS_ENTER ;
              p->_prev = q ;
              q = p ;
          }
          if (_EntryList != NULL) {
            //q为cxq队列最后一个结点
              q->_next = _EntryList ;
              _EntryList->_prev = q ;
          }
          _EntryList = w ;
       }
 
      w = _EntryList  ;
      if (w != NULL) {
          ExitEpilog (Self, w) ;//从_EntryList中唤醒线程
          return ;
      }
      w = _cxq ;
      if (w == NULL) continue ; //如果_cxq和_EntryList队列都为空，自旋
 
      for (;;) {
          //自旋再获得cxq首结点
          ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
          if (u == w) break ;
          w = u ;
      }
      /**
      *下面执行的是：cxq不为空，_EntryList为空的情况
      **/
      if (QMode == 1) {//结合前面的代码，如果QMode == 1，_EntryList不为空，直接从_EntryList中唤醒线程
         // QMode == 1 : drain cxq to EntryList, reversing order
         // We also reverse the order of the list.
         ObjectWaiter * s = NULL ;
         ObjectWaiter * t = w ;
         ObjectWaiter * u = NULL ;
         while (t != NULL) {
             guarantee (t->TState == ObjectWaiter::TS_CXQ, "invariant") ;
             t->TState = ObjectWaiter::TS_ENTER ;
             //下面的操作是双向链表的倒置
             u = t->_next ;
             t->_prev = u ;
             t->_next = s ;
             s = t;
             t = u ;
         }
         _EntryList  = s ;//_EntryList为倒置后的cxq队列
      } else {
         // QMode == 0 or QMode == 2
         _EntryList = w ;
         ObjectWaiter * q = NULL ;
         ObjectWaiter * p ;
         for (p = w ; p != NULL ; p = p->_next) {
             guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
             p->TState = ObjectWaiter::TS_ENTER ;
             //构造成双向的
             p->_prev = q ;
             q = p ;
         }
      }
      if (_succ != NULL) continue;
      w = _EntryList  ;
      if (w != NULL) {
          ExitEpilog (Self, w) ; //从_EntryList中唤醒线程
          return ;
      }
   }
}
```

ExitEpilog用来唤醒线程：

```c++
void ObjectMonitor::ExitEpilog (Thread * Self, ObjectWaiter * Wakee) {
   assert (_owner == Self, "invariant") ;
   _succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
   ParkEvent * Trigger = Wakee->_event ;
   Wakee  = NULL ;
   OrderAccess::release_store_ptr (&_owner, NULL) ;
   OrderAccess::fence() ;                            
   if (SafepointSynchronize::do_call_back()) {
      TEVENT (unpark before SAFEPOINT) ;
   }
   DTRACE_MONITOR_PROBE(contended__exit, this, object(), Self);
   Trigger->unpark() ; //唤醒线程
   // Maintain stats and report events to JVMTI
   if (ObjectMonitor::_sync_Parks != NULL) {
      ObjectMonitor::_sync_Parks->inc() ;
   }
}
```

重量级锁出口流程图：

![0](concurrent/images/重量级锁2.png)

#### 自旋

通过对源码的分析，发现多处存在自选和tryLock操作，那么这些操作好不好，如果tryLock过少，大部分线程都会挂起，因为在拥有对象锁的线程释放锁后不能及时感知，导致用户态和和心态状态转换较多，效率低下，极限思想就是：没有自选，所有线程挂起，如果tryLock过多，存在两个问题：1.即使自选避免了挂起，但是自选的代价超过了挂起，得不偿失，那我还不如不要自旋了。2.如果自旋依然不能避免大部分挂起的话，那就是又自旋又挂起，效率太低。极限思维就是：无限自旋，拜拜浪费了CPU资源，所以在代码中每个自旋和tryLock的插入应该都是经过侧后决定的。

## 总结

**引入偏向锁的目的：**在只有单线程执行的情况下，尽量减少不必要的轻量级锁执行路径，轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只依赖一次CAS原子指令置换ThreadID，之后只要判断线程ID是否为当前线程即可，偏向锁使用了一种等到竞争出现才释放锁的机制，消除偏向锁的开销还是蛮大的。如果同步资源或代码一直都是多线程访问的，那么消除偏向锁这一步骤对你来说就是多余的，可以通过-XX:-UseBiasedLocking=false来关闭。

**引入轻量级锁的目的：**在多线程交替执行同步块的情况下，尽量避免重量级锁引起的性能消耗(用户态和和心态的切换)，但是如果多个线程在同一时刻进入临界区，会导致轻量级锁膨胀升级至重量级锁，所以轻量级锁的出现并非是要替代重量级锁。

**重入：**对于不同级别的锁都有重入策略

* 偏向锁：单线程独占，重入只检查ThreadID是否等于该线程
* 轻量级锁：重入将栈帧中lock record的header设置为null，重入退出，只用弹出栈帧，直到最后一个重入退出CAS写回数据释放锁
* 重量级锁：重入_recursions++，退出_recursions--

![0 ](concurrent/images/synchroized6.png)