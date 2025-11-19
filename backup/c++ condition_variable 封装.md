### 封装类
```

#include <condition_variable>
#include <mutex>

  // semaphore for queue/deque data structures in VideoPipe, used for producer-consumer pattern.
  // it blocks the consumer thread until data has come.
  class vp_semaphore
  {
  public:
      vp_semaphore() {
          count_ = 0;
      }

      void signal() {
          std::unique_lock<std::mutex> lock(mutex_);
          ++count_;
          cv_.notify_one();
      }

      void wait() {
          std::unique_lock<std::mutex> lock(mutex_);
          cv_.wait(lock, [=] { return count_ > 0; });
          --count_;
      }

  private:
      std::mutex mutex_;
      std::condition_variable cv_;
      int count_;
  };

```
### 调用方法
**其他类中声明** ：vp_semaphore in_queue_semaphore;

> 其他类的方法使用
生产者
// notify consumer of in_queue
in_queue_semaphore.signal();
消费者
// wait for producer, make sure in_queue is not empty.
in_queue_semaphore.wait();



