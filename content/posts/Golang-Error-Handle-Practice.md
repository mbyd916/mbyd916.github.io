---
title: "Golang 错误处理实践"
date: 2021-06-05T21:19:16+08:00
draft: false

categories:
- 技巧
- Golang
tags:
- 错误处理  
keywords:
- Golang 错误处理 
---


任何语言，错误处理都是至关重要的，开发人员只有学会正确地处理错误，才可能写出健壮的程序。本文主要介绍使用 Golang 这门语言在错误处理方面的一些实践，首先来看一下我们可能遇到的一些问题。

# 问题

Go 代码中常见的错误处理片段，被不少人诟病，可能处理业务逻辑的核心代码没几行，类似语句却写了一大堆 :-(

```
if err != nil {
    return err
}
```

或者（当然，还有这样的，仅抛给上层调用方哪儿够，得自己也打印一份日志 :-) ）

```
if err != nil {
    logs.CtxError(ctx, "failed to xxx: %s", err)
    return err
}
```

错误处理是很重要的，Go 语言鼓励开发人员当可能发生错误时，去明确地检查错误，这可能使得代码很冗长，不过本文介绍的一些方法可以简化重复的错误处理工作。

（吐槽时间）
(balabalabala...)

总结一下：

1. 缺少错误上下文和堆栈信息；
2. 冗余，代码冗长，主逻辑割裂；
3. 分层开发，日志泛滥；

# 是什么

接下来先了解一下 Go 的 `error` 类型，然后我们再想办法逐一解决上述问题。Go 社区流行很多“谚语”，其中有一句和“错误”有关的，来自 Go 语言合作者 Rob Pike 大神。

> *Errors are values.*
> 
> by [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike)

`error` 是内置类型，定义为一个 `interface`。

```
type error interface {
    Error() string
}
```

标准库 `errors` 包提供了一个默认错误类型 `errorString`，借助一个字符串字段保存错误信息，开发者可以通过 `errors.New("xyz")` 创建一个“标准”错误。

```
// New returns an error that formats as the given text.
// Each call to New returns a distinct error value even if the text is identical.
func New(text string) error {
        return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
        s string
}

func (e *errorString) Error() string {
        return e.s
}
```

此外，`fmt` 包还提供了一个函数 `Errorf`，用来创建格式化“错误”，允许开发人员添加一些上下文信息，这种方法更常用。

当然，开发者也可以自定义错误类型，通过实现 `error` 接口，为程序“错误”提供更详细的**上下文信息，** 如标准库的 `PathError`：

```
// PathError records an error and the operation and
// file path that caused it.
type PathError struct {
    Op string    // "open", "unlink", etc.
    Path string  // The associated file.
    Err error    // Returned by the system call.
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

调用出错（如打开一个不存在的文件）时，可能看到的提示：

```
open /not_exists: no such file or directory
```

此外，还可以通过类型断言获取具体错误类型，进行检查并处理。

```
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }

    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // Recover some space.
        continue
    }

    return
}
```

# 为什么

我们再来简单看下 Go 语言的前辈们是如何处理错误的？一般有两种方法：

-   返回值检查
-   异常机制

分别以 C 和 Java 两种语言将“字符串转为整型”为例进行说明：

```
// 当返回值=0，无法区分转换是否成功
int atoi(const char *str);

// 通过errno、返回值以及出参endptr判断
long int strtol(const char *nptr, char **endptr, int base);

// errno:
// EINVAL
// ERANGE


 // 
    result = strtol(value, &eptr, 10);
    if (result == 0)
    {
        if (errno == EINVAL)
        {
            printf("Conversion error occurred: %d\n", errno);
            exit(0);
        }
    }

    if (result == LONG_MIN || result == LONG_MAX)
    {
        if (errno == ERANGE)
            printf("The value provided was out of range\n");
    }

    //...
```

C 只能有一个返回值

-   无法区分正常和异常返回；
-   引入“出参”形式，函数声明变得更复杂；

```
// 抛异常
public static int parseInt(String s, int radix)
                    throws NumberFormatException

                    

    try {
            int result = Integer.parseInt("123abcd", 10);
            System.out.println(result);
            // other logic
        } catch (Exception e) {
            e.printStackTrace();

        }


 // java.lang.NumberFormatException: For input string: "123abcd"
 //       at java.base/java.lang.NumberFormatException.forInputString(NumberFormatException.java:68)
 //       at java.base/java.lang.Integer.parseInt(Integer.java:652)
 //       at Hello.main(Hello.java:6)
```

Java 异常机制 try-catch-finally

-   错误处理和返回值完全分开；
-   正常代码与错误处理逻辑分开，提高可读性；
-   “受检”异常不能被忽略，需要显示声明并处理。

```
func Atoi(s string) (int, error)

func ParseInt(s string, base int, bitSize int) (i int64, err error)



    result, err := strconv.ParseInt("123abcd", 10, 64)
    if err != nil {
        fmt.Println(err)
        return
    }

    // strconv.ParseInt: parsing "123abcd": invalid syntax
```

Go 错误处理本质上也是通过**检查返回值**实现的，不过：

-   Go 支持多值返回，可以将业务返回值和错误返回值区分开；
-   可以用 `_` 显式忽略错误；
-   并且定义了内置的 `error` 类型，方便开发者进行扩展；

使用“异常”有不少优势，为啥 Go 不这么做呢？

> 我们认为，将“异常”处理耦合到控制结构（如 try-catch-finally）会导致代码混乱。它还倾向于鼓励程序员将太多的常见错误（例如，无法打开文件）标记为“异常”的。
>
> From [Go 官方 FAQ](https://golang.org/doc/faq#exceptions)

[Why Go Gets Exceptions Right?](https://dave.cheney.net/2012/01/18/why-go-gets-exceptions-right)


那么 Go 认为的“异常”情况是怎样的呢？

当 Go 程序出现不可恢复的运行时错误，会抛出 `panic`（直译“恐慌”），如：

-   索引越界；
-   类型断言失败等；

**注**：一般不建议使用 `panic` 作为 Go 的错误处理方法。

不过初始化程序时可以使用，如程序启动依赖的必要条件不能满足时：

```
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

# 怎么做

接下来我们来看下如何解决一开始提到的问题：

### 1. 常用包

-   Dave [pkg/errors](https://pkg.go.dev/github.com/pkg/errors)
-   Go 1.3 或以上标准库 [errors](https://golang.org/pkg/errors/)和 [fmt](https://pkg.go.dev/fmt#Errorf)


![error pkgs](https://raw.githubusercontent.com/mbyd916/blog-hg/fd5d27fef45a761a0ef24a8032d671e9db58709b/static/images/error_pkgs.png)

推荐第1种，能记录错误堆栈信息；

![errors.Wrap](https://raw.githubusercontent.com/mbyd916/blog-hg/fd5d27fef45a761a0ef24a8032d671e9db58709b/static/images/errors_wrap.png)

![errors.Cause](https://raw.githubusercontent.com/mbyd916/blog-hg/fd5d27fef45a761a0ef24a8032d671e9db58709b/static/images/errors_cause.png)

标准库 `errors` 使用举例：

```
func main() {
        e1 := errors.New("1st error")
        e2 := fmt.Errorf("2nd: %w", e1)
        e3 := fmt.Errorf("3rd: %w", e2)
        e4 := fmt.Errorf("4th: %w", e3)

        fmt.Println(e1)
        fmt.Println(e2)
        fmt.Println(e3)
        fmt.Println(e4)
        fmt.Println("====================================")

        e5 := errors.Unwrap(e4)
        fmt.Println("e5 == e3?", e5 == e3)
        fmt.Println("e5 Is e3?", errors.Is(e5, e3))

        fmt.Println("====================================")
        fmt.Println("e5 == e1?", e5 == e1)
        fmt.Println("e5 Is e1?", errors.Is(e5, e1))

  

// output:      
// 1st error
// 2nd: 1st error
// 3rd: 2nd: 1st error
// 4th: 3rd: 2nd: 1st error
// ====================================
// e5 == e3? true
// e5 Is e3? true
// ====================================
// e5 == e1? false
// e5 Is e1? true
}
```

```
type MyError struct {
        err string
}

func (e *MyError) Error() string {
        return e.err
}

func main() {
        e1 := &MyError{"1st error"}
        e2 := fmt.Errorf("2nd: %w", e1)
        e3 := fmt.Errorf("3rd: %w", e2)
        e4 := fmt.Errorf("4th: %w", e3)

        fmt.Println(e1)
        fmt.Println(e2)
        fmt.Println(e3)
        fmt.Println(e4)
        
        fmt.Println("====================================")
        var err5 *MyError
        fmt.Println(errors.As(e4, &err5))
        fmt.Println(err5)


// output: 
// 1st error
// 2nd: 1st error
// 3rd: 2nd: 1st error
// 4th: 3rd: 2nd: 1st error
// ====================================
// true
// 1st error
}
```

### 2. 更优雅地处理错误

示例 1：统计文件行数

```
func CountLines(r io.Reader) (int, error) {
        var (
                br    = bufio.NewReader(r)
                lines int
                err   error
        )

        for {
                 _, err = br.ReadString('\n')
                 lines++
                 if err != nil {
                     break
                 }
        }

        if err != io.EOF {
                return 0, err
        }

        return lines, nil
 }
```
(**注**：仅作错误处理说明用，以上代码存在 bug，如空文件，统计行数为 1)

我们也许可以从标准库 `bufio` 包 [Scanner](https://golang.org/pkg/bufio/#Scanner) 类型找找灵感，`Scan` 方法并未直接返回 `error` 类型，而是返回了一个 `boolean` 类型，还提供了一个 `Err` 方法返回发生的错误。

```
func CountLines(r io.Reader) (int, error) {
        sc := bufio.NewScanner(r)
        lines := 0

        for sc.Scan() {
                lines++
        }

        return lines, sc.Err()
}
```

将错误处理与主流程分开，提升代码可读性。

示例 2:

```
type Header struct {
        Key, Value string
}

type Status struct {
        Code   int
        Reason string
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
        _, err := fmt.Fprintf(w, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)
        if err != nil {
                return err
        }

        for _, h := range headers {
                _, err := fmt.Fprintf(w, "%s: %s\r\n", h.Key, h.Value)
                if err != nil {
                        return err
                }
        }

        if _, err := fmt.Fprint(w, "\r\n"); err != nil {
                return err
        } 

        _, err = io.Copy(w, body) 

        return err
}
```

改造之后

```
type errWriter struct {
        io.Writer
        err error
}

func (e *errWriter) Write(buf []byte) (int, error) {
        if e.err != nil {
                return 0, e.err
        }

        var n int
        n, e.err = e.Writer.Write(buf)
        return n, nil
}

func WriteResponse(w io.Writer, st Status, headers []Header, body io.Reader) error {
        ew := &errWriter{Writer: w} 
        fmt.Fprintf(ew, "HTTP/1.1 %d %s\r\n", st.Code, st.Reason)

        for _, h := range headers {
               fmt.Fprintf(ew, "%s: %s\r\n", h.Key, h.Value)
        }

        fmt.Fprint(ew, "\r\n")
        io.Copy(ew, body)

        return ew.err
}
```

另外大家用到的 Gorm 框架，能进行链式调用，也采用了类似的实现思路。

```
// DB GORM DB definition
type DB struct {
   *Config
   Error        error
   RowsAffected int64
   Statement    *Statement
   clone        int
}

lead := &Lead{LockState: Unlocked}
db.WithContext(ctx).Table(tbl).
   Select("lock_state").
   Where("id=? AND lock_state=?", id, Locked).Updates(lead).Error
```

每一个方法都返回`*DB`类型，没有额外返回`error`。

### 3. 仅在最上层调用打日志

```
func main() {
   err := parseConf("not_exist.json")
   if err != nil {
      log.Printf("parse conf: %s", err)
      return
   }
}

func parseConf(name string) error {
   content, err := readFile(name)
   if err != nil {
      log.Printf("read file: %s", err)
      return err
   }

   // Parse JSON content.
   _ = content

   return nil
}

func readFile(name string) ([]byte, error) {
   f, err := os.Open(name)
   if err != nil {
      log.Printf("open file: %s", err)
      return nil, err
   }
   defer f.Close()

   buf := make([]byte, 0, 512)
   // Read the file.

   return buf, nil
}



// 2021/05/14 18:43:05 open file: open not_exist.json: no such file or directory
// 2021/05/14 18:43:05 read file: open not_exist.json: no such file or directory
// 2021/05/14 18:43:05 parse conf: open not_exist.json: no such file or directory
```

```
package main

import (
   "log"
   "os"

   "github.com/pkg/errors"
)

func main() {
   err := parseConf("not_exist.json")
   if err != nil {
      log.Printf("parse conf: %+v", err)
      return
   }
}

func parseConf(name string) error {
   content, err := readFile(name)
   if err != nil {
      return err
   }

   // Parse JSON content.
   _ = content

   return nil
}

func readFile(name string) ([]byte, error) {
   f, err := os.Open(name)
   if err != nil {
      return nil, errors.Wrap(err, "open file")
   }
   defer f.Close()

   buf := make([]byte, 0, 512)
   // Read the file.

   return buf, nil
}

// 2021/05/14 18:46:36 parse conf: open not_exist.json: no such file or directory
// open file
// main.readFile
//        /Users/marvel/Workspace/errors/test.go:32
// main.parseConf
//        /Users/marvel/Workspace/errors/test.go:19
// main.main
//        /Users/marvel/Workspace/errors/test.go:11
// runtime.main
//         /Users/marvel/sdk/go1.16/src/runtime/proc.go:225
// runtime.goexit
//        /Users/marvel/sdk/go1.16/src/runtime/asm_amd64.s:1371
```

HTTP 服务还可以利用日志中间件实现，在 API 最顶层打印日志。

### 4. 通过 panic 和 recover 进行错误处理

```
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()

    return regexp.doParse(str), nil
}
```

`recover` 只能在 `defer` 函数体内使用。

```
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```

**局限性**：只能用在同一个包内，将内部产生的 `panic` 转为 `error` 返回给调用者；而不能主动抛 `panic` 给调用者。

### 5. 并发场景下的错误处理

可以使用官方工具包 [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup) ，对 `sync.WaitGroup` 进行了巧妙的封装，简化了同步处理逻辑，同时允许并发执行的子任务将可能出现的错误返回给调用方。

```
func main() {
        g := new(errgroup.Group)
        var urls = []string{
                "http://www.golang.org/",
                "http://www.google.com/",
                "http://www.somestupidname.com/",
        }

        for _, url := range urls {
                // Launch a goroutine to fetch the URL.
                url := url // https://golang.org/doc/faq#closures_and_goroutines
                g.Go(func() error {
                        // Fetch the URL.
                        resp, err := http.Get(url)
                        if err == nil {
                               resp.Body.Close()
                        }

                        return err
                })
        }

        // Wait for all HTTP fetches to complete.
        if err := g.Wait(); err == nil {
                fmt.Println("Successfully fetched all URLs.")
       }
}
```
### 6. 一些规范

-   错误描述**小写**开头，加前缀（如包名），如 `image: unknown format`

-   自定义错误类型以 `Error` 结尾，变量以 `Err` 或 `err` 开头；

-   函数（或方法）最后一个返回值返回 `error` 类型，不要返回具体的错误类型；

-   自定义 `Error` 接口，包含其他判别具体错误的方法；

```
package net

type Error interface {
    error
    Timeout() bool   // Is the error a timeout?
    Temporary() bool // Is the error temporary?
}
```

### 7. 特别注意的坑

```
type MyError struct {
  code int
  msg string
}

func (e *MyError) Error() string {
  return fmt.Sprintf("code=%d, msg=%s", e.code, e.msg) 
}

var ErrBad = &MyError{code: 500, msg: "something bad occurs"}

// Bad
func Handle() error {
  var err *MyError = nil
  if bad() {
    err = ErrBad
  }

  return err
}



// Good
func Handle() error {
  if bad() {
    return ErrBad
  }

  return nil
}
```

# 参考

1.  https://golang.org/doc/effective_go#errors
1.  https://blog.golang.org/error-handling-and-go
1.  https://blog.golang.org/errors-are-values
1.  https://github.com/golang/go/wiki/Errors
1.  https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
1.  https://dave.cheney.net/2019/01/27/eliminate-error-handling-by-eliminating-errors
1.  https://blog.golang.org/go1.13-errors
1.  https://www.sohu.com/a/342949702_657921
1.  [Go语言中的错误处理（Error Handling in Go）](https://ethancai.github.io/2017/12/29/Error-Handling-in-Go/)
1.  https://dave.cheney.net/paste/gocon-spring-2016.pdf
1.  https://coolshell.cn/articles/21140.html
1.  https://blog.golang.org/defer-panic-and-recover
