# Condition.h

## Condition

Condition封装了一个pthread_cond_t和一个MutexLock&，提供的接口有：
- wait()：利用UnassignGuard来解决pthread_cond_wait会释放锁的问题
- notify()：对pthread_cond_signal的封装
- notifyAll()：对pthread_cond_broadcast的封装
- waitForSeconds(double seconds)：对pthread_cond_timedwait的封装，等待指定的秒数，返回是否超时，但结合了上面的UnassignGuard来解决pthread_cond_timedwait会释放锁的问题

```cpp
class Condition : noncopyable
{
 public:
  explicit Condition(MutexLock& mutex)
    : mutex_(mutex)
  {
    MCHECK(pthread_cond_init(&pcond_, NULL));
  }

  ~Condition()
  {
    MCHECK(pthread_cond_destroy(&pcond_));
  }

  void wait()
  {
    MutexLock::UnassignGuard ug(mutex_);
    MCHECK(pthread_cond_wait(&pcond_, mutex_.getPthreadMutex()));
  }

  // returns true if time out, false otherwise.
  bool waitForSeconds(double seconds);

  void notify()
  {
    MCHECK(pthread_cond_signal(&pcond_));
  }

  void notifyAll()
  {
    MCHECK(pthread_cond_broadcast(&pcond_));
  }

 private:
  MutexLock& mutex_;
  pthread_cond_t pcond_;
};
```

pthread_cond_wait和pthread_cond_timedwait会在等待时：
1. 释放锁
2. 等待条件满足
3. 重新加锁
因此，如果在等待过程中，其他线程调用了MutexLock的unlock()，会出现下面的问题：
- 1. 线程A调用Condition的wait()，进入等待状态，此时线程A会释放锁，但holder_仍然是线程A的id
- 2a. 线程B调用MutexLock的unlock()，此时线程A的holder_仍然是线程A的id，导致线程B无法成功解锁，出现死锁
- 2b. 线程B调用lock()，线程B获得了锁，也可以成功解锁，但在线程A重新加锁时，发现holder_是0，再次调用assertLocked()或者unlock()时会出现断言失败

故而Condition的wait()和waitForSeconds()利用了MutexLock的UnassignGuard的RAII机制：
- 线程进入等待前暂时将holder_置为0，表示当前线程不持有锁
- 线程等待结束后将holder_重新设置为当前线程id，表示当前线程重新持有锁