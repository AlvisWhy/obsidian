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

grep -i(ignore case) 
		-o(only extract contained piece)
		-E (regex)
		-v(revert match)
		-c(count line num)
		-x(all line full match)

awk separate the line
awk -F, '{print $1, $2}'