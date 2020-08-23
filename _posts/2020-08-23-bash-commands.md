---
title:  "Bash Commands: scripts, built-ins, and keywords"
last_modified_at: 
categories: 
  - Bash
tags:
  - bash
toc: true
toc_label: 
---

Bash can be an intimidating language if you are new to bash scripts. But note that bash scritps are simply collections of commands that can be entered in the command line. Today let's see how these commands can be classified.

To find out the type of your command, you can use
```
type -t <Command>
```

## Scripts

Scripts are executables that can be run with an interpreter provided by the system. Scripts are written in languages like Bash, Python, or Perl. As long as you have the interpreter for these languaes, you can use them like commands. `chmod` which is located in /bin/chmod is one of the executables/scripts.

```
$ type -t chmod
file
```

## Built-ins

These are commands provided by the Shell. A lot of these may look familiar to you. For example, `pwd` is one of them. You don't need an interpreter to use these commands. And as you may have guessed, these are generally faster and more efficient that running non-builtin commands.

```
$ type -t pwd
builtin
```

To see a full list of built-ins, you can use `compgen -b`
```
$ compgen -b
.
:
[
alias
bg
bind
break
builtin
caller
cd
command
compgen
complete
continue
declare
dirs
disown
echo
enable
eval
exec
exit
export
false
fc
fg
getopts
hash
help
history
jobs
kill
let
local
logout
popd
printf
pushd
pwd
read
readonly
return
set
shift
shopt
source
suspend
test
times
trap
true
type
typeset
ulimit
umask
unalias
unset
wait
```


## Keywords

Because you write scripts with Bash, you would expect commands like `if` and `else` to control the logic flow. These are not files located in your file system or builtins that come with the shell. These keywords are simply a part of the language. 
```
$ type -t if
keyword
```

You can see the full list by using `compgen -k`
```
$ compgen -k 
if
then
else
elif
fi
case
esac
for
select
while
until
do
done
in
function
time
{
}
!
[[
]]
```