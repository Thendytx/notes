1.  **字符串**
    1. **ngx_str_t**

            typedef struct {
                size_t      len;
                u_char     *data;
            } ngx_str_t;
        这样就可以克服C字符串被0截断的问题
    2. **字符串处理宏**
        1. ngx_string(str)<br>字符串初始化，str是普通的C语言字符串

                #define ngx_string(str)     { sizeof(str) - 1, (u_char *) str }
                // 要求str是一个字面值字符串。
                // 如果是数组，那么将不是字符串实际长度，而是数组大小-1
                // 如果是指针，那么将是指针大小-1
                // 因此只能是类似"asdadsa"这样的字面值字符串
                // 或者如果是数组，必须保证数组正好存放了字符串和0
        2. ngx_null_string<br>定义一个空字符串

                #define ngx_null_string     { 0, NULL }
        3. ngx_str_set(str, text)<br>修改字符串内容。str是已经存在的ngx_str_t。

                #define ngx_str_set(str, text)                                               \
                    (str)->len = sizeof(text) - 1; (str)->data = (u_char *) text
                // 可以看到，str应该是ngx_str_t结构，text应该是字面值字符串
        4. ngx_str_null(str)<br>将str设置为空。str同上。

                #define ngx_str_null(str)   (str)->len = 0; (str)->data = NULL
        5. ngx_tolower(c)<br>字符串首字母转小写

                #define ngx_tolower(c)      (u_char) ((c >= 'A' && c <= 'Z') ? (c | 0x20) : c)
        6. ngx_toupper(c)<br>字符串首字母转小写

                #define ngx_tolower(c)      (u_char) ((c >= 'a' && c <= 'z') ? (c & ~0x20) : c)
        7. ngx_strncmp(s1, s2, n)<br>比较s1、s2前n个字符

                实质是strncmp，要求s1、s2是一般的C语言字符串。即，能转成类型const char *的值。
        8. ngx_strcmp(s1, s2)<br>比较s1、s2

                实质是strcmp，要求s1、s2是一般的C语言字符串。即，能转成类型const char *的值。
        9. ngx_strstr(s1, s2)<br>判断s2是否为s1子串

                实质是strstr，要求s1、s2是一般的C语言字符串。即，能转成类型const char *的值。
        10. ngx_strlen(s)<br>获取字符串长度

                实质是strlen，要求s是一般的C语言字符串。即，能转成类型const char *的值。
        11. ngx_strchr(s1, c)<br>获取c在s1中首次出现的位置

                实质是strchr，要求s能转成const char *、c能转成int
        12. ngx_memcpy(dst, src, n)<br>将dst的前n个字节由src的前n个字节覆盖

                如果memcpy可用，那么使用memcpy。否则使用自行实现的ngx_memcpy。
        13. ngx_cpymem(dst, src, n)<br>将dst的前n个字节由src的前n个字节覆盖并返回dst从第n+1个字节开始的字符串

                如果memcpy可用，那么使用memcpy。否则使用自行实现的ngx_memcpy。
    3. **字符串处理函数**
2. **数组**
    1. **ngx_array_t**

            typedef struct {
                void        *elts;      // 指向实际存储数据的数据块
                ngx_uint_t   nelts;     // 已存放数据的数量
                size_t       size;      // 每个数据的大小
                ngx_uint_t   nalloc;    // 可存储数据的数量
                ngx_pool_t  *pool;      // 存储数组的内存池
            } ngx_array_t;
    2. **ngx_array_create**

            ngx_array_t *ngx_array_create(ngx_pool_t *p, ngx_uint_t n, size_t size);
        p是内存池地址，n是数组初始大小，size是每个数据的大小<br>
        首先从内存池分配数组结构体所需要的空间，并对结构体进行初始化。<br>
        结构体初始化时会再从内存池中分配n*size的空间，并将返回的地址赋值给elts参数。<br>
        结构体初始化完成后，由于还没有插入数据，nelts参数的值为0，nalloc和size参数的值为函数参数入参值。<br>
        初始化函数返回内存池中为数组分配的内存起始位置。
    3. **ngx_array_push**

            void *ngx_array_push(ngx_array_t *a);
        该函数的参数为一个数组的地址<br>
        该函数返回一个地址，用户可向该地址写入值来向传入的数组添加数据。<br>
        1. **ngx_array_push内部处理逻辑**
            1. 如果数组的容量和已分配的数量不相等，表示内存池中有足够的空间供数据存放，此时直接返回数据所需要的内存空间，并将数组已使用的个数加1

                    elt = (u_char *) a->elts + a->size * a->nelts;
                    a->nelts++;
           2. 如果数组已分配的数量等于数组的容量，表示此时数组已经存满。一般的做法是开辟一块较大的新内存空间，将当前数据全部赋值到新内存地址。由于Nginx的数组容量是在内存池上分配的，因此不一定需要新开辟空间，这需要依据内存池是否有新的可用空间来确定。

                    // 这部分代码应该是上一部分代码的后续。顺序结构的后续的意思。
                    // 即，执行到这里时，上面那部分已经执行完了
                    if ((u_char *) a->elts + size == p->d.last　&& p->d.last + a->size <= p->d.end)
                    // 内存池当前节点上仍有剩余空间存放数组新数据

                    // 问题：上面这个逻辑判断的前面是什么意思呢？不好说。
                    // 猜想：size是数组已用空间的大小，即size = a->nelts * a->size。
                    // 问题：什么情况下会出现(u_char *) a->elts + size != p->d.last的情况呢
                    {
                        p->d.last += a->size;   // 这里将last后移
                        a->nalloc++;            // 然后数组大小+1
                    } else {
                        new = ngx_palloc(p, 2 * size);  // 当内存池地址不够用时，需要新申请内存池。申请内存池
                                                        // 的大小是原数组大小的2倍
                        if (new == NULL) {
                            return NULL;
                        }
                        ngx_memcpy(new, a->elts, size);// 内存池初始化之后，将原数组依次赋值到新地址上
                        a->elts = new;
                        a->nalloc *= 2;

                        // 问题：这里书上应该说错了吧。ngx_palloc并不是创建新的内存池，而是从内存池中申请新的内存。
                        // 问题：pool什么时候更新呢？与之相关的另一个问题，当小块内存不够用的时候呢？为什么只检查p->d.last？
                        // 猜想：这里的代码应该不是全部的源码，编者编得有问题。
                    }
            3. 当存储数组的内存池有剩余空间时，插入数据直接在当前内存池上向后扩容；当存储数组的内存池空间不足时，需要开辟新的内存空间并将数组数据依次赋值到新内存地址上，以此来保证数组数据的连续性。
    4. **空间释放**
        1. 当数组使用完毕释放时，直接释放内存池上的空间即可，不用将内存交还给操作系统，从而保证申请和释放内存的高效性。被释放的内存仍可以另作他用，实现内存的重复利用，减少了系统开销。
3. **链表**
    1. **struct ngx_list_part_s**

            // 链表节点
            struct ngx_list_part_s {
                void             *elts;     // 指向节点的实际存储地址
                ngx_uint_t        nelts;    // nelts是当前节点已分配的数据量
                ngx_list_part_t  *next;     // 指向链表的下一个节点
            };
    2. **ngx_list_t**

            // 链表头
            typedef struct {
                ngx_list_part_t  *last;     // 指向链表的最后一个节点
                ngx_list_part_t   part;     // 链表的第一个节点
                size_t            size;     // 每个数据的大小
                ngx_uint_t        nalloc;   // 每个节点可存储的数据个数
                ngx_pool_t       *pool;     // 链表头部所在的内存池
            } ngx_list_t;
    3. **ngx_list_create**

        用于创建链表的函数

            ngx_list_t *ngx_list_create(ngx_pool_t *pool, ngx_uint_t n, size_t size);
        先从内存池中申请头部所占用的空间，并对ngx_list_t结构体进行初始化。<br>
        由于还没有插入数据，part的nelts参数值为0，next指向NULL。<br>
    4. **ngx_list_push**

        类似ngx_array_push

            void *ngx_list_push(ngx_list_t *list);
        1. **大致逻辑**
            1. 如果节点上已分配的数据数量还未达到nalloc值，表示内存池中有足够的空间存放新数据，此时直接返回数据所需要的内存空间，并将节点已使用的个数加1

                    elt = (char *) last->elts + l->size * last->nelts;
                    last->nelts++;
            2. 当链表节点上已分配的数据数量等于nalloc时，表示当前链表节点已经存满，需要重新开辟一个内存池来存放链表的新节点。

                    if (last->nelts == l->nalloc) {
                        last = ngx_palloc(l->pool, sizeof(ngx_list_part_t));
                        if (last == NULL) {
                            return NULL;
                        }

                        last->elts = ngx_palloc(l->pool, l->nalloc * l->size);
                        if (last->elts == NULL) {
                            return NULL;
                        }

                        last->nelts = 0;
                        last->next = NULL;

                        l->last->next = last;
                        l->last = last;
                    }
                    // ytx: 这里的l应该是链表头
    5. **链表使用空间的释放**

        Nginx在链表使用完毕之后，直接释放链表所在的内存池上的空间，这样的操作更简单而且不用考虑单个节点的内存释放，易于维护。

        问题：所以这个内存池只存放链表吗？
4. **队列**

    队列通常是一种先进先出的数据结构。在实际工作中，队列多用于异步的任务处理。Nginx的队列由**包含头节点**的双向循环链表实现，是一种双向队列。
    1. ngx_queue_t

        这是nginx的队列定义

            typedef struct ngx_queue_s  ngx_queue_t;
            struct ngx_queue_s {
                ngx_queue_t  *prev;
                ngx_queue_t  *next;
            };
    2. ngx_queue_init

            #define ngx_queue_init(q)                      \
                (q)->prev = q;                             \
                (q)->next = q
            // 可以看到，初始化就是将ngx_queue_t类型变量q的prev和next都指向自己，从而构成了一个空队列
    3. ngx_queue_insert_head与ngx_queue_insert_tail
        1. ngx_queue_insert_head
        
                #define ngx_queue_insert_head(h, x)        \
                    (x)->next = (h)->next;                 \
                    (x)->next->prev = x;                   \
                    (x)->prev = h;                         \
                    (h)->next = x
    4. ngx_queue_remove
        
        节点的删除只是将其移出队列，节点内存的释放要由用户来完成

            #define ngx_queue_remove(x)               \
                (x)->next->prev = (x)->prev;          \
                (x)->prev->next = (x)->next
    5. ngx_queue_data
            
        获取nginx队列中的数据

            #define ngx_queue_data(q, type, link)                    \
                (type *) ((u_char *) q - offsetof(type, link))
        1. offsetof
            1. C语言的一个库函数，它返回一个结构体成员相对于结构体开头的字节偏移量。
            2. q是一个队列节点，是ngx_queue_t类型的变量，同时也是某个结构体的成员。这个结构体是真正的队列上的节点。例如，该结构体可能是struct ngx_connection_s。
                1. struct ngx_connection_s
                        
                        struct ngx_connection_s {
                            void               *data;
                            ngx_event_t        *read;
                            ngx_event_t        *write;
                            ...
                            ngx_queue_t         queue;
                            ...
                        };
                2. 从q的地址，向前找offset(struct ngx_connection_s, queue)，就找到了data的地址。再通过强制类型转换，就可以取得节点存储的值。
    1. **队列操作**
        1. **队列拆分**
        
            以数据q为界，将队列h拆分为h和n两个队列，其中拆分后数据q位于第二个队列中。
            
                #define ngx_queue_split(h, q, n)     \
                    (n)->prev = (h)->prev;           \    // n->prev指向原最后节点
                    (n)->prev->next = n;             \    // 原最后节点的前一节点改为n
                    (n)->next = q;                   \    // q作为队列n的第一个节点
                    (h)->prev = (q)->prev;           \    // q之前的节点作为h队列的最后一个节点
                    (h)->prev->next = h;             \    // h队列新的最后一个节点的下个节点调整指向
                    (q)->prev = n;
                // q作为队列n的第一个节点           
                    
        2. **队列合并**
            
            将队列n添加到队列h的尾部
            
                #define ngx_queue_add(h, n)               \
                    (h)->prev->next = (n)->next;          \    // h的最后节点有了后续
                    (n)->next->prev = (h)->prev;          \    // n的第一个节点前面接到h的最后节点
                    (h)->prev = (n)->prev;                \    // n的最后一个节点是新的h队列的最后一个节点
                    (h)->prev->next = h;
                    // h队列新的最后一个节点的下个节点调整指向
        3. **其他队列操作宏**
            1. ngx_queue_empty(h)
            
                判空
            2. ngx_queue_head(h)
               
                取队头
            3. ngx_queue_last(h)
               
                 取得队列的最后一个节点
            4. ngx_queue_sentinel(h)
                
                取得队列结构指针
                ps:不知道啥意思
            5. ngx_queue_next(q)
                
                取得节点q的下一个节点
            6. ngx_queue_prev(h)
                
                取得q的前一个节点
5. **散列**