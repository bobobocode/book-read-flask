# click包——快速创建命令行

Flask的命令行功能使用了第三方模块Click来实现.
使用Click的两个注解可以快速将一个普通函数编程命令行接口.
看一下Click官方文档中的例子, 非常简单便捷
```
import click

@click.command()
@click.option("--count", default=1, help="Number of greetings.")
@click.option("--name", prompt="Your name",
              help="The person to greet.")
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for _ in range(count):
        click.echo("Hello, %s!" % name)

if __name__ == '__main__':
    hello()
```
其中在hello函数上, 使用 @click.command() 装饰一个函数, 使之成为命令行接口;
使用 @click.option() 装饰函数, 为其添加命令行选项.
该脚本执行起来如下:
```
$ python hello.py --count=3
Your name: Click
Hello, Click!
Hello, Click!
Hello, Click!
```

--name参数设置了prompt, 所以当不指定的时候, 执行起来会提示用户输入.
```
$ python hello.py
Your name: Click
Hello Click!
```
自动生成--help支持 
```
$ python hello.py --help
Usage: hello.py [OPTIONS]
 
  Simple program that greets NAME for a total of COUNT times.
 
Options:
  --count INTEGER  Number of greetings.
  --name TEXT      The person to greet.
  --help           Show this message and exit.
```
可以加或者不加 = 来指定参数
```
$ python hello.py --count 3 --name Click
Hello Click!
Hello Click!
Hello Click!
 
$ python hello.py --count=3 --name=Click
Hello Click!
Hello Click!
Hello Click!
```

Flask用到的Click中一个特别重要的功能, 就是可任意嵌套命令行工具的概念.
也就是实现我们在命令行中经常用到的子命令.这个能力是通过Command类和Group类设计实现的.
还有MultiCommand
