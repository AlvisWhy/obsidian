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

Using export, all of the variables can be reached by any process with $

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

