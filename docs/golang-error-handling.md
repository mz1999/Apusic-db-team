# Golang error处理实践

Go的错误处理机制很简洁，使用`errors.New(text)`创建 `error`，方法的调用者一般按照如下模式处理：

```
if err != nil {
        return err
}
```

这样做最大的问题是`error`中没有保存方法调用栈等上下文信息，只能靠创建时传递的`string`参数来区分`error`，很难定位错误发生的具体位置。例如：

```
import (
	"fmt"
	"errors"
)

func f1() error {
	return f2()
}

func f2() error {
	return f3()
}

func f3() error {
	return errors.New("std error")
}

func main() {
	if err := f1(); err != nil {
		fmt.Printf("%+v", err)
	}
}
```

执行的输出为：

```
std error
```

在实际的程序中调用关系复杂，仅凭错误信息很难定位错误源头。TiDB 使用了[juju/errors](https://github.com/juju/errors)来记录调用栈：

```
import (
	"github.com/juju/errors"
	"fmt"
)

func jf1() error {
	err := jf2()
	if err != nil {
		return errors.Trace(err)
	}
	return nil
}

func jf2() error {
	err := jf3()
	if err != nil {
		return errors.Trace(err)
	}
	return nil
}

func jf3() error {
	return errors.New("juju error")
}

func main() {
	if err := jf1(); err != nil {
		fmt.Printf("%+v", err)
	}
}
```

这段代码的输出为：

```
github.com/mz1999/error/main.go:25: juju error
github.com/mz1999/error/main.go:19: 
github.com/mz1999/error/main.go:11: 
```

可以看到，如果想记录调用栈，每次都需要调用`errors.Trace`。这样做比较繁琐，而且每次`trace`时内部都会调用`runtime.Caller`，性能不佳。TiDB已经调研了新的第三方包[pkg/errors](https://github.com/pkg/errors)准备[替换掉juju/errors](https://github.com/pingcap/tidb/issues/7125)。

使用`pkg/errors`会简单很多，和标准库的`errors`一致，但可以记录调用栈信息：

```
import (
	"fmt"
	"github.com/pkg/errors"
)

func pf1() error {
	return pf2()
}

func pf2() error {
	return pf3()
}

func pf3() error {
	return errors.New("pkg error")
}

func main() {
	if err := pf1(); err != nil {
		fmt.Printf("%+v", err)
	}
}
```

这段代码的输出为：

```
pkg error
main.pf3
	/Users/mazhen/Documents/works/goworkspace/src/github.com/mz1999/error/main.go:17
main.pf2
	/Users/mazhen/Documents/works/goworkspace/src/github.com/mz1999/error/main.go:13
main.pf1
	/Users/mazhen/Documents/works/goworkspace/src/github.com/mz1999/error/main.go:9
main.main
	/Users/mazhen/Documents/works/goworkspace/src/github.com/mz1999/error/main.go:21
runtime.main
	/usr/local/go/src/runtime/proc.go:198
runtime.goexit
	/usr/local/go/src/runtime/asm_amd64.s:2361
```

这样看，使用`pkg/errors`替代标准库的`errors`就可以满足我们的需求。

另外，`pkg/errors`的作者还给了一些[最佳实践的建议](https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package)：

* 在你自己的代码中，在错误的发生点使用`errors.New` 或 `errors.Errorf` ：

```
func parseArgs(args []string) error {
        if len(args) < 3 {
                return errors.Errorf("not enough arguments, expected at least 3, got %d", len(args))
        }
        // ...
}
```

* 如果你接收到一个`error`，一般简单的直接返回：

```
if err != nil {
       return err
}
```

* 如果你是调用第三方的包或标准库时接收到`error`，使用 `errors.Wrap` or `errors.Wrapf` 包装这个`error`，它会记录在这个点的调用栈：

```
f, err := os.Open(path)
if err != nil {
        return errors.Wrapf(err, "failed to open %q", path)
}
```

* Always return errors to their caller rather than logging them throughout your program.

* 在程序的top level，或者是worker goroutine，使用 `%+v` 输出`error`的详细信息。

```
func main() {
        err := app.Run()
        if err != nil {
                fmt.Printf("FATAL: %+v\n", err)
                os.Exit(1)
        }
}
```

* 如果需要抛出包含MySQL错误码的内部错误，可以使用`errors.Wrap`包装，附带上报错位置的调用栈信息：

```
func pf3() error {
	return errors.Wrap(mysql.NewErr(mysql.ErrCantCreateTable, "tablename", 500), "")
}
```

这样，我们既拿到了完整调用栈，又可以使用`errors.Cause`获取MySQL的错误码等信息：

```
if err := pf1(); err != nil {
    fmt.Printf("%+v", err)

    var sqlError *mysql.SQLError
    if m, ok := errors.Cause(err).(*mysql.SQLError); ok {
    	sqlError = m
    } else {
    	sqlError = mysql.NewErrf(mysql.ErrUnknown, "%s", err.Error())
    }
    
    fmt.Printf("\nMySQL error code: %d, state: %s", sqlError.Code, sqlError.State)
}
```


