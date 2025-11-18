```c++


 #include <boost/algorithm/string.hpp>
    #include <boost/regex.hpp>
    #include <boost/filesystem.hpp>

    std::string str = "one,two,three,four"; 
    std::vector<std::string> result;
    boost::split(result, str, boost::is_any_of(","));

    for (const auto& s : result) {
        std::cout << s << std::endl;
    }
    std::string str2 = "Hello World,Hello World";
    boost::algorithm::replace_all(str2,"World","Boost");
    std::cout << str2 << std::endl; // 输出: Hello Boost,Hello Boost

    std::string str3 = "   Hello Boost   ";
    boost::trim(str3);
    std::cout << str3 << std::endl; //输出：Hello Boost
    
    std::string str4 = "Hello World,Hello World";
    boost::replace_first(str4,"World","Boost");
    std::cout << str4 << std::endl; // 输出: Hello Boost,Hello World

    std::string str5 = "Hello World,Hello World";
    boost::replace_last(str5,"World","Boost");
    std::cout << str5 << std::endl; // 输出: Hello World,Hello Boost

    std::string str6 = "2024-05-20 13:45:30";
    boost::regex date_regex("-");
    std::string result2 = boost::regex_replace(str6, date_regex, "/");
    std::cout << result2 << std::endl; // 输出: 2024/05/20 13:45:30

    std::string str7 = "Apple banana apple orange APPLE,Apple,Apple,Apple";
    boost::replace_nth(str7,"Apple",3,"Banana");
    std::cout << str7 << std::endl; //输出 Apple banana apple orange APPLE,Apple,Apple,Banana
```