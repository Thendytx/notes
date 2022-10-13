1. 多进程
    1. master-worker
        1. master与worker之间通过信号量和 进行通信
        2. worker之间使用共享内存
    2. 高性能
        1. 异步非阻塞
            1. 多种支持，Linux下默认epoll，还有poll、select、kqueue、eventport等。
            2. event模块封装了这些不同的事件驱动模型
        2. cpu绑定
            1. worker进程数一般设置为cpu核心数，通过worker_cpu_affinity将worker进程与固定的核绑定，避免cpu频繁进行上下文切换，提高cpu缓存命中率
    3. 进程间负载均衡
        1. 全局锁
            ngx_trylock_accept_mutex<br>
            监听之前抢锁，获取到accept_mutex才监听端口并接受新的连接
        2. 负载均衡算法
            每个worker进程有一个ngx_accept_disabled全局变量<br>
            初始化：ngx_accept_disabled = ngx_cycle->connection_n / 8 - ngx_cycle->free_connection_n;<br>
            其中，ngx_cycle->connection_n是每个worker进程可同时接受的最大连接数，ngx_cycle->free_connection_n为空闲连接数
            <br>初始值为-7/8 * ngx_cycle->connection_n。抢到时，该值+1。<br>ps:最大连接数一般为8的整数倍，如1024等。
            <br>当ngx_accept_disabled为正数时，说明已经使用了最大连接数的7/8，worker非常繁忙，将放弃本次对accept_mutex的争抢，并使ngx_accept_disabled减一
2. 模块化
    1. 5类模块（官方分类）
        1. 核心模块
            每个核心模块定义了一种模块风格，名称格式为ngx_xxx_module<br>
            xxx包括：core、http、events、mail、openssl、errlog
        2. 配置模块
            只有ngx_conf_module一个模块。<br>
            其他模块生效前都要依赖配置模块处理配置指令、完成准备工作
        3. HTTP模块
            处理http请求
        4. Event模块
            一系列在不同系统、不同内核版本的事件驱动模块
        5. Mail模块
            邮件服务相关的模块，IMAP、POP3、SMTP等的代理
    2. 模块接口
        1. 模块接口规范

                struct ngx_mudule_t{
                    ngx_uint_t      ctx_index;
                    ngx_uint_t      index;
                    char           *name;
                    ngx_uint_t      spare0;
                    ngx_uint_t      spare1;
                    ngx_uint_t      version;
                    const char     *signature;
                    void           *ctx;                                //各类模块的规范通过该变量进行抽象
                    ngx_command_t  *commands;
                    ngx_uint_t      type;

                    ngx_int_t     (*init_master)    (ngx_log_t *log);
                    ngx_int_t     (*init_module)    (ngx_cycle_t *cycle);
                    ngx_int_t     (*init_process)   (ngx_cycle_t *cycle);
                    ngx_int_t     (*init_thread)    (ngx_cycle_t *cycle);
                    void          (*exit_thread)    (ngx_cycle_t *cycle);
                    void          (*exit_process)   (ngx_cycle_t *cycle);
                    void          (*exit_master)    (ngx_log_t *log);

                    uintptr_t       spare_hook0;
                    uintptr_t       spare_hook1;
                    uintptr_t       spare_hook2;
                    uintptr_t       spare_hook3;
                    uintptr_t       spare_hook4;
                    uintptr_t       spare_hook5;
                    uintptr_t       spare_hook6;
                    uintptr_t       spare_hook7;
                };
            1. 核心模块
                ctx指向ngx_core_mudule_t结构体。所有核心模块要按照这个结构体的规范来实现。<br>

                    typedef struct{
                        ngx_str_t       name;
                        void         *(*create_conf)        (ngx_cycle_t *cycle);
                        char         *(*init_conf)          (ngx_cycle_t *cycle, void *conf);
                    }ngx_core_module_t;
            2. HTTP模块
                ctx指向ngx_http_module_t。<br>

                    typedef struct {
                        ngx_int_t   (*preconfiguration)     (ngx_conf_t *cf);
                        ngx_int_t   (*postconfiguration)    (ngx_conf_t *cf);

                        void       *(*create_main_conf)     (ngx_conf_t *cf);
                        char       *(*init_main_conf)       (ngx_conf_t *cf, void *conf);

                        void       *(*create_srv_conf)      (ngx_conf_t *cf);
                        char       *(*merge_srv_conf)       (ngx_conf_t *cf, void *prev, void *conf);

                        void       *(*create_loc_conf)      (ngx_conf_t *cf);
                        char       *(*merge_loc_conf)       (ngx_conf_t *cf, void *prev, void *conf);
                    } ngx_http_module_t;
    3. 模块分工
        1. nginx主框架关注6个核心模块的实现，每个核心模块管理各个相关的模块。<br>如，http模块统一由ngx_http_module管理，包括创建各HTTP模块存储配置项的结构体和执行各模块的初始化操作的时间点。
        2. 启动时要进行配置文件的解析：基础是配置模块和解析引擎
            1. 找到对该指令感兴趣的模块并调用该模块预先设定好的处理函数<br>
            多数情况下，这里会将参数保存到该模块存储配置项的结构体中并进行初始化操作<br>
            核心模块在启动过程中
                1. 会创建用于保存该“家族”所有存储配置结构体的容器
                2. 会按顺序将各结构体组织起来
                从而，核心模块可以管理其他该类型的模块；nginx按照序号从这些全局容器中迅速获取某个模块的配置项
        3. 启动时，Event模块根据用户配置以及操作系统选择一个事件驱动模型
            Linux中默认为epoll。<br>
            在Worker进程被派生（fork）出来并进入初始化阶段时，Event模块会创建各自的epoll对象，并通过epoll_ctl系统调用将监听端口的fd参数添加到epoll中。
        4. 用户请求处理
            主要是HTTP模块<br>
            Nginx将HTTP模块划分为11个阶段，每个阶段理论上允许多个模块执行相应的逻辑。<br>
            各HTTP模块会将各自的handler函数以hook的形式挂载到某个阶段。ps：这里可能还不太懂。hook和callback有什么区别呢。<br>
            Nginx的Event模块会根据各种事件调度HTTP模块依次执行各阶段的handler处理方法，并通过返回值来判定是继续向下执行还是结束当前请求，这种流水线式的请求处理流程使各HTTP模块完全解耦<br>
            开发者在完成模块核心处理逻辑之后，只需要考虑将handler函数注册到哪个阶段即可。
3. 事件驱动框架
    1. 一般事件驱动框架
        事件收集器、事件分发器、事件处理器<br>
        1. 事件收集器
            收集所有事件
        2. 事件分发器
            将事件分发到目标对象
        3. 事件处理器
            事件的消费者，接收分发来的事件并处理
    2. nginx事件驱动
        1. Event模块作为事件管理器(事件收集器？)和事件分发器
        2. 任何模块都可能成为事件消费者，处理完之后将控制权交还给Event模块
    3. 事件驱动模型
        如上，各种都有实现，Linux默认为epoll