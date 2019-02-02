# 1 头文件与API设计
## 1.1 数据类型设计
通过JSON格式我们知道，JSON一共7中数据类型，用一个枚举类型表示
```c
typedef enum { LEPT_NULL, LEPT_FALSE, LEPT_TRUE, LEPT_NUMBER, LEPT_STRING, LEPT_ARRAY, LEPT_OBJECT } lept_type;
```
## 1.2 数据结构设计
由于 chapter 1 只解析 null true false三种类型，故结构体只需要存储一个lept_type
```c
typedef struct {
    lept_type type;
}lept_value;
```
## 1.3 函数API设计
### 1.3.1 解析JSON函数
解析JSON函数应该是如下的样子
```c
int lept_parse(lept_value* v, const char* json);
```
解析是否成功，具体可以分为以下四种状态，我们用枚举来表示
```c
enum {
    LEPT_PARSE_OK = 0,
    LEPT_PARSE_EXPECT_VALUE,
    LEPT_PARSE_INVALID_VALUE,
    LEPT_PARSE_ROOT_NOT_SINGULAR
};
```
### 1.3.2 访问结果的函数
```c
lept_type lept_get_type(const lept_value* v);
```
# 2 单元测试
极简的测试框架
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "leptjson.h"

static int main_ret = 0;
static int test_count = 0;
static int test_pass = 0;

#define EXPECT_EQ_BASE(equality, expect, actual, format) \
    do {\
        test_count++;\
        if (equality)\
            test_pass++;\
        else {\
            fprintf(stderr, "%s:%d: expect: " format " actual: " format "\n", __FILE__, __LINE__, expect, actual);\
            main_ret = 1;\
        }\
    } while(0)

#define EXPECT_EQ_INT(expect, actual) EXPECT_EQ_BASE((expect) == (actual), expect, actual, "%d")

static void test_parse_null() {
    lept_value v;
    v.type = LEPT_TRUE;
    EXPECT_EQ_INT(LEPT_PARSE_OK, lept_parse(&v, "null"));
    EXPECT_EQ_INT(LEPT_NULL, lept_get_type(&v));
}

/* ... */

static void test_parse() {
    test_parse_null();
    /* ... */
}

int main() {
    test_parse();
    printf("%d/%d (%3.2f%%) passed\n", test_pass, test_count, test_pass * 100.0 / test_count);
    return main_ret;
}
```
# 3 具体实现
## 3.1 lept_context结构体
为了减少传递参数，我们创建一个结构体
```c
typedef struct {
    const char* json;
}lept_context;
```
## 3.2 lept_parse的实现
首先，类型前边的空白字符不应该影响我们判断，我们首先要找到不是空白字符的地方，所以要实现一个处理空白字符的函数 
```c
static void lept_parse_whitespace(lept_context* c){
    const char *p = c->json;
    while(*p == ' ' || *p == '\t' || *p == '\n' || *p == '\r')
        p++;
    c->json = p;
}
```
找到第一个非空白符后，我们要判断是否属于null，true，false，‘\0’以及这几种都不是。综上，编写以下函数处理这五种情况
```c
static int lept_parse_value(lept_context* c, lept_value* v){
    switch (*c->json){
        case 'n':
            return lept_parse_null(c, v);
        case 't':
            return lept_parse_true(c, v);
        case 'f':
            return lept_parse_false(c, v);
        case '\0':
            return LEPT_PARSE_EXPECT_VALUE;
        default:
            return LEPT_PARSE_INVALID_VALUE;
    }
}
```
return lept_parse_null(c, v);
return lept_parse_true(c, v);
return lept_parse_false(c, v);
实现是相似的，
```c
static int lept_parse_null(lept_context* c, lept_value* v){
    EXPECT(c, 'n');
    if(c->json[0] != 'u' || c->json[1] != 'l' || c->json[2] != 'l')
        return LEPT_PARSE_INVALID_VALUE;
    c->json += 3;
    v->type = LEPT_NULL;
    return LEPT_PARSE_OK;
}
```
最后，解析函数实现如下
```c
int lept_parse(lept_value* v, const char* json){
    lept_context c;
    int ret;
    assert(v != NULL);
    c.json = json;
    v->type = LEPT_NULL;
    lept_parse_whitespace(&c);
    if((ret = lept_parse_value(&c, v)) == LEPT_PARSE_OK){
        lept_parse_whitespace(&c);
        if(*c.json != '\0')
            ret = LEPT_PARSE_ROOT_NOT_SINGULAR;
    }

    return ret;
}
```
## 3.2 lept_get_type实现
```c
lept_type lept_get_type(const lept_value* v){
    assert(v != NULL);
    return v->type;
}
```
