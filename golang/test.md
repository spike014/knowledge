## 文件规范

单元测试的⽂件的 `package` 名称可以和要测试的 `package` 名称相同，也可以以此 `package` 加 `_test` 作为包名。⽐如要测试 `package abc` , 则单元测试⽂件中的包名可以是 `package abc_test` ，这样的好处是单元测试函数只能访问要测试的公开的函数、结构和变量，更贴近使⽤这个包的场景。 `Go` 标准库中这两种⽅法都有使⽤。

## 一些实用的库

- [github.com/rakyll/gotest](https://github.com/rakyll/gotest) 用于支持色彩打印单元测试结果
- [github.com/agiledragon/gomonkey](https://github.com/agiledragon/gomonkey) 运行时修改函数，方法，接口的实现
- [github.com/nikolaydubina/go-cover-treemap](https://github.com/nikolaydubina/go-cover-treemap) 图形化输出测试覆盖率

---

> 参考资料：
> - go_test_workshop：https://github.com/smallnest/go_test_workshop