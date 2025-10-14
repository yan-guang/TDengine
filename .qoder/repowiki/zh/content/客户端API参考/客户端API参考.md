# 客户端API参考

<cite>
**本文档引用的文件**   
- [taos.h](file://include/client/taos.h)
- [demo.c](file://examples/c/demo.c)
- [asyncdemo.c](file://examples/c/asyncdemo.c)
- [prepare.c](file://examples/c/prepare.c)
- [schemaless.c](file://examples/c/schemaless.c)
- [tmq.c](file://examples/c/tmq.c)
</cite>

## 目录
1. [连接管理](#连接管理)
2. [查询执行](#查询执行)
3. [结果集处理](#结果集处理)
4. [预处理语句](#预处理语句)
5. [异步API](#异步api)
6. [无模式写入](#无模式写入)
7. [数据生命周期示例](#数据生命周期示例)

## 连接管理

`taos_connect` 函数用于建立与TDengine数据库的连接。该函数是客户端应用程序与数据库交互的入口点。

### taos_connect

**功能描述**  
建立与TDengine数据库服务器的连接。

**函数原型**  
```c
TAOS *taos_connect(const char *ip, const char *user, const char *pass, const char *db, uint16_t port);
```

**参数说明**  
- `ip`: 服务器IP地址或主机名
- `user`: 用户名，默认为"root"
- `pass`: 密码，默认为"taosdata"
- `db`: 默认数据库名称，可为NULL
- `port`: 服务器端口，默认为0（使用默认端口）

**返回值**  
成功时返回TAOS连接句柄，失败时返回NULL。可通过`taos_errstr(NULL)`获取错误信息。

**线程安全**  
该函数是线程安全的，可在多线程环境中安全调用。

**使用示例**  
```c
TAOS *taos = taos_connect("localhost", "root", "taosdata", NULL, 0);
if (taos == NULL) {
    printf("连接服务器失败，原因：%s\n", taos_errstr(NULL));
    exit(1);
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L192-L192)
- [demo.c](file://examples/c/demo.c#L54-L73)

## 查询执行

TDengine C API提供了同步和异步两种查询执行模式，满足不同应用场景的需求。

### taos_query

**功能描述**  
执行SQL查询语句，支持DDL、DML等所有SQL操作。

**函数原型**  
```c
TAOS_RES *taos_query(TAOS *taos, const char *sql);
```

**参数说明**  
- `taos`: 有效的连接句柄
- `sql`: 要执行的SQL语句

**返回值**  
返回TAOS_RES结果集句柄。执行失败时，结果集不为NULL，需通过`taos_errno()`检查错误码。

**线程安全**  
该函数不是线程安全的，同一连接不应在多个线程中同时调用。

**使用示例**  
```c
TAOS_RES *result = taos_query(taos, "CREATE DATABASE demo");
if (taos_errno(result) != 0) {
    printf("创建数据库失败，原因：%s\n", taos_errstr(result));
    taos_free_result(result);
    return -1;
}
taos_free_result(result);
```

**Section sources**
- [taos.h](file://include/client/taos.h#L265-L265)
- [demo.c](file://examples/c/demo.c#L74-L132)

### taos_select_db

**功能描述**  
设置当前连接的默认数据库。

**函数原型**  
```c
int taos_select_db(TAOS *taos, const char *db);
```

**参数说明**  
- `taos`: 有效的连接句柄
- `db`: 数据库名称

**返回值**  
成功返回0，失败返回非0值。

**使用示例**  
```c
if (taos_select_db(taos, "demo") != 0) {
    printf("选择数据库失败\n");
    return -1;
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L274-L274)

## 结果集处理

结果集处理是查询操作的重要组成部分，包括数据获取、元信息读取和资源释放。

### taos_fetch_row

**功能描述**  
从结果集中获取下一行数据。

**函数原型**  
```c
TAOS_ROW taos_fetch_row(TAOS_RES *res);
```

**参数说明**  
- `res`: 有效的结果集句柄

**返回值**  
成功时返回指向行数据的指针，无更多数据或出错时返回NULL。

**线程安全**  
该函数不是线程安全的，同一结果集不应在多个线程中同时处理。

**使用示例**  
```c
TAOS_ROW row;
int num_fields = taos_field_count(result);
TAOS_FIELD *fields = taos_fetch_fields(result);

while ((row = taos_fetch_row(result))) {
    char temp[1024] = {0};
    taos_print_row(temp, row, fields, num_fields);
    printf("%s\n", temp);
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L268-L268)
- [demo.c](file://examples/c/demo.c#L74-L132)

### taos_fetch_fields

**功能描述**  
获取结果集的字段元信息。

**函数原型**  
```c
TAOS_FIELD *taos_fetch_fields(TAOS_RES *res);
```

**参数说明**  
- `res`: 有效的结果集句柄

**返回值**  
指向TAOS_FIELD数组的指针，包含每个字段的名称、类型和大小信息。

**使用示例**  
```c
TAOS_FIELD *fields = taos_fetch_fields(result);
for (int i = 0; i < taos_field_count(result); i++) {
    printf("字段%d: %s, 类型: %d, 大小: %d\n", 
           i, fields[i].name, fields[i].type, fields[i].bytes);
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L277-L277)

### taos_free_result

**功能描述**  
释放结果集占用的资源。

**函数原型**  
```c
void taos_free_result(TAOS_RES *res);
```

**参数说明**  
- `res`: 要释放的结果集句柄

**使用说明**  
每次查询后都必须调用此函数释放资源，否则会导致内存泄漏。

**Section sources**
- [taos.h](file://include/client/taos.h#L271-L271)

## 预处理语句

预处理语句提供了更高效、更安全的SQL执行方式，特别适合批量数据操作。

### taos_stmt_init

**功能描述**  
初始化预处理语句对象。

**函数原型**  
```c
TAOS_STMT *taos_stmt_init(TAOS *taos);
```

**参数说明**  
- `taos`: 有效的连接句柄

**返回值**  
成功时返回TAOS_STMT句柄，失败时返回NULL。

**使用示例**  
```c
TAOS_STMT *stmt = taos_stmt_init(taos);
if (stmt == NULL) {
    printf("预处理语句初始化失败\n");
    return -1;
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L198-L198)
- [prepare.c](file://examples/c/prepare.c#L10-L233)

### taos_stmt_prepare

**功能描述**  
准备SQL语句模板。

**函数原型**  
```c
int taos_stmt_prepare(TAOS_STMT *stmt, const char *sql, unsigned long length);
```

**参数说明**  
- `stmt`: 预处理语句句柄
- `sql`: SQL语句模板，可包含?占位符
- `length`: SQL语句长度，0表示自动计算

**返回值**  
成功返回0，失败返回错误码。

**使用示例**  
```c
const char *sql = "INSERT INTO m1 VALUES(?,?,?,?,?,?,?,?,?,?,?)";
if (taos_stmt_prepare(stmt, sql, 0) != 0) {
    printf("准备语句失败\n");
    taos_stmt_close(stmt);
    return -1;
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L204-L204)

### taos_stmt_bind_param

**功能描述**  
绑定参数值到预处理语句。

**函数原型**  
```c
int taos_stmt_bind_param(TAOS_STMT *stmt, TAOS_MULTI_BIND *bind);
```

**参数说明**  
- `stmt`: 预处理语句句柄
- `bind`: 参数绑定数组

**返回值**  
成功返回0，失败返回错误码。

**使用示例**  
```c
TAOS_MULTI_BIND params[11];
// ... 配置参数绑定 ...
taos_stmt_bind_param(stmt, params);
taos_stmt_add_batch(stmt);
```

**Section sources**
- [taos.h](file://include/client/taos.h#L213-L213)

### taos_stmt_execute

**功能描述**  
执行预处理语句。

**函数原型**  
```c
int taos_stmt_execute(TAOS_STMT *stmt);
```

**参数说明**  
- `stmt`: 预处理语句句柄

**返回值**  
成功返回0，失败返回错误码。

**使用示例**  
```c
if (taos_stmt_execute(stmt) != 0) {
    printf("执行语句失败\n");
    exit(1);
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L218-L218)

## 异步API

异步API提供了非阻塞的数据操作方式，适合高并发场景。

### taos_query_a

**功能描述**  
异步执行SQL查询。

**函数原型**  
```c
void taos_query_a(TAOS *taos, const char *sql, __taos_async_fn_t fp, void *param);
```

**参数说明**  
- `taos`: 连接句柄
- `sql`: SQL语句
- `fp`: 回调函数指针
- `param`: 传递给回调函数的参数

**使用示例**  
```c
void query_callback(void *param, TAOS_RES *res, int code) {
    if (code != 0) {
        printf("查询失败，代码：%d\n", code);
    } else {
        printf("查询成功\n");
    }
    taos_free_result(res);
}

taos_query_a(taos, "SELECT * FROM m1", query_callback, NULL);
```

**Section sources**
- [taos.h](file://include/client/taos.h#L295-L295)
- [asyncdemo.c](file://examples/c/asyncdemo.c#L78-L193)

### taos_fetch_rows_a

**功能描述**  
异步获取结果集行数据。

**函数原型**  
```c
void taos_fetch_rows_a(TAOS_RES *res, __taos_async_fn_t fp, void *param);
```

**参数说明**  
- `res`: 结果集句柄
- `fp`: 回调函数
- `param`: 用户参数

**使用示例**  
```c
void fetch_callback(void *param, TAOS_RES *res, int code) {
    if (code > 0) {
        TAOS_ROW row;
        while ((row = taos_fetch_row(res))) {
            // 处理行数据
        }
    }
    taos_free_result(res);
}
```

**Section sources**
- [taos.h](file://include/client/taos.h#L298-L298)

## 无模式写入

无模式写入API支持直接写入行协议数据，无需预先定义表结构。

### taos_schemaless_insert

**功能描述**  
执行无模式数据插入。

**函数原型**  
```c
TAOS_RES *taos_schemaless_insert(TAOS *taos, char *lines[], int numLines, int protocol, int precision);
```

**参数说明**  
- `taos`: 连接句柄
- `lines`: 数据行数组
- `numLines`: 行数
- `protocol`: 协议类型（TSDB_SML_LINE_PROTOCOL等）
- `precision`: 时间戳精度

**返回值**  
结果集句柄，用于检查操作结果。

**使用示例**  
```c
char *lines[] = {
    "sta0,t0=true,t1=127i8,c0=true,c1=127i8 1546300800000",
    "sta1,t0=false,t1=128i8,c0=false,c1=128i8 1546300800001"
};
TAOS_RES *res = taos_schemaless_insert(taos, lines, 2, 
                                      TSDB_SML_LINE_PROTOCOL, 
                                      TSDB_SML_TIMESTAMP_MILLI_SECONDS);
```

**Section sources**
- [taos.h](file://include/client/taos.h#L335-L335)
- [schemaless.c](file://examples/c/schemaless.c#L20-L72)

## 数据生命周期示例

以下示例展示了完整的数据操作生命周期，包括连接、查询、结果处理和资源释放。

```c
int main(int argc, char *argv[]) {
    // 1. 建立连接
    TAOS *taos = taos_connect(argv[1], "root", "taosdata", NULL, 0);
    if (taos == NULL) {
        printf("连接失败：%s\n", taos_errstr(NULL));
        exit(1);
    }

    // 2. 执行查询
    TAOS_RES *result = taos_query(taos, "SELECT * FROM m1");
    if (taos_errno(result) != 0) {
        printf("查询失败：%s\n", taos_errstr(result));
        taos_free_result(result);
        taos_close(taos);
        exit(1);
    }

    // 3. 处理结果
    TAOS_ROW row;
    TAOS_FIELD *fields = taos_fetch_fields(result);
    int num_fields = taos_num_fields(result);
    
    while ((row = taos_fetch_row(result))) {
        char temp[256] = {0};
        taos_print_row(temp, row, fields, num_fields);
        printf("%s\n", temp);
    }

    // 4. 释放资源
    taos_free_result(result);
    taos_close(taos);
    taos_cleanup();
    
    return 0;
}
```

**错误处理**  
所有API调用都应检查返回值和错误状态。使用`taos_errno()`获取错误码，`taos_errstr()`获取错误信息。

**资源管理**  
- 每个`taos_query`调用后必须调用`taos_free_result`
- 每个`taos_connect`调用后必须调用`taos_close`
- 程序结束时调用`taos_cleanup`释放全局资源

**Section sources**
- [demo.c](file://examples/c/demo.c#L54-L132)
- [asyncdemo.c](file://examples/c/asyncdemo.c#L78-L193)
- [prepare.c](file://examples/c/prepare.c#L10-L233)