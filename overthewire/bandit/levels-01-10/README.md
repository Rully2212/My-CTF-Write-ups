# Learning Linux Commands with OverTheWire Bandit

This write-up summarizes the commands learned while working through the early OverTheWire Bandit levels. The goal is to understand how Linux commands find and read files, rather than memorize answers.

> Note: Do not include actual passwords or flags in public write-ups on GitHub. Use commands, explanations, and placeholders such as `<PASSWORD_BANDIT11>`.

---

## 1. Reading a File Named `-`

If a file is named:

```bash
-
```

Do not simply run:

```bash
cat -
```

`cat -` reads from keyboard input or `stdin` instead of opening a file named `-`.

Correct command:

```bash
cat ./-
```

Alternative:

```bash
cat -- -
```

Explanation:

```bash
./-
```

refers to a file named `-` in the current directory.

```bash
--
```

marks the end of command options, so the following `-` is treated as a filename.

If the terminal is already waiting for input after running `cat -`, press:

```bash
CTRL + D
```

---

## 2. Reading a Filename with Spaces That Starts with `--`

Example filename:

```bash
--spaces in this filename--
```

There are two issues:

1. The filename contains spaces, so it must be quoted.
2. It starts with `--`, so it may be interpreted as a command option.

Correct command:

```bash
cat "./--spaces in this filename--"
```

Alternative:

```bash
cat -- "--spaces in this filename--"
```

Explanation:

```bash
"file name"
```

makes the shell treat the filename as one argument, even when it contains spaces.

```bash
./
```

specifies that the file is in the current directory and prevents its name from being interpreted as a command option.

---

## 3. Reading Hidden Files That Start with a Dot

On Linux, files whose names start with a dot `.` are hidden files.

Example filename:

```bash
...Hiding-From-You
```

To list hidden files, use:

```bash
ls -la
```

To read the file:

```bash
cat ./...Hiding-From-You
```

Key details:

```bash
.
```

means the current directory.

```bash
..
```

means the parent directory.

```bash
...
```

has no special command meaning here. In a filename such as `...Hiding-From-You`, the three dots are simply part of the name.

---

## 4. Finding Files by Size and Permissions with `find`

If there are many directories such as:

```bash
maybehere00
maybehere01
maybehere02
...
maybehere19
```

use `find` instead of checking each directory individually.

Example clue: the file is `1033 bytes`, human-readable, and not executable.

Command:

```bash
find . -type f -size 1033c ! -executable -readable
```

Explanation:

```bash
.
```

starts the search in the current directory.

```bash
-type f
```

matches files rather than directories.

```bash
-size 1033c
```

matches files exactly 1033 bytes long. The suffix `c` means bytes.

```bash
! -executable
```

matches files that are not executable.

```bash
-readable
```

matches readable files.

After finding the file path, read its contents:

```bash
cat ./path/find-result
```

Example:

```bash
cat ./maybehere07/.file2
```

To read the matching files directly:

```bash
find . -type f -size 1033c ! -executable -readable -exec cat {} \;
```

---

## 5. Searching the Entire Server

A clue may say that the file is "somewhere on the server." In that case, searching only the current directory is not sufficient.

If you use:

```bash
find . -type f -user bandit7 -group bandit6 -size 33c
```

the search starts only from the current directory.

To search the entire system, start from `/`:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

Explanation:

```bash
/
```

starts at the root directory and searches the system.

```bash
-user bandit7
```

matches files owned by the user `bandit7`.

```bash
-group bandit6
```

matches files belonging to the group `bandit6`.

```bash
-size 33c
```

matches files that are 33 bytes long.

```bash
2>/dev/null
```

hides errors such as `Permission denied`.

After finding the path, read the file:

```bash
cat /path/found-file
```

Or read it directly:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null -exec cat {} \;
```

---

## 6. Common `find` Commands

Find all files:

```bash
find . -type f
```

Find all directories:

```bash
find . -type d
```

Find files by name:

```bash
find . -name "file.txt"
```

Find hidden files:

```bash
find . -name ".*"
```

Find files of a specific size:

```bash
find . -type f -size 1033c
```

Find files larger than 1 MB:

```bash
find . -type f -size +1M
```

Find files smaller than 1 KB:

```bash
find . -type f -size -1k
```

Find readable files:

```bash
find . -type f -readable
```

Find executable files:

```bash
find . -type f -executable
```

Find files that are not executable:

```bash
find . -type f ! -executable
```

Find files owned by a specific user:

```bash
find . -type f -user bandit5
```

Find files belonging to a specific group:

```bash
find . -type f -group bandit5
```

Find files by permissions:

```bash
find . -type f -perm 644
```

Find files with at least one execute permission bit set:

```bash
find . -type f -perm /111
```

Find empty files:

```bash
find . -type f -empty
```

Find empty directories:

```bash
find . -type d -empty
```

Run a command on the files returned by `find`:

```bash
find . -type f -name "*.txt" -exec cat {} \;
```

---

## 7. Differences Between `grep`, `cat`, `wget`, `curl`, and `scp`

`grep` searches text in files or command output; it does not download files.

Example:

```bash
grep "millionth" data.txt
```

This finds lines containing the word `millionth` in `data.txt`.

To read a file:

```bash
cat data.txt
```

To download a file from a URL:

```bash
wget http://example.com/file.txt
```

or:

```bash
curl -O http://example.com/file.txt
```

To download a file from an SSH server to the local computer:

```bash
scp -P 2220 bandit6@bandit.labs.overthewire.org:/path/file .
```

Explanation:

```bash
-P 2220
```

uses SSH port 2220.

```bash
:/path/file
```

specifies the file location on the server.

```bash
.
```

saves the file in the current local directory.

For Bandit, downloading files is usually unnecessary. Read or process them directly on the server.

---

## 8. Finding Lines That Appear Once

To find lines that appear only once in a file, combine `sort` and `uniq`.

Command:

```bash
sort data.txt | uniq -u
```

Explanation:

```bash
sort data.txt
```

sorts the file contents so identical lines are adjacent.

```bash
uniq -u
```

prints lines that occur only once.

To count the occurrences of each line:

```bash
sort data.txt | uniq -c
```

To show lines whose occurrence count is 1:

```bash
sort data.txt | uniq -c | grep " 1 "
```

The shortest command for this Bandit level is:

```bash
sort data.txt | uniq -u
```

---

## 9. Finding Human-Readable Strings

Use `strings` to extract human-readable text from a binary file.

Basic command:

```bash
strings data.txt
```

If there is too much output, filter it with `grep`.

For example, find lines containing `=`:

```bash
strings data.txt | grep "="
```

Find a specific word:

```bash
strings data.txt | grep "password"
```

Ignore letter case:

```bash
strings data.txt | grep -i "password"
```

A common workflow:

```bash
file data.txt
strings data.txt
strings data.txt | grep "="
```

Explanation:

```bash
file data.txt
```

identifies the file type.

```bash
strings data.txt
```

extracts readable text from a binary file.

```bash
grep "="
```

filters for output containing `=`.

---

## 10. Decoding Base64

If a file already contains Base64-encoded text, do not encode it again.

Incorrect command:

```bash
base64 data.txt
```

The command above encodes the content again.

To decode Base64, use:

```bash
base64 -d data.txt
```

or:

```bash
base64 --decode data.txt
```

After obtaining the password, log in to the next level:

```bash
ssh bandit11@bandit.labs.overthewire.org -p 2220
```

Complete workflow:

```bash
ls
cat data.txt
base64 -d data.txt
ssh bandit11@bandit.labs.overthewire.org -p 2220
```

---

## 11. Decoding ROT13 with `tr`

ROT13 replaces each letter with the letter 13 positions later in the alphabet, wrapping around at the end.

ROT13 command:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Alternative without `cat`:

```bash
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
```

Explanation:

```bash
A-Za-z
```

represents all uppercase letters `A-Z` and lowercase letters `a-z`.

```bash
N-ZA-Mn-za-m
```

represents the alphabet rotated by 13 positions.

Uppercase ROT13 mapping:

```text
ABCDEFGHIJKLMNOPQRSTUVWXYZ
NOPQRSTUVWXYZABCDEFGHIJKLM
```

This means:

```text
A -> N
B -> O
C -> P
D -> Q
E -> R
F -> S
G -> T
H -> U
I -> V
J -> W
K -> X
L -> Y
M -> Z
N -> A
O -> B
P -> C
...
Z -> M
```

The sequence:

```bash
N-ZA-M
```

is shorthand for:

```text
NOPQRSTUVWXYZABCDEFGHIJKLM
```

Because:

```bash
N-Z
```

means the letters `N` through `Z`.

```bash
A-M
```

means the letters `A` through `M`.

Therefore:

```bash
N-ZA-M
```

is equivalent to:

```text
NOPQRSTUVWXYZABCDEFGHIJKLM
```

For lowercase letters:

```bash
n-za-m
```

is equivalent to:

```text
nopqrstuvwxyzabcdefghijklm
```

Example:

```bash
echo "hello" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Output:

```text
uryyb
```

If applied again:

```bash
echo "uryyb" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Output:

```text
hello
```

ROT13 uses the same command for encoding and decoding.

---

## 12. What Is `tr`?

`tr` is a Linux command for translating characters. It can replace, delete, or squeeze repeated characters in text input.

Basic syntax:

```bash
tr 'source_characters' 'replacement_characters'
```

Example of replacing letters:

```bash
echo "abc" | tr 'abc' '123'
```

Output:

```text
123
```

This means:

```text
a -> 1
b -> 2
c -> 3
```

Convert lowercase letters to uppercase:

```bash
echo "hello" | tr 'a-z' 'A-Z'
```

Output:

```text
HELLO
```

Delete spaces:

```bash
echo "h e l l o" | tr -d ' '
```

Output:

```text
hello
```

Delete digits:

```bash
echo "abc123" | tr -d '0-9'
```

Output:

```text
abc
```

---

## 13. A Common `tr` Error

Example of an incorrect command:

```bash
cat data.txt | tr 'A-Za-z'
```

Error:

```text
tr: missing operand after 'A-Za-z'
Two strings must be given when translating.
```

This happens because character translation with `tr` requires two strings:

```bash
tr 'source_characters' 'destination_characters'
```

Correct ROT13 command:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

---

## 14. Essential Command Reference

Read a regular file:

```bash
cat data.txt
```

Read a file named `-`:

```bash
cat ./-
```

Read a filename containing spaces:

```bash
cat "./--spaces in this filename--"
```

List hidden files:

```bash
ls -la
```

Find files by size:

```bash
find . -type f -size 1033c
```

Search for a file across the system:

```bash
find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

Search for text in a file:

```bash
grep "word" data.txt
```

Find lines that occur only once:

```bash
sort data.txt | uniq -u
```

Extract strings from a binary file:

```bash
strings data.txt
```

Decode Base64:

```bash
base64 -d data.txt
```

Decode ROT13:

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

Log in to Bandit over SSH:

```bash
ssh banditX@bandit.labs.overthewire.org -p 2220
```

Replace `banditX` with the level you want to access, such as `bandit11`.

---

