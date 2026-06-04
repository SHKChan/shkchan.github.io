---
title: Python语言规范和风格规范
author: SHKChan
date: 2022-09-15
category: Code_Convention
layout: post
---
参考[Google开源项目风格指南](https://zh-google-styleguide.readthedocs.io/en/latest/contents/),规范一下Python编程习惯,在这里记录一下本人认为较为重要的点,便于自己查阅.

## [Python语言规范](https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/python_language_rules/#)

- `导入:仅对包和模块使用导入,而非函数或类,'typing'模块除外`
- `Lambda函数:lambda input : output.和条件表达式:x = 1 if cond else 2.适用于单行函数`
- `尽可能使用隐式False.在Python中0,None,[],{}和""都被认为是False`
- `使用代码类型注释以提供代码可读性和可维护性,eg:def func(a: int) - >List[int] or a: SomeType = some_function()`

## [Python风格规范](https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/python_style_rules/#)

- 命名规范

  ```
  模块名写法(文件名): module_name
  包名写法: package_name
  类名: ClassName
  方法名: method_name
  异常名: ExceptionName
  函数名: function_nam
  全局常量名: GLOBAL_CONSTANT_NAME
  全局变量名: global_var_name
  实例名: instance_var_name
  函数参数名: function_parameter_name
  局部变量名: local_var_name
  函数名,变量名和文件名应该是描述性的,尽量避免缩写,特别要避免使用非项目人员不清楚难以理解的缩写,不要通过删除单词中的字母来进行缩写. 始终使用 .py 作为文件后缀名,不要用破折号.
  ```
  下划线开头：
  单下划线 `_`开头意味着**受保护**的内部变量/函数,不会被 `import`导出,
  双下划线 `__`开头意味着类的**私有**变量或方法.
  **内部**意味着模块内,类内私有/保护变量/方法.
- `行长度:不超过80个字符.不推荐使用\连接行,可利用(),{}和[]中的行隐式连接特点连接行`
- `括号:宁缺毋滥地使用括号.不要再返回或者条件语句中使用括号,但是元组中使用括号是可以的`
- `缩进:用4个空格来缩进代码,绝对不要使用tab`
- `仅当'],)和}'与末尾元素不同一行时,额外添加,以便指示序列格式化为每行一项`
- `顶级定义中缠绵空两行,方法定义之间空一行`
- `程序的main文件应该以 #!/usr/bin/python3 开始.`
- `类:如果怕一个类不继承自其他类,就显示地从object继承.嵌套类也一样.`
- `字符串:避免在循环中用+和+=来累加字符串.可用list.append()以及''.joint(list)代替.`
- `TODO注释的格式:# TODO(name@email): what you want to do`
- `导入格式:每个导入应该独占一行, typing的导入除外.导入顺序应为标准库 第三方库 本地库,内部按照字母表顺序排列.`
- `主功能应该放在一个main()函数中.并以if __name__ == '__main__':进行保护`
- 函数长度:功能尽量集中,简单,小巧.尽量不超过40行,否则应考虑拆分.
