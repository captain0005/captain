```
#include <boost/asio.hpp>
#include <boost/beast.hpp>
#include <boost/json.hpp>
#include <iostream>
#include <functional>
#include <map>
#include <string>
#include <thread>
#include <boost/thread.hpp>
#include <csignal>
#include <vector>
#include <memory>

namespace asio = boost::asio;
namespace beast = boost::beast;
namespace http = beast::http;
namespace json = boost::json;
using tcp = asio::ip::tcp;

// 全局标志，用于控制服务器退出
volatile std::sig_atomic_t g_running = true;

// 信号处理函数
void signalHandler(int signal) {
    std::cout << "Received signal " << signal << ", shutting down..." << std::endl;
    g_running = false;
}

// 定义处理器函数类型
typedef std::function<void(const http::request<http::string_body>&, http::response<http::string_body>&)> RequestHandler;

// 路由管理器类，用于注册和查找路由
class Router {
private:
    std::map<std::string, RequestHandler> routes;
    std::string getRouteKey(http::verb method, const std::string& target) {
        return std::to_string(static_cast<int>(method)) + ":" + target;
    }
public:
    void registerRoute(http::verb method, const std::string& target, RequestHandler handler) {
        std::string key = getRouteKey(method, target);
        routes[key] = handler;
    }
    bool handleRequest(const http::request<http::string_body>& req, http::response<http::string_body>& res) {
        std::string key = getRouteKey(req.method(), std::string(req.target()));
        auto it = routes.find(key);
        if (it != routes.end()) {
            it->second(req, res);
            return true;
        }
        return false;
    }
};

// 全局路由管理器实例
Router g_router;

// 初始化路由表
void initializeRoutes() {
    g_router.registerRoute(http::verb::get, "/status", [](const http::request<http::string_body>& req, http::response<http::string_body>& res) {
        json::object response_json;
        response_json["status"] = "Server is running!";
        res.result(http::status::ok);
        res.set(http::field::content_type, "application/json");
        res.body() = json::serialize(response_json);
        res.prepare_payload();
    });

    g_router.registerRoute(http::verb::post, "/greet", [](const http::request<http::string_body>& req, http::response<http::string_body>& res) {
        json::object response_json;
        try {
            json::value parsed_body = json::parse(req.body());
            std::string name = parsed_body.as_object()["name"].as_string().c_str();
            response_json["message"] = "Hello, " + name + "!";
        } catch (...) {
            response_json["error"] = "Invalid JSON format.";
        }
        res.result(http::status::ok);
        res.set(http::field::content_type, "application/json");
        res.body() = json::serialize(response_json);
        res.prepare_payload();
    });
    // 可以根据需要继续添加更多路由...
}

// 处理请求的函数
void handle_request(
    http::request<http::string_body> req,
    http::response<http::string_body>& res
) {
    json::object response_json;
    std::cout << "Method: " << req.method() << "...\n";
    std::cout << "Target: " << req.target() << "...\n";
    std::cout << "Body: " << req.body() << "...\n\n";
    if (!g_router.handleRequest(req, res)) {
        response_json["error"] = "Unknown endpoint.";
        res.result(http::status::not_found);
        res.set(http::field::content_type, "application/json");
        res.body() = json::serialize(response_json);
        res.prepare_payload();
    }
}

// 异步会话类
class Session : public std::enable_shared_from_this<Session> {
public:
    Session(tcp::socket socket)
        : socket_(std::move(socket)) {}

    void start() {
        do_read();
    }

private:
    tcp::socket socket_;
    beast::flat_buffer buffer_;
    http::request<http::string_body> req_;

    void do_read() {
        auto self = shared_from_this();
        http::async_read(socket_, buffer_, req_,
            [self](beast::error_code ec, std::size_t) {
                if (!ec) {
                    self->handle_request();
                }
                // 连接关闭由客户端控制，或可在此处关闭
            });
    }

    void handle_request() {
        http::response<http::string_body> res;
        ::handle_request(req_, res);

        auto self = shared_from_this();
        http::async_write(socket_, res,
            [self](beast::error_code ec, std::size_t) {
                beast::error_code ignore_ec;
                self->socket_.shutdown(tcp::socket::shutdown_send, ignore_ec);
                // 会话对象自动析构
            });
    }
};

// 异步accept
void do_accept(asio::io_context& ioc, tcp::acceptor& acceptor) {
    acceptor.async_accept(
        [&ioc, &acceptor](beast::error_code ec, tcp::socket socket) {
            if (!ec) {
                std::make_shared<Session>(std::move(socket))->start();
            }
            if (g_running) {
                do_accept(ioc, acceptor);
            }
        }
    );
}

// 异步服务器主函数
void run_server(asio::io_context& ioc, unsigned short port, unsigned int thread_pool_size) {
    tcp::acceptor acceptor(ioc, tcp::endpoint(tcp::v4(), port));
    do_accept(ioc, acceptor);

    std::cout << "HTTP Server running on port " << port << "...\n\n";
    std::vector<std::thread> threads;
    for (unsigned int i = 0; i < thread_pool_size; ++i) {
        threads.emplace_back([&ioc]() { ioc.run(); });
    }
    for (auto& t : threads) t.join();
    std::cout << "Server shutdown complete\n";
}

int main() {
    try {
        std::signal(SIGINT, signalHandler);
        std::signal(SIGTERM, signalHandler);

        initializeRoutes();
        asio::io_context io_context;

        unsigned int thread_pool_size = std::thread::hardware_concurrency();
        if (thread_pool_size == 0)
            thread_pool_size = 4;
        run_server(io_context, 8080, thread_pool_size);
    }
    catch (std::exception& e) {
        std::cerr << "Server error: " << e.what() << "\n";
    }
    return 0;
}
```
###run_server函数是异步HTTP服务器的核心启动与管理函数，主要功能如下：

**服务器初始化与启动：**

创建TCP acceptor并绑定到指定的IPv4端口
调用do_accept函数启动异步连接接受循环，使服务器能够非阻塞地接受新连接
输出服务器启动信息，指示服务器正在哪个端口运行
**线程池创建与管理：**

创建std::vector<std::thread>容器来管理工作线程池
根据thread_pool_size参数，创建指定数量的工作线程
每个线程都运行io_context.run()，启动I/O事件循环
选中的代码行threads.emplace_back([&ioc]() { ioc.run(); });创建一个新线程并捕获io_context引用，使该线程开始处理异步操作队列
**服务器生命周期管理：**

使用join()等待所有工作线程完成，确保服务器优雅关闭
线程池中的所有线程共享同一个io_context实例，协同处理异步事件
当io_context停止时（通常由信号处理器设置g_running=false触发），所有线程会退出其事件循环
输出服务器关闭完成的信息
**工作原理：**

这种设计采用了Boost.Asio的"线程池 + 单io_context"模式，提高了服务器处理并发请求的能力
主线程负责初始化和等待，而工作线程处理实际的网络I/O事件
异步接受连接和异步处理请求相结合，实现了高性能的非阻塞服务器架构
该函数是整个异步HTTP服务器的骨架，通过线程池机制充分利用多核CPU资源，同时保持异步非阻塞的高效特性。

###HTTP请求在线程池中的处理流程遵循Boost.Asio的异步事件驱动模型，具体过程如下：

**1. 异步连接接受阶段**
当客户端发起连接请求时：

do_accept函数调用acceptor.async_accept()发起非阻塞异步接受操作
这个操作会被提交到io_context的事件队列中
线程池中的任一工作线程（通过ioc.run()）会处理这个异步接受操作
当有连接到达时，系统会从线程池中选择一个线程来执行回调函数
**2. 请求处理初始化阶段**
连接接受成功后：

回调函数中创建Session共享指针对象：std::make_shared<Session>(std::move(socket))->start()
这个操作仍然在线程池中的线程上执行
Session::start()方法被调用，启动请求处理流程
**3. 异步请求读取阶段**
在Session::do_read()中：

调用http::async_read()发起异步读取HTTP请求
这个异步操作再次被提交到io_context事件队列
此时执行权会交还给线程池，当前线程可以继续处理其他事件
当有数据到达时，系统会再次从线程池中分配一个可用线程来执行读取完成的回调
**4. 请求处理与响应阶段**
请求读取完成后：

回调函数中调用self->handle_request()处理请求
请求处理可以在任意一个线程池线程上执行
处理完请求后，调用http::async_write()异步发送响应
响应发送完成后关闭连接
###线程池调度机制
**整个过程中，线程池的工作原理是：**

所有线程共享同一个io_context实例，通过ioc.run()启动事件循环
当有异步操作完成（连接到达、数据可读、写入完成等），io_context会从事件队列中取出对应的回调任务
从线程池中选择一个可用的工作线程来执行该回调任务
线程执行完回调后，会继续等待处理io_context中的下一个任务
**关键技术点**
非阻塞模型：所有I/O操作都是异步非阻塞的，线程不会被长时间阻塞在I/O等待上
共享执行上下文：多线程共享同一个io_context，避免了线程间同步的复杂性
任务调度：Boost.Asio自动负责将异步任务分发给可用线程，实现负载均衡
零拷贝优化：通过std::move避免不必要的数据拷贝，提高性能
这种设计使得服务器能够高效处理大量并发连接，每个连接的各个阶段（接受、读取、处理、响应）可能在不同的线程上执行，但都是由同一个线程池统一调度管理。
