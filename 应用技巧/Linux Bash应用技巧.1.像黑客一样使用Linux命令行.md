# Linux高效输入

参考文章

1. [像黑客一样使用Linux命令行](https://talk.linuxtoy.org/using-cli/#1)
    - 在线PPT
2. [学习T神的《像黑客一样使用Linux命令行》](https://zlotus.github.io/2014/07/07/using-cli-like-a-hacker/)

## 组合命令查询手册

|               命令引用                |               参数引用                |       参数修饰        |
| :-----------------------------------: | :-----------------------------------: | :-------------------: |
| ! [ ! \| [ ? ] 字符串 \| [ - ] 数字 ] | : [ ^ \| $ \| \* \| n \| n\* \| x-y ] | : [ h \| t \| r \| e] |

## 命令修改

- `^string` 或 `!:s/string/`: 删除上一条命令中第1个匹配到的指定的字符串`string`
- `^old^new` 或 `!:s/old/new`: 替换上一条命令中输错或输少的部分, 也是只匹配第1个
- `!:gs/old/` 或 `!:gs/old/new`: 将上一条命令中`old`部分全部删除或替换成`new`

------

关于命令修改, 要记住的是

- 一删
- 二换
- 三全变

## 1. 命令引用

- `!!`: 重复上一条命令
- `!his`: 执行最近的以`his`开头的命令
- `!?his: 执行最近的包含`his`字符串的命令(看来跟`ctrl+r`作用相同)
- `!n`: 执行第n个命令, 注意这个n不是倒数的, 而是按照`history`命令显示的序号来的
- `!-n`: 这个才是执行倒数第n个命令的(从1开始计数的哦), 即`!-1` == `!!`
- `!#`: 引用当前行, 比如`!#:1`是当前行的第1个参数, 一般都是作为后面的参数引用前面的

------

关于命令引用, 要记住的是

- !!
- ![?]字符串
- ![-]数字
- !#

## 2. 参数引用

- `!$`: 代表上一条命令的最后一个参数
- `!^`: 代表上一条命令的第一个参数(真的是参数, 也就是命令名不计算在内
- `!*`: 代表上一条命令的所有参数
- `!:n`: 代表上一条命令第n个参数
- `!:x-y`: 代表上一条命令从第x到第y个参数
- `!:n*`: 代表上一条命令从 n 开始到最后的参数

关于参数引用, 要记住的是

- n
- ^|$
- [n]*
- x-y

## 3. 参数修饰

- `:h`: 选取参数中的路径开头.(其实就是以`/`分割字符串)
- `:t`: 选取参数中的路径结尾
- `:r`: 选取参数中的文件名.(这个就是以`.`分割了)
- `:e`: 选取参数中的扩展名.(说了是以`.`分割了, 如果是扩展名是`.tar.gz`的话, 这个东西就只能得到`gz`...)
