## 说明
基于 `github.com/marmotedu/errors` 包，只针对`error code`做优化:
- WithMessage 支持对error_code的多次描述，丰富提示信息
- 支持error_code的完整调用栈输出

比如，执行`fmt.Printf("%#+v", err)`语句，得到如下输出:

```
[
  {
    "caller":"#1 /home/f2/github/sample-code/examples/main.go:74 (main.getUser)",
    "code":100301,
    "error":"error change err_code",
    "message":"Encoding failed due to an error with the data: json miss required filed: 'name'"
  },
  {
    "caller":"#0 /home/f2/github/sample-code/examples/main.go:86 (main.queryDatabase)",
    "code":100101,
    "error":"connect failed",
    "message":"Database error"
  },
  {"callstack":"
    /home/f2/github/sample-code/examples/main.go:86 (queryDatabase);
    /home/f2/github/sample-code/examples/main.go:72 (getUser);
    /home/f2/github/sample-code/examples/main.go:63 (findUser);
    /home/f2/github/sample-code/examples/main.go:53 (bindUser);
    /home/f2/github/sample-code/examples/main.go:13 (main);
    /usr/local/go/src/runtime/proc.go:250 (main);
    /usr/local/go/src/runtime/asm_amd64.s:1594 (goexit);"
  }
]
```

从这个日志输出，我们可以得到以下几个信息：
- `"caller": "#1` 说明这个错误经历了2次error包函数的封装, 第后一次(#1)在`main.go:74 (main.getUser)`, 最开始是(#0)在`main.go:86 (main.queryDatabase)`
- 在`callstack`中看到，最原始的错误发生在 /home/f2/github/sample-code/examples/main.go 文件的 86行 的函数(queryDatabase)


## 错误描述规范

错误描述包括：对外的错误描述(message/code)和对内的错误描述(error)两部分。

### 对外的错误描述
> 对外错误信息是直接暴露给用户的，不能包含敏感信息

- 对外暴露的错误，统一大写开头，结尾不要加`.`
- 对外暴露的错误，要简洁，并能准确说明问题
- 对外暴露的错误说明，应该是 `该怎么做` 而不是 `哪里错了`
- 告诉用户他们可以做什么，而不是告诉他们不能做什么。
- 当声明一个需求时，用 must 而不是 should。例如，must be greater than 0、must match regex '[a-z]+'。
- 当声明一个格式不对时，用 must not。例如，must not contain。
- 当声明一个动作时用 may not。例如，may not be specified when otherField is empty、only name may be specified。
- 引用文字字符串值时，请在单引号中指示文字。例如，ust not contain '..'。
- 当引用另一个字段名称时，请在反引号中指定该名称。例如，must be greater than request。
- 指定不等时，请使用单词而不是符号。例如，must be less than 256、must be greater than or equal to 0 (不要用 larger than、bigger than、more than、higher than)。
- 指定数字范围时，请尽可能使用包含范围。
- 建议 Go 1.13 以上，error 生成方式为 fmt.Errorf("module xxx: %w", err)。
- 错误描述用小写字母开头，结尾不要加标点符号。

### 对内的错误描述
> 对内错误信息是给开发者用于定位排查的，是最为原始的错误，可能包含敏感信息

## 错误记录规范
当错误发生时，调用log包打印错误，通过err的callstack信息，能够定位到错误发生的位置。当使用这种方式来打印日志时，需要中遵循以下规范：

- 在错误`产生`的原始位置调用日志，log错误信息;
- 其它位置`不需要`重复记录err, 但可以记录相关入参和trace_id, 然后return err;
- 错误使用`fmt.Printf("%#+v", err)`，可记录完整的堆栈;
- 错误使用`err.Error()`或`fmt.Sprintf("%v", err)`，获取对外错误描述，返回给客户端;
- 错误使用`errors.Code(err)`，获取对外错误码，返回给客户端;
- 错误使用`errors.HTTPStatus(err)`，获取此次HTTP响应的状态码;


### 常规使用案例
- 当调用第三方库的函数报错，立即使用`errors.Wrap()`函数进行封装,并log。比如：
```go
if err := mqtt.Connect(); err != nil {
    err = errors.Wrap(err, code.ErrMqttConnect) // 返回一个新的err，log到日志
    log.Errorf("mqtt.Connect, error: %#+v", err)
    return err
}
```

- 当解析客户端请求体不符合规范，需要返回错误，用`errors.New()`。比如：
```go
if err := json.Unmarshal(req.Body, &param); err != nil {
    newerr := errors.New(code.ErrBadRequest, "ErrBadRequest") // 构造一个errors的类型
    errors.WithMessage(newerr, err.Error()) // 将具体的json错误，加入到对外错误的描述
    log.Infof("xxx bad request, info: %#+v", newerr)
    return newerr
}
```

- 当已知是errors的类型，需要转换为业务类型错误，用`errors.Spawn()`。比如
```go
if err := dbClient.QueryStaffById(ctx, "staffId"); err != nil {
    if errors.IsCode(err, code.ErrNoRow) { // db没有报错，只是没有该条数据,封装为ErrNoRow
        newerr := errors.Spawn(err, code.ErrUserNotFound) // 将原始err作为新错误的cause，方便定位根因，但根据业务逻辑，转换对外错误为UserNotFound
        errors.WithMessage(newerr, "staffId: xxx not exists") // 将具体的业务数据描述，反馈给前端
        log.Infof("user not found: trace_id: %s, staffId: %s", ctx.traceId, staffId) // 不需要重复log错误，只需记录相关业务参数，特别是trace_id
        return newerr // 返回新err
    }

    // 与业务无关的数据库错误, 无需再次log，直接返回
    return err
}
```

该 errors 包使用，可参考：[zhongjin616/sample-code](https://github.com/zhongjin616/sample-code/blob/master/README.md)
