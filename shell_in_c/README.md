# shell

---

一个shell主要包括三个阶段：

+ 初始化：执行相应的初始化配置
+ 解释（操作）：根据相应的输入(stdin)，来进行相应的命令行操作
+ 结束：结束整个shell进程

首先需要对shell本身进行解析，实际上shell是这样一个进程，即在其中输入相应的指令，其会按照相应的操作，fork相应的子进程，并对其进行处理。

故实际上，我们应该需要三个部分来完成整个shell，当然，在写出整个shell之前，还是需要先完成一个整体的框架，那么将其规划为：

```c
int main()
{
    lsh_loop();

    return EXIT_SUCCESS;
}

```

即shell始终在等待lsh_loop()函数结束，并返回。故实际上具体的操作都将在lsh_loop中实现。

并且根据我们之前的分析，`lsh_loop`具体的操作应该是根据相应的参数进行相应的处理

```c
void lsh_loop()
{
    char *line;
    char **args;
    int status;

    do {
        printf("> ");
        line = lsh_read_line();
        args = lsh_split_line(line);
        status = lsh_execute(args);

        free(line);
        free(args);
    }while(status);
}
```

### 命令行读取

根据相应的理解，那么对于命令行的读取需要利用stdin来进行相应的操作即可，其首先开创一个空间来存储命令行，并以结束标志而停止读取。

```c
#define LSH_TOK_BUFSIZE 64
char *lsh_read_line(void)
{
    int bufsize = LSH_RL_BUFSIZE;
    int position = 0;
    char *buffer = malloc(sizeof(char) * bufsize);
    int c;

    if (!buffer)
    {
        fprintf(stderr,"lsh: allocation error\n");
        exit(EXIT_FAILURE);
    }

    while(1)
    {
        c = getchar();

        if(c ==EOF || c == '\n')
        {
            buffer[position] = '\0';
            return buffer;
        }
        else
            buffer[position] = c;
        position++;

        if(position >= bufsize)
        {
            bufsize += LSH_RL_BUFSIZE;
            buffer = realloc(buffer,bufsize);
            if(!buffer)
            {
                fprintf(stderr,"lsh: allocation error\n");
                exit(EXIT_FAILURE);
            }
        }
    }
}

```

但是，实际上,readline函数可以简单利用getline函数实现：

```c
char *lsh_read_line(void)
{
  char *line = NULL;
  ssize_t bufsize = 0; // have getline allocate a buffer for us
  getline(&line, &bufsize, stdin);
  return line;
}
```

接下来，需要对得到的命令行进行相应的处理，

```c
char **lsh_split_line(char *line)
{
    int bufsize = LSH_TOK_BUFSIZE,position = 0;
    char **tokens = malloc(bufsize * sizeof(char*));
    char *token;

    if(!token)
    {
        fprintf(stderr,"lsh: allocation error\n");
        exit(EXIT_FAILURE);
    }

    token = strtok(line, LSH_TOK_DELIM);
    while(token != NULL)
    {
        tokens[position] = token;
        position++;
        
        if(position >= bufsize)
        {
            bufsize += LSH_TOK_BUFSIZE;
            token = realloc(tokens,bufsize*sizeof(char*));
            if(!tokens)
            {
                fprintf(stderr , "lsh: allocation error\n");
                exit(EXIT_FAILURE);
            }
        }
        token = strtok(NULL,LSH_TOK_DELIM);
    }
    tokens[position] = NULL;
    return tokens;

}

```

### 创建子进程

对于新进程的创建，只有两种可能性（unix 类操作系统中）,一是在启动系统时直接产生的Init，即第一个进程，进程0，其pid = 0，二是利用父进程创建子进程，所以实际上所有运行的进程都是Init进程的子进程或者其后续子进程。

关于创建子进程的具体操作为：
利用系统调用函数`fork`创建子进程，这个函数被调用时，操作系统会拷贝原进程的内容到新的进程中并让两个进程并发。`fork`将在两个不同进程中返回两个不同的值，子进程中返回0，父进程中返回其子进程pid(process id)。由于`fork`函数在进行调用的时候，实际上是直接拷贝，然而我们创建一个新进程，显然是希望它能够执行新的操作，而不是原来的操作，故而，我们需要使用这样一个函数`exec`，其使得操作系统停止当前进程的运行，并为其加载上新的程序，并在另一块地址空间中运行。因此，我们这样就完成了整个过程，首先在shell进程中利用fork函数创建子进程，对于子进程而言，利用exec来替换原来的程序，并执行新的程序，而对于父进程而言，则需要等待子进程运行完成，即需要调用wait函数。

在子进程中，我们希望运行用户所给出的命令，故而使用`execvp`，一个`exec`函数多种变体中的一个，在这里，v代表一个vector，表示一个字符串参数的数组，p代表除了提供运行程序的相应路径，我们还将给出进程的名字，当其返回-1时，代表运行的进程出错。

```c
int lsh_launch(char **args)
{
    pid_t pid,wpid;
    int status;
    
    pid = fork();
    if(pid == 0)
    {
        //子进程
        if(execvp(args[0],args) == -1)
            perror("lsh");
        exit(EXIT_FAILURE);
    }
    else if(pid < 0)
    {
        perror("lsh");
    }
    else
    {
        //父进程
        do{
            wpid = waitpid(pid,&status,WUNTRACED);
        }while(!WIFEXITED(status) && !WIFSIGNALED(status));
    }

    return 1;
}

```

### 处理命令行

根据相应的命令行指令进行处理，首先需要存储相应的命令行：

```c
char *builtin_str[] = {
    "cd",
    "help",
    "exit"
};
//在这里只实现一个极为简易的shell ，只包括这三条指令，后续可以考虑增加指令，包括ls等等
```

然后需要将当前的命令与内存里存有的命令进行比对：

```c
int (*builtin_func[]) (char **) = {
    &lsh_cd,
    &lsh_help,
    &lsh_exit
};

int lsh_execute(char **args)
{
    int i;

    if(args[0] == NULL)
        return 1;

    for (i = 0;i<lsh_num_builtins();i++)
    {
        if(strcmp(args[0],builtin_str[i]) == 0)
            return (*builtin_func[i])(args);
    }

    return lsh_launch(args);

}

int lsh_num_builtins(){
    return sizeof(builtin_str) / sizeof(char *);
}

```

如果发现能够匹配，则利用相应的进行调用即可，否则调用相应的主函数进行对于指令的处理。

下面主要讨论这三个函数的具体实现：

#### exit

退出函数实际上十分简单，只需要对其进行相应的退出处理即可。即直接退出，返回0（表示正常退出），对于一个c程序而言，主函数中，或者说一个进程返回0表示正常结束，否则表示异常结束。

```c
int lsh_exit(char **args)
{
    return 0;
}
```



#### cd

cd 的操作实际上非常简单，但是一定要对其进行相应的注意。首先cd一定是需要进入到相应的目录中，即除了cd本身之外，需要有第二个参数`args[1]`，否则需要报错，即告诉其需要相应的地址。

如果有相应的参数，那么实际上是需要调用c语言中的查找当前路径是否存在的函数`chdir`，并根据返回值进行判断。

```c
int lsh_cd(char **args)
{
    if(args[1] == NULL)
    {
        fprintf(stderr,"lsh: expected argment to \"cd\"\n");
    }
    else
        if(chdir(args[1]) != 0)
            perror("lsh");
}

```



#### help

`help`函数会根据相应的处理，来得到相应的结果。首先打印一些基本的东西，然后显示出当前所拥有的所有命令的可能，并打印出相关信息说明即可。

```c
int lsh_help(char **args)
{
    int i;
    printf("Stephen Brennan's LSH\n");
    printf("Type program names and arguments, and hit enter.\n");
    printf("The following are built in:\n");

    for (i=0;i<lsh_num_builtins();i++)
        printf("  %s\n",builtin_str[i]);

    printf("Use the man command for information on other programs. \n");
    return 1;
}
```

