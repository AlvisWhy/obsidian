rwxr-xr-x
creator - user belong to a group -other people
read write executable 

chmod a+x
a(all) u(user) g(group) o(other people)

chmod u+x script.sh
chmod g-w file.txt
chmod 755 app.sh(rwxr-xr-x)

top
df -h (disk free, show load and availability for disks)
free -m (show load in RAM)
uptime
iostat (show disk IO state)

\`` `` or $() first execute and then get
$ : get the assigned value
if [. ]
then
elif []
then
fi

if only when skip outside the 'if else', 'fi' wiil be added.

```
read choice
case $choice
a) date;;
b) ls;;
c) who;;
d) uptime;;
*) echo invalid choice


for i in 1 2 3 4 5
do
echo "welcome $i"
done

count = 0
num = 10
while [ $count -lt 10 ]
do
	echo `expr $num - 1`
done
```

grep         -i(ignore case) 
		-o(only extract contained piece)
		-E (regex)
		-v(revert match)
		-c(count line num)
		-x(all line full match)

awk separate the line
awk -F, '{print $1, $2}'


DotFile

Dotfile

    ↓

文件名以 . 开头
ls

    ↓

默认不显示 dotfiles
*
    ↓

默认不匹配 dotfiles
ls -a
    ↓

显示所有文件，包括 dotfiles
.*
    ↓

glob 匹配以 . 开头的文件

> **Linux/Bash 把 `.` 开头的文件视为隐藏文件，普通的 `ls` 和 `*` glob 默认都会忽略它们；需要全部查看时使用 `ls -a`。**

要真正理解这条规则，关键是区分 ls -a 与 .* 在匹配范围上的差别，因为前者用于查看目录内容，后者会参与路径展开。

The single dot folder (`.`) is a special folder that represents the folder that you are in.

```shell
rename -n 's/(.*)/new$1/' *
```
'.' will not match the dotfiles.
".*"  can match the dotfiles and the file strcuture

the bash shell translates the variable into its value if it’s in double quotes, but does not if it’s in single quotes.
```shell
MYSENTENCE="A sentence with $MYSTRING in it"
echo $MYSENTENCE                               # Outputs the sentence with the variable dereferenced
MYSENTENCE='A sentence with $MYSTRING in it'
echo $MYSENTENCE                               # Outputs the sentence with the variable not dereferenced

```

ReadOnly Variable
```shell
readonly MYVAR = astring
```

Using export, all of the variables can be reached by any child process with $

```shell
env
//Outputs all exported variables

compgen -v
```
compgen
   ↓
completion generator
   ↓
告诉我“Tab 补全时可能有哪些东西”
-c → commands
-a → aliases
-b → builtins
-v → variables
-e → exported variables

Array In Bash
```shell
echo $BASH_VERSINFO 
echo $BASH_VERSINFO[0]   # Does not work
echo ${BASH_VERSINFO[0]} # Will work
echo ${BASH_VERSINFO}

```

Shell 的变量展开（parameter expansion）在单引号 `'...'` 中不会发生；在双引号 `"..."` 中会发生；`${...}` 是变量展开的语法形式。

Functions

```shell
function myfunc {
    echo $1
    echo $2
}
myfunc "Hello World"
myfunc Hello World

function myfunc {
    local myvar="Hi from inside the function"   # Creates a variable local to the function
    echo $myvar
}
myfunc
echo $myvar
local myvar="Will this work?"                   # Will not work
```


#### Functions, Builtins, Aliases and Programs[#](https://www.educative.io/courses/master-the-bash-shell/functions#Functions-Builtins-Aliases-and-Programs)

**Builtins** are commands that come ‘built in’ to the bash shell program. Normally you can’t easily tell the difference between a builtin, a program or a function, but after reading this you will be able to.

```shell
function cd() {
    echo 'No!'
}
//override
cd /tmp
builtin cd /tmp
cd -

//stop override
unset -f cd
cd /tmp
cd -
```

**Programs** are executable files. Commonly-used examples of these are programs such as `grep`, `sed`, `vi`, and so on

Aliases are strings that the shell takes and translates to whatever that string is aliased to.
```shell
alias cd=doesnotexist
alias
cd
unalias cd
cd /tmp
cd -
alias
```

#### Pipes and Redirects

- A pipe passes _standard output_ as the _standard input_ to another command
    
- A _redirect_ sends a channel of output to a file

0 → stdin  → 输入
1 → stdout → 正常输出
2 → stderr → 错误输出

```shell
command 2> err.txt

//Output both output and error output
command > output.txt 2> error.txt

//正常输出和错误输出都会去 stdout 当前去的地方
command 2>&1

//把 stdout 和 stderr 都写进同一个文件。
command > output.log 2>&1
//stdout → output.log
//stderr → output.log
```

`>>` 是 **追加重定向（append redirection）**。

意思是：

> 把命令的标准输出追加到 `logfile` 文件末尾，而不是覆盖原来的内容。

```shell
echo line1 > file3
cat file3
echo line2 > file3   # Overwrites file3, replacing its contents with 'line2'
cat file3
echo line3 >> file3  # Appends to file3
cat file3
```

Source

source
   ↓
在当前 shell 中执行文件
   ↓
变量、alias、function 等修改会留在当前 shell

$PATH
  ↓
Bash 查找可执行程序时使用的目录列表
看看 Bash 当前会去哪些目录找命令。路径之间用.间隔，如果需要运行当前目录需要把.加入当前目录

```shell
PATH=${PATH}:.
```

**Shebang（#!）** 是写在脚本第一行的一种特殊标记，用来告诉操作系统：
> **“这个脚本应该使用哪个解释器来执行。”**

例如最常见的：
#!/bin/bash


```
echo $(echo hello1 $(echo hello2))   # Nesting echoes
echo `echo hello1 `echo hello2``     # Nesting echoes with backticks fails
echo `echo hello1 \`echo hello2\``   # Nesting echoes with backticks and backslashes ok
```
#### TEST OPERATOR
[  ...  ]
 ↓
传统 test 命令

[[ ... ]]
 ↓
Bash 专门提供的、更强的条件判断

返回前一个判断的结果，0是true 1是false
```shell
echo $? 
```

-d → 是不是目录
-a → 两个条件是否 AND（但不推荐）/ 文件是否存在
-z → 字符串是不是空的, 空的是true
-n → non-zero length → 非空字符串

-d → directory
-f → file
-e → exists
-z → zero-length string
-n → non-zero-length string
&& → AND
|| → OR

| 操作  | 用法    | 含义           |
| --- | ----- | ------------ |
| 数字  | `-gt` | greater than |
| 数字  | `-lt` | less than    |
| 数字  | `-eq` | equal        |
| 字符串 | `==`  | 相等           |
| 字符串 | `!=`  | 不相等          |
| 字符串 | `<`   | 字典序更小        |
| 字符串 | `>`   | 字典序更大        |
用数字> <也会转化为字符比较

变量展开时，如果变量内容可能包含空格，通常使用 `"$variable"`。
if
```shell
for f in $(ls *txt)
do
    echo "File $f contains: $(cat $f)"
done
```
whlile
```shell
n = 0
while true
do 
	sleep 1
	((n++))
	if[[$n -eq 60]]
	then
		break
	fi
end
```
case
```shell
a=1                                        # Initialize 'a' variable
case "$a" in                               # Start case statement
1) echo 'a is 1'; echo 'ok1';;             # If a == 1, then output ok1
2) echo 'a is 2'; echo 'ok2';;             # If a == 2, then output ok2
*) echo 'a is unmatched'; echo 'failure';; # If nothing was matched, output 'failure'
esac                                       # End case statement
```
getopts
```shell
cat > case.sh << 'EOF'
#!/bin/bash
while getopts "ab:c" opt
do
    case "$opt" in
    a) echo '-a invoked';;
    b) echo "-b invoked with argument: ${OPTARG}";;
    c) echo '-c invoked';;
    esac
done
EOF
chmod +x case.sh
```

Exit codes
- `0` - ‘command successfully run’ (OK)
- `1` - Used when there is an error but no specific number reserved to indicate what it was (general error)
- `2` - Misuse of shell `builtin` command
- `126` - Permission problem or command is not executable
- `127` - No file found matching this command
- `128` - Invalid exit argument given (eg exit 1.76)

几个 Bash 特殊变量
```shell
$?   → 上一条命令的 exit status
$$   → 当前 shell 的 PID
$!   → 最近后台进程的 PID
$#   → 参数数量
$@   → 所有参数
$0   → 脚本名称
$1   → 第一个参数
```

Set
开启...模式
`-o` (on) and `+o` (off)
```shell
set -o posix
```

```shell
set -o errexit    = set -e
set -o xtrace     = set -x
set -o nounset
```
- The `errexit` option tells bash to exit the script if any command fails
- The `xtrace` option outputs each command as it is being run. This is really useful for seeing what command was actually run if (for example) you are using variables within your commands. It also helps you see the order in which commands are being run
- The `nounset` option gets bash to throw an error if a variable is not set when it is referred to

pipefail
Will exit 1 while any step of a pipe fail
```shell
set -o pipefail
```

`<()`：Process Substitution

```shell
diff <(ls dir1) <(ls dir2)
```

> **执行命令，把输出提供成一个类似文件的输入源。**

<(command)  → 把 command 的输出，当成“文件”给别人读 ; >(command)  → 把别人写入这个“文件”的内容，送给 command

`tee` 是 Linux/Bash 里一个非常常用的命令，可以理解成：

> **把输入“复制”一份，同时送到多个地方。**

```shell
echo "hello" | tee output.txt
```
          ┌──→ terminal
          │
input → tee
          │
          └──→ file

`tee` 最常见的用途：看日志 + 保存日志

```shell
echo "hello" | tee >(grep hello) >(grep world)
```

#### subshell

Using () to start a subshell 

the code is treated as a single unit

```shell
(
echo Some output, ls to follow
ls
) >> logfile

```

`()` vs `{}`
for () , it is starting a subshell. For {} No subshell started and need a ; to stop the cmd

Another useful feature of subshells is that if you change folder, then that folder change only applies within the subshell. When you return, you go back to where you were.

1 {} 为了防止变量名粘连
2`${...}` 是 Bash 的**参数展开（parameter expansion）语法**。

```shell
//var未定义或为空时使用default
${var:-default}

//var未定义时使用default
${var-default}
```

HTTP curl常用参数

```shell
-k       --insecure
-f       --fail
-s       --silent
-S       --show-error
-L       --location

-X       指定 HTTP method
-d       POST data
-F       multipart/form-data
-H       HTTP header

-o       指定输出文件
-O       使用 URL 文件名

-u       Basic Auth
-b       Cookie
-c       保存 Cookie

-I       只看 headers
-i       headers + body
-v       debug

-m       最大请求时间
--connect-timeout
--retry
```

`IFS` 是 Bash 里非常重要的一个特殊变量，全称是：
> **Internal Field Separator（内部字段分隔符）
> “一个字符串应该按照什么字符拆成多个字段。”**

What we want to do is treat the spaces like any other character. We can do this by altering the `IFS` variable to remove the space and re-running the for loop:
```shell
IFS=$'\t\n'
for f in $(ls)
do
    ls $f
done
```

```shell
find . -type f | xargs -n1 grep somestring

find . -type f -print0 | xargs -0 -n1 grep somestring
```

-print0 找到文件以后，不使用换行符分隔，而是使用 **NUL byte** 分隔。
-0 意思就是：
> 使用 NUL byte 作为输入字段的分隔符。

Readline Library

Here’s a table of the most frequently used shortcuts for reference:

\C is Control

|Readling Shortcut|Effect|
|---|---|
|\C-a|Takes you to start of line|
|\C-e|Takes you to end of line|
|\C-h|Delete character|
|\C-l|Clear screen|
|\C-r|Review history|
The `-n` flag tells echo not to print a newline.
```shell
echo -n 'JUST THESE CHARACTERS'
```

'...'       字面量，几乎什么都不解释
"...$var"   双引号，允许变量/命令等展开
$'...\n'    ANSI-C quoting，解释转义字符

|系统|换行符|
|---|---|
|Linux / Unix|`\n`|
|Windows|`\r\n`|
#### Prompt String

`PS1`：最常见的命令提示符
```
PS1='My super prompt>>>>   '
```
`PS2`：命令还没结束时 多行命令也是 PS2

```shell
echo hello \
world
```

`PS4`：`set -x` 调试

常见的 Prompt Escape 加入PS字符可以定义文字

|写法|含义|
|---|---|
|`\u`|username|
|`\h`|hostname（短）|
|`\H`|完整 hostname|
|`\w`|当前完整路径|
|`\W`|当前目录名|
|`\t`|当前时间 `HH:MM:SS`|
|`\d`|日期|
|`\n`|换行|
|`\r`|carriage return|
|`\!`|当前 history number|
|`\$`|普通用户 `$`，root `#`|

hereDOC
不加引号 里面会进行 expansion。
```shell
<< END
$var
$(command)
`command`
END
```
加引号
```shell
<< 'END'
$var
$(command)
`command`
END
```
里面不会进行 expansion。
<<- 'END' 可以去掉内部每一行的 Tab 

Bash history
```bash
!! //重复上一条命令
!$ //上一条命令最后一个参数
！* //上一条命令所有参数
! -n //执行前n条命令
!string //执行做后一条以string开头的命令
!?About? //找最近一条**包含 `About`** 的历史命令。
```

Return all absolute paths
```shell
function sharefiles() {
    for file in $(ls "$@"); do
        echo -n $(pwd)
        [[ $(pwd) != "/" ]] && echo -n /
        echo $file
    done
}
sharefiles
```

JOB Control
```
jobs
```
查看 jobs。

```
fg %1
```
Job 1 → 前台。

```
bg %1
```
Job 1 → 后台继续运行。

```
wait %1
```
等待 Job 1 完成。

```
Ctrl+C
```
发送 `SIGINT`，通常终止前台程序。

```
Ctrl+Z
```
发送 `SIGTSTP`，暂停前台程序。

Job Control
│
├── Jobs
│   └── shell 管理的进程/进程组
│
├── Job Specification
│   ├── %1       → job 1
│   ├── %2       → job 2
│   ├── %+       → current job
│   └── %-       → previous job
│
├── Signals
│   ├── Ctrl+C → SIGINT  → interrupt
│   ├── Ctrl+Z → SIGTSTP → stop
│   └── SIGCONT          → continue
│
├── fg
│   └── background/stopped → foreground
│
├── bg
│   └── stopped → background + continue
│
└── wait
    └── shell 等待 background job 完成

#### Trap EXIT

当 shell 即将退出时执行 `xxx`。
```shell
trap 'xxx' 
```

例如脚本创建临时文件：
```
#!/bin/bash

tmpfile=/tmp/test.txt

touch "$tmpfile"

trap 'rm -f "$tmpfile"' EXIT

# 做一些事情
...
```
最后无论脚本怎么正常退出，都把临时文件删除。

Debug
bash -n xxxx.sh

|Option|名称|作用|
|---|---|---|
|`-n`|noexec|不执行，只检查语法|
|`-v`|verbose|打印读取到的脚本|
|`-x`|xtrace|打印实际执行的命令|
#### String Manipulation
```shell

A='12345678901234567890'
echo "${#A}"               # How long is 'A'?

echo ${A:2}  #从2index处取全部
echo ${A:2:3}  #从2index处取三个

# The first part enclosed in `/` signs represents what’s searched for, and the second what is replaced:

echo "${A/234/432}"。#只换出现的第一个
echo "${A//234/432}"。#换全部

C=AbCd
echo ${C,}   #第一个换小写
echo ${C,,}    #全换小写
echo ${C^}
echo ${C^^}    #大写

  
```

sed → 修改/替换文本 
awk → 按列处理、过滤、统计 特别适合处理 CSV / columns
perl → 更强大的文本处理/正则/脚本

```shell
sed 's/hello/hi' file.txt #每行第一个换
sed 's/hello/hi/g' file   #全部换
sed -i 's/hello/hi/g' file.txt  #原位替换

sed '2d' file.txt  #删除第 2 行
sed '2d，5' file.txt #删除第 2-5 行
sed '/error/d' file.txt  #删除包含 `error` 的行

sed -n '2p' file.txt     
#-n → 不默认打印 ，2p → print 第 2 行
```
awk
$1 → 第一列 $2 → 第二列 $3 → 第三列
```shell
awk '{print $1}' file.txt
awk '{print $0}' file.txt   #整行。

awk -F',' '{print $2}' file.csv   #Field separator = `,`

#`awk` + 条件 awk '条件 {动作}' 文件
awk '$3 > 30 {print $2}' file.txt
awk '{sum += $2} END {print sum}' file.txt

awk '{print NR, $0}' file.txt #`NR`：当前第几行
awk '{print NF}' file.txt #`NF`：这一行有多少列
```
Perl
```shell
# Perl 替换
perl -pe 's/hello/hi/g' file.txt
# -p 读取每一行 → 执行代码 → 打印这一行  
perl -pi -e 's/foo/bar/g' file.txt #原位修改

perl -ne '代码' file
#- `-n` = 逐行读取，但不自动打印 - `-e` = **直接执行后面的 Perl 不用写 `.pl` 脚本文件）
-n = no automatic print 
-p = print each line 
-e = execute code

```