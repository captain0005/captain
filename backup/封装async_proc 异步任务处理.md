###封装代码
```
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>
#include <atomic>

// 1. 定义你的信号量类（不变）
class vp_semaphore {
public:
    vp_semaphore() : count_(0) {}
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


class async_proc {
public:
    using Task = std::function<void()>;
    async_proc()
    {
        tid = std::thread([this]() {
            while (live) {
                semaphore_cv.wait();
                if (!live)
                    break;
                // std::lock_guard<std::mutex> lock(task_mtx); 可以不加 semaphore_cv中已经有对 task_que数量的判断
                if (task_que.empty()) {
                    continue;
                }
                auto tsk = task_que.front();
                task_que.pop();
                tsk();
            }
        });
    }
    ~async_proc()
    {
        live = false;
        semaphore_cv.signal();
        tid.join();
    }

    template <typename F, typename... Args> void post_task(F &&f, Args &&...args)
    {
        Task tsk = std::bind(std::forward<F>(f), std::forward<Args>(args)...);
        std::lock_guard<std::mutex> lock(task_mtx);
        task_que.push(tsk);
        semaphore_cv.signal();
    }

    async_proc(const async_proc &) = delete;
    async_proc &operator=(const async_proc &) = delete;

private:
    std::queue<Task> task_que;
    std::mutex task_mtx;
    //声明为原子变量，确保在多线程环境下的安全访问
    std::atomic<bool> live{true};
    std::thread tid;
    vp_semaphore semaphore_cv;
};
```

###使用示例
```
// 简单的函数示例
void print_hello() {
    std::cout << "Hello from thread! Thread ID: " << std::this_thread::get_id() << std::endl;
}

// 带参数的函数示例
void print_message(const std::string& message, int number) {
    std::cout << "Message: " << message << ", Number: " << number 
              << " (Thread ID: " << std::this_thread::get_id() << ")" << std::endl;
}

// 带返回值的函数示例（在异步环境中返回值会被忽略，除非通过引用或指针传递）
void calculate_and_store(int a, int b, int& result) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 模拟耗时操作
    result = a + b;
    std::cout << "Calculated: " << a << " + " << b << " = " << result << std::endl;
}

int main() {
    std::cout << "Main thread ID: " << std::this_thread::get_id() << std::endl;
    
    // 创建异步处理器实例
    async_proc processor;
    
    // 示例1：提交无参函数
    std::cout << "Submitting print_hello task..." << std::endl;
    processor.post_task(print_hello);
    
    // 示例2：提交带参数的函数
    std::cout << "Submitting print_message task..." << std::endl;
    processor.post_task(print_message, "Async task", 42);
    
    // 示例3：提交Lambda表达式
    std::cout << "Submitting lambda task..." << std::endl;
    processor.post_task([]() {
        std::cout << "Lambda executed! " 
                  << "Current time: " << std::chrono::system_clock::to_time_t(std::chrono::system_clock::now()) << std::endl;
    });
    
    // 示例4：提交带状态的任务（使用引用传递结果）
    int result = 0;
    std::cout << "Submitting calculation task..." << std::endl;
    processor.post_task(calculate_and_store, 10, 32, std::ref(result));
    
    // 示例5：提交多个任务，展示并发执行
    std::cout << "Submitting multiple tasks..." << std::endl;
    for (int i = 0; i < 5; ++i) {
        processor.post_task([i]() {
            std::cout << "Batch task " << i << " executing..." << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(50)); // 模拟不同的执行时间
            std::cout << "Batch task " << i << " completed." << std::endl;
        });
    }
    
    // 等待所有任务完成
    std::this_thread::sleep_for(std::chrono::seconds(1));
    
    // 显示通过引用获取的计算结果
    std::cout << "Final result from calculation: " << result << std::endl;
    
    std::cout << "Main function completed." << std::endl;
    return 0;
}
```