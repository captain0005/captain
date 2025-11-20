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
    // 路由表，键为HTTP方法+路径，值为对应的处理函数
    std::map<std::string, RequestHandler> routes;
    
    // 生成路由键
    std::string getRouteKey(http::verb method, const std::string& target) {
        return std::to_string(static_cast<int>(method)) + ":" + target;
    }
    
public:
    // 注册路由
    void registerRoute(http::verb method, const std::string& target, RequestHandler handler) {
        std::string key = getRouteKey(method, target);
        routes[key] = handler;
    }
    
    // 查找并执行路由处理函数
    bool handleRequest(const http::request<http::string_body>& req, http::response<http::string_body>& res) {
        std::string key = getRouteKey(req.method(), std::string(req.target()));
        
        auto it = routes.find(key);
        if (it != routes.end()) {
            // 找到对应的处理函数，执行它
            it->second(req, res);
            return true;
        }
        
        return false; // 未找到匹配的路由
    }
};

// 全局路由管理器实例
Router g_router;

// 初始化路由表
void initializeRoutes() {
    // 注册GET /status路由
    g_router.registerRoute(http::verb::get, "/status", [](const http::request<http::string_body>& req, http::response<http::string_body>& res) {
        json::object response_json;
        response_json["status"] = "Server is running!";
        
        // 设置响应
        res.result(http::status::ok);
        res.set(http::field::content_type, "application/json");
        res.body() = json::serialize(response_json);
        res.prepare_payload();
    });
    
    // 注册POST /greet路由
    g_router.registerRoute(http::verb::post, "/greet", [](const http::request<http::string_body>& req, http::response<http::string_body>& res) {
        json::object response_json;
        
        try {
            // 解析请求体
            json::value parsed_body = json::parse(req.body());
            
            // 提取name字段
            std::string name = parsed_body.as_object()["name"].as_string().c_str();
            
            // 构造问候消息
            response_json["message"] = "Hello, " + name + "!";
        } catch (...) {
            // 错误处理
            response_json["error"] = "Invalid JSON format.";
        }
        
        // 设置响应
        res.result(http::status::ok);
        res.set(http::field::content_type, "application/json");
        res.body() = json::serialize(response_json);
        res.prepare_payload();
    });
    
    // 可以根据需要继续添加更多路由...
}


// Function to handle incoming HTTP requests and produce appropriate responses
void handle_request(
    http::request<http::string_body> req,         // Incoming HTTP request
    http::response<http::string_body>& res        // Outgoing HTTP response (to be filled in)
) {
    // JSON object to hold the response body
    json::object response_json;

    // Log request information to the console for debugging
    std::cout << "Method: " << req.method() << "...\n";
    std::cout << "Target: " << req.target() << "...\n";
    std::cout << "Body: " << req.body() << "...\n\n";

    // 尝试处理请求
    if (!g_router.handleRequest(req, res)) {
        // 如果路由管理器未处理请求，返回404 Not Found
        response_json["error"] = "Unknown endpoint.";
        res.result(http::status::not_found);
        res.set(http::field::content_type, "application/json");
        res.body() = json::serialize(response_json);
        res.prepare_payload();
    }
}


// 处理单个连接的函数
void handle_connection(tcp::socket socket) {
    try {
        beast::flat_buffer buffer;
        http::request<http::string_body> req;
        http::read(socket, buffer, req);

        http::response<http::string_body> res;
        handle_request(req, res);

        http::write(socket, res);
    } catch (std::exception& e) {
        std::cerr << "Connection error: " << e.what() << "\n";
    }
}

// HTTP Server function
void run_server(asio::io_context& ioc, unsigned short port, unsigned int thread_pool_size) {
     // 创建线程池
    asio::thread_pool pool(thread_pool_size);

    tcp::acceptor acceptor(ioc, tcp::endpoint(tcp::v4(), port));

    std::cout << "HTTP Server running on port " << port << "...\n\n";

    while (true) {
        tcp::socket socket(ioc);
        acceptor.accept(socket);

        // 将连接处理任务提交到线程池
        asio::post(pool, [socket = std::move(socket)]() mutable {
            handle_connection(std::move(socket));
        });

        // beast::flat_buffer buffer;
        // http::request<http::string_body> req;
        // http::read(socket, buffer, req);

        // http::response<http::string_body> res;
        // handle_request(req, res);

        // http::write(socket, res);
    }
    std::cout << "Server shutting down, waiting for active connections...\n";
    // 等待所有线程池任务完成
    // 等待所有线程池任务完成
    pool.join();
    std::cout << "Server shutdown complete\n";
}

int main() {
    try {

        // 注册信号处理
        std::signal(SIGINT, signalHandler);
        std::signal(SIGTERM, signalHandler);

        // 初始化路由表
        initializeRoutes();
        asio::io_context io_context;
        //因为使用的是同步，所以不需要启动io_context的工作线程
        // // 启动io_context的工作线程
        // boost::thread io_thread([&io_context]() {
        //     io_context.run();
        // });
        
        // 线程池大小可以根据CPU核心数或需求调整
        unsigned int thread_pool_size = std::thread::hardware_concurrency();
        if (thread_pool_size == 0)
            thread_pool_size = 4; // 默认值
        run_server(io_context, 8080, thread_pool_size);

        // 关闭io_context
        // io_context.stop();
        // if (io_thread.joinable()) {
        //     io_thread.join();
        // }

    }
    catch (std::exception& e) {
        std::cerr << "Server error: " << e.what() << "\n";
    }
    return 0;
}```