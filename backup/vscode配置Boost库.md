```
cmake_minimum_required(VERSION 3.16.3)
  project(testBoost VERSION 0.1.0)
  
  # 设置C++标准
  set(CMAKE_CXX_STANDARD 11)
  set(CMAKE_CXX_STANDARD_REQUIRED ON)
  
  # 查找Boost库（不需要特定组件）
  #find_package(Boost REQUIRED)
  #if(Boost_FOUND)
  #    message("Boost found: ${Boost_VERSION}")
  #   include_directories(${Boost_INCLUDE_DIRS})
  #else()
  #    message(FATAL_ERROR "Boost not found!")
  #endif()
  #需要thread和filessystem组件
  find_package(Boost REQUIRED COMPONENTS thread filesystem)
  if(Boost_FOUND)
      message("Boost found: ${Boost_VERSION}")
      include_directories(${Boost_INCLUDE_DIRS})
  else()
      message(FATAL_ERROR "Boost not found!")
  endif()
  
  # 添加可执行文件
  add_executable(testBoost version.cpp)
  #对于仅标头库，您无需链接到已编译的 Boost 库。相反，您只需找到 Boost 并添加包含目录：
  #target_include_directories(testBoost PRIVATE ${Boost_INCLUDE_DIRS})
  # 链接Boost库
  target_link_libraries(testBoost PRIVATE Boost::thread pthread
      Boost::filesystem
      )
```