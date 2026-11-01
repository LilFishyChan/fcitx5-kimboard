cmake_minimum_required(VERSION 3.16)
project(inline-keyboard-status VERSION 0.1.0 LANGUAGES CXX)

# 版本号编进二进制：插件连上来时会推一行 version:<x.y.z>，键盘那边拿它和自己期望的
# 版本比对，不一致就提示用户更新 addon
add_compile_definitions(INLINE_KEYBOARD_VERSION="${PROJECT_VERSION}")

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Fcitx5Core REQUIRED)

add_library(inline-keyboard-status MODULE src/inline-keyboard-status.cpp
    src/addon-support.cpp
    src/client-channel.cpp
    src/heartbeat.cpp
    src/candidate-payload.cpp)
target_link_libraries(inline-keyboard-status PRIVATE Fcitx5::Core)

# 去掉 lib 前缀：conf 里写 Library=inline-keyboard-status 时 fcitx5 直接找
# inline-keyboard-status.so（实测它不会自动补 lib）。
set_target_properties(inline-keyboard-status PROPERTIES PREFIX "")

# 安装规则要装齐两个文件：库 + addon 的 .conf。
# 只装 .so 的话，`cmake --install --prefix /usr` 出来的树里缺 conf，
# fcitx5 根本不会认这个 addon。
include(GNUInstallDirs)
install(TARGETS inline-keyboard-status LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/fcitx5)

# conf 里的 @LIBDIR@ 必须在**安装时**按实际 prefix 展开：
# configure_file 用的是 configure 阶段的 prefix，`--install --prefix /usr`
# 装到别的前缀就会指向错误的库路径（包装到别人机器上直接坏掉）。
# 用 install(CODE) 让它在安装那一刻才算，用户级（~/.local）和系统级（/usr）都正确。
install(CODE "
    file(READ \"${CMAKE_CURRENT_SOURCE_DIR}/inline-keyboard-status.conf\" _kb_conf)
    string(REPLACE \"@LIBDIR@\" \"\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}/fcitx5\"
           _kb_conf \"\${_kb_conf}\")
    file(WRITE \"\$ENV{DESTDIR}\${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_DATADIR}/fcitx5/addon/inline-keyboard-status.conf\"
               \"\${_kb_conf}\")
    message(STATUS \"Installing: inline-keyboard-status.conf\")
")

# 单测：不依赖 fcitx5，直接编译跑断言
enable_testing()

add_executable(test-keysym-map tests/test_keysym_map.cpp)
target_include_directories(test-keysym-map PRIVATE src)
add_test(NAME keysym-map COMMAND test-keysym-map)

# 候选数据的序列化：addon 与插件之间的唯一数据契约
add_executable(test-candidate-payload
    tests/test_candidate_payload.cpp
    src/candidate-payload.cpp
)
target_include_directories(test-candidate-payload PRIVATE src)
add_test(NAME candidate-payload COMMAND test-candidate-payload)

# 客户端 socket 的写入 / 失败判定：fcitx5 崩在这里过（EBADF 被当致命异常），
# 每个分支都钉住。
add_executable(test-client-channel
    tests/test_client_channel.cpp
    src/client-channel.cpp
)
target_include_directories(test-client-channel PRIVATE src)
add_test(NAME client-channel COMMAND test-client-channel)

# 心跳调度：只响一次、第二个连接没心跳 —— 两个真实回归都在这里
add_executable(test-heartbeat
    tests/test_heartbeat.cpp
    src/heartbeat.cpp
    src/client-channel.cpp
)
target_include_directories(test-heartbeat PRIVATE src)
add_test(NAME heartbeat COMMAND test-heartbeat)
