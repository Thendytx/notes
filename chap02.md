1. 概述

    熟悉nginx配置文件，包括语法结构、基础配置、使用http核心模块配置静态web服务器的方法以及反向代理服务器
2. 运行时nginx进程间关系
    
    worker进程之间通过共享内存、原子操作等进程间通信机制实现负载均衡功能
    1. master进程可以单独提供服务。 
    2. master进程需要较大的权限，通常使用root用户启动master进程，worker进程的权限要小于或等于master进程，这样master进程才能完全管理worker进程。当任意worker进程出现错误导致coredump时，master进程会立即启动新的worker进程。
    3. 多worker可以提高服务的健壮性，充分利用多核架构进行并发处理。一个worker进程可以同时处理请求数只受限于内存大小，不同worker进程之间处理并发请求时几乎没有同步锁限制，worker进程通常不会进入睡眠状态。在进程数与cpu核心数相同时，能够充分利用每个CPU核心，（最好再加上cpu核心绑定），并且进程间切换代价最低。
3. nginx配置通用语法
    1. 配置项

        配置项由 配置项名 + 配置项值列表 组成<br>必须具体模块对具体配置项的要求来确定对于配置项格式的约定， 详见chap04
        1. 块配置项

            这是指 配置项值是 由一对大括号包起来的一系列配置项 的配置项， 大括号内的配置项同时生效
            <br>
            块配置项可以嵌套， 内层块的配置继承外层块的配置； 内外块的配置相冲突时， 具体使用哪个配置， 取决于具体的模块。
        2. 配置项语法格式

            1. 基本配置项语法

                配置项名 配置项值1 配置项值2 ……;
                1. 配置项名：必须是某个模块希望处理的， 否则nginx认为非法。 配置项名以空格作为结束符
                2. 配置项值：可以是数字或字符串， 可以有正则表达式。 当有多个配置项值时， 它们之间以空格作为分隔符， 具体可以或必须有多少个配置项值， 取决于具体的模块。 
                3. 分号：每行配置的结束标识
                4. 空格：默认情况下， 空格作为分隔符解释。 如果配置项值中包含空格符或*其他语法符号*， 应当使用单引号或双引号将配置项值括住。
        3. 配置项的注释

            '\#' 注释一行配置项
        4. 配置项的单位

            大部分模块的配置项遵循一些通用的规定， 如空间不必以字节数指定， 时间不必以毫秒数指定。<br>当然， 具体是否可以使用这些单位取决于具体的模块。<br>chap04描述了nginx框架提供的14种预设解析方法， 其中一些方法可以解析下列单位。
            1. 空间单位

                K/k，M/m，分别代表KB与MB
            2. 时间单位

                ms, s, m, h, d, w, M, y<br>毫秒、秒、分、时、日、周、月、年
        5. 配置中的变量

            有的模块支持在配置项中使用变量； 变量以$在开头标识。<br>许多模块在解析请求时会提供多个变量， 以使其他模块可以即时使用。
4. nginx基本配置

    nginx运行时， 几个核心模块和一个事件类模块是必须加载的。 这些模块运行时支持的配置项成为基本配置， 是其他模块执行时都依赖的配置项。<br>
    一些配置项， 在没有显式配置时， 也有默认值。
    1. 第一类， 用于调试进程和定位问题的配置项
        1. daemon on|off;
            默认为daemon on;。 设定是否以守护进程方式运行。 off时， 便于跟踪调试nginx， 尤其是对于fork出的子进程
        2. master_process on|off;
            默认为master_process on;。 设定是否以master/worker方式工作。 off时， 便于跟踪调试nginx， 因为此时没有子进程。
        3. error_log /path/file level;
            默认为error_log logs/error.log error;。<br>error日志用于定位nginx问题， 应根据需求设置error日志的路径和级别。<br>/path/file可以是具体文件， 也可以是/dev/null， 这样就相当于关闭了日志， 或者也可以是strerr， 这样就输出到了标准错误<br>level是输出级别， 有debug、info、notice、warn、error、crit、alert、emerg。 nginx会输出大于等于设定等级的所有日志信息。 等级较低时， 应注意使/path/file有足够的空间。<br>注： 如果设置为debug级别， 必须在configure时加入--with-debug配置项。
        4. debug_points [stop|abort];
            对于几个关键错误逻辑， nginx设置了调试点。 如果这里设置为stop， 则nginx会发送SIGSTOP信号以调试。 如果这里设置为abort， 则产生一个coredump文件。<br>通常不用这个配置
        5. debug_connnection [IP|CIDR];
            仅对指定的客户端输出debug级别的日志。<br>该配置属于事件类配置， 因此应当放到events配置块中。 它的值可以是IP地址或CIDR地址。 如10.224.66.14或10.224.57.0/24。<br>这样， 只有来自以上IP的请求才会输出debug级别的日志， 其他请求沿用error_log中配置的日志级别。<br>上面的配置对修复bug很有用， 特别是定位高并发请求下才有的问题时。<br>执行configure加入--with-debug参数， 本配置项才会生效。
        6. worker_rlimit_core size;
            限制coredump文件的大小。<br>
            进程出现错误或收到信号终止时， linux系统会将进程执行时内存内容(核心映像)写入core文件， 这就是核心转储。 内存越界等行为会导致进程被操作系统强制结束并生成core文件， 其中有出错时的堆栈、寄存器内容等信息。<br>但core文件可能很大， 为防止磁盘占满， 可以限制core文件大小。
        7. working_directory path;
            指定coredump文件生成目录。 worker进程的工作目录。 这个配置项的唯一用途就是设置coredump文件的目录， 以便定位问题。 因此， 需要确保worker进程有权限向该配置项指定的目录写入文件。
    2. 第二类， 正常运行的配置项
        1. env VAR|VAR=VALUE;
            设置系统环境变量。 如， env TESTPATH=/tmp/;
        2. include /path/file;
            将其他配置文件嵌入当前的nginx.conf中。 它的参数可以是绝对路径也可以是nginx.conf的相对路径， 并且支持使用通配符
        3. pid path/file;
            默认：pid logs/nginx.pid;
            pid文件保存master进程的pid。 默认与configure执行时的--pid-path路径相同， 也可以随时修改， 但应确保nginx有权在相应目录创建pid文件， 因为pid文件直接影响到nginx的运行
        4. user username [groupname];
            默认：user nobody nobody;
            user配置项用于设置master进程启动后， fork出的worker进程的用户与用户组。 如果缺省groupname， 认为组名与用户名相同。 若执行configure时使用了 --user=username 和 --group=groupname， 将使用参数中指定的用户和用户组。
        5. worker_rlimit_nofile_limit;
            限制一个worker进程最多打开的句柄数。
        6. worker_rlimit_sigpending limit;
            设置每个用户发往nginx的信号队列的大小， 超过该大小后， 队列不能容纳， 该用户发送的信号量将被丢弃。
    3. 第三类， 优化性能
        1. worker_processes number;
            默认：worker_processes 1;<br>
            如果worker进程不会出现阻塞调用， 那么应设置与CPU内核数相同的进程数， 否则应多设置一些。
        2. worker_cpu_affinity cpumask [cpumask…]
            绑定nginx