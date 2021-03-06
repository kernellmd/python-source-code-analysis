# Python源码剖析笔记0-C语言基础回顾

> 要分析python源码，C语言的基础不能少，特别是指针和结构体等知识。这篇文章先回顾C语言基础，方便后续代码的阅读。

## 1 关于ELF文件
linux中的C编译得到的目标文件和可执行文件都是ELF格式的，可执行文件中以segment来划分，目标文件中，我们是以section划分。一个segment包含一个或多个section，通过readelf命令可以看到完整的section和segment信息。看一个栗子：

```
char pear[40];
static double peach;
int mango = 13;
char *str = "hello";

static long melon = 2001;

int main()
{
        int i = 3, j;
        pear[5] = i;
        peach = 2.0 * mango;
        return 0;
}
```
这是个简单的C语言代码，现在分析下各个变量存储的位置。其中mango，melon属于data section，pear和peach属于common section中，而且peach和melon加了static，说明只能本文件使用。而str对应的字符串"helloworld"存储在rodata section中。main函数归属于text section，函数中的局部变量i,j在运行时在栈中分配空间。注意到前面说的全局未初始化变量peach和pear是在common section中，这是为了强弱符号而设置的。那其实最终链接成为可执行文件后，会归于BSS segment。同样的，text section和rodata section在可执行文件中都属于同一个segment。

更多ELF内容参见《程序猿的自我修养》一书。


## 2 关于指针
想当年学习C语言最怕的就是指针了，当然《c与指针》和《c专家编程》以及《高质量C编程》里面对指针都有很好的讲解，系统回顾还是看书吧，这里我总结了一些基础和易错的点。环境是ubuntu14.10的32位系统，编译工具GCC。

### 2.1 指针易错点

```
/***
指针易错示例1 demo1.c
***/

int main()
{
        char *str = "helloworld"; //[1]
        str[1] = 'M'; //[2] 会报错
        char arr[] = "hello"; //[3]
        arr[1] = 'M';
        return 0;
}
```
demo1.c中，我们定义了一个指针和数组分别指向了一个字符串，然后修改字符串中某个字符的值。编译后运行会发现[2]处会报错，这是为什么呢？用命令```gcc -S demo1.c``` 生成汇编代码就会发现[1]处的helloworld是存储在rodata section的，是只读的，而[3]处的是存储在栈中的。所以[2]报错而[3]正常。在C中，用[1]中的方式创建字符串常量并赋值给指针，则字符串常量存储在rodata section。而如果是赋值给数组，则存储在栈中或者data section中（如[3]就是存储在栈中）。示例2给出了更多容易出错的点，可以看看。

```
/***
指针易错示例2 demo2.c
***/
char *GetMemory(int num) {
        char *p = (char *)malloc(sizeof(char) * num);
        return p;
}

char *GetMemory2(char *p) {
        p = (char *)malloc(sizeof(char) * 100);
}

char *GetString(){
        char *string = "helloworld";
        return string;
}

char *GetString2(){
        char string[] = "helloworld";
        return string;
}

void ParamArray(char a[])
{
        printf("sizeof(a)=%d\n", sizeof(a)); // sizeof(a)=4，参数以指针方式传递
}

int main()
{
		int a[] = {1, 2, 3, 4};
        int *b = a + 1;
        printf("delta=%d\n", b-a); // delta=4，注意int数组步长为4
        printf("sizeof(a)=%d, sizeof(b)=%d\n", sizeof(a), sizeof(b)); //sizeof(a)=16, sizeof(b)=4
        ParamArray(a); 
        
        
        //引用了不属于程序地址空间的地址，导致段错误
        /*
        int *p = 0;
        *p = 17;         
        */
        
        char *str = NULL;
        str = GetMemory(100);
        strcpy(str, "hello");
        free(str); //释放内存
        str = NULL; //避免野指针

		//错误版本，这是因为函数参数传递的是副本。
		/*
        char *str2 = NULL;
        GetMemory2(str2);
        strcpy(str2, "hello");
        */

        char *str3 = GetString();
        printf("%s\n", str3);

		//错误版本，返回了栈指针，编译器会有警告。
		/*
        char *str4 = GetString2();
        */
        return 0;
}
```
### 2.2 指针和数组
在2.1中也提到了部分指针和数组内容，在C中指针和数组在某些情况下可以相互转换来使用，比如```char *str="helloworld"```可以通过```str[1]```来访问第二个字符，也可以通过```*(str+1)```来访问。
此外，在函数参数中，使用数组和指针也是等同的。**但是指针和数组在有些地方并不等同，需要特别注意。**

比如我定义一个数组```char a[9] = "abcdefgh";```(注意字符串后面自动补\0),那么用a[1]读取字符'b'的流程是这样的：

- 首先，数组a有个地址，我们假设是9980。
- 然后取偏移值，偏移值为索引值*元素大小，这里索引是1，char大小也为1，因此加上9980为9981，得到数组a第1个元素的地址。（如果是int类型数组，那么这里偏移就是1 * 4 = 4）
- 取地址9981处的值，就是'b'。

那如果定义一个指针```char *a = "abcdefgh";```，我们通过a[1]来取第一个元素的值。跟数组流程不同的是：

- 首先，指针a自己有个地址，假设是4541.
- 然后，从4541取a的值，也就是字符串“abcdefgh”的地址，假定是5081。
- 接着就是跟之前一样的步骤了，5081加上偏移1，取5082地址处的值，这里就是'b'了。

通过上面的说明可以发现，指针比数组多了一个步骤，虽然看起来结果是一致的。因此，下面这个错误就比较好理解了。在demo3.c中定义了一个数组，然后在demo4.c中通过指针来声明并引用它，显然是会报错的。如果改成```extern char p[];```就正确了（当然声明你也可以写成extern char p[3],声明里面的数组大小跟实际大小不一致是没有关系的），一定要保证定义和声明匹配。

```
/***
demo3.c
***/
char p[] = "helloworld";

/***
demo4.c
***/
extern char *p;
int main()
{
        printf("%c\n", p[1]);
        return 0;
}
```

## 3 关于typedef和#define
typedef和#define都是经常用的，但是它们是不一样的。一个typedef可以塞入多个声明器，而#define一般只能有一个定义。在连续声明中，typedef定义的类型可以保证声明的变量都是同一种类型，而#define不行。此外，typedef是一种彻底的封装类型，在声明之后不能再添加其他的类型。如代码中所示。


```
#define int_ptr int *
int_ptr i, j; //i是int *类型，而j是int类型。

typedef char * char_ptr;
char_ptr c1, c2; //c1, c2都是char *类型。

#define peach int
unsigned peach i; //正确

typdef int banana;
unsigned banana j; //错误，typedef声明的类型不能扩展其他类型。
```

另外，typedef在结构体定义中也很常见，比如下面代码中的定义。需要注意的是，[1]和[2]是很不同的。当你如[1]中那样用typedef定义了struct foo，那么其实除了本身的foo结构标签，你还定义了foo这种结构类型，所以可以直接用foo来声明变量。而如[2]中的定义是不能用bar来声明变量的，因为它只是一个结构变量，并不是结构类型。

还有一点需要说明的是，结构体是有自己名字空间的，所以结构体中的字段可以跟结构体名字相同，比如[3]中那样也是合法的，当然尽量不要这样用。后面一节还会更详细探讨结构体，因为在Python源码中也有用到很多结构体。

```
typedef struct foo {int i;} foo; //[1]
struct bar {int i;} bar; //[2]

struct foo f; //正确，使用结构标签foo
foo f; //正确，使用结构类型foo

struct bar b; //正确，使用结构标签bar
bar b; // 错误，使用了结构变量bar，bar已经是个结构体变量了，可以直接初始化，比如bar.i = 4;

struct foobar {int foorbar;}; //[3]合法的定义
```

## 4 关于结构体
在学习数据结构的时候，定义链表和树结构会经常用到结构体。比如下面这个：

```
struct node {
    int data;
    struct node* next;
};
```
在定义链表的时候可能就有点奇怪了，为什么可以这样定义，貌似这个时候struct node还没有定义好为什么就可以用next指针指向用这个结构体定义了呢？

### 4.1 不完全类型
这里要说下C语言里面的不完全类型。C语言可以分为函数类型，对象类型以及不完全类型。而对象类型还可以分为标量类型和非标量类型。算术类型（如int，float，char等）和指针类型属于标量类型，而定义完整的结构体，联合体，数组等都是非标量类型。而不完全类型是指没有定义完整的类型，比如下面这样的

```
struct s;
union u;
char str[];
```
具有不完全类型的变量可以通过多次声明组合成一个完全类型。比如下面2词声明str数组是合法的：

```
char str[];
char str[10];
```
此外，如果两个源文件定义了同一个变量，只要它们不全部是强类型的，那么也是可以编译通过的。比如下面这样是合法的，但是如果将file1.c中的```int i;```改成强定义如```int i = 5;```那么就会出错了。

```
//file1.c
int i;

//file2.c
int i = 4;
```

### 4.2 不完全类型结构体
不完全类型的结构体十分重要，比如我们最开始提到的struct node的定义，编译器从前往后处理，发现```struct node *next```时，认为struct node是一个不完全类型，next是一个指向不完全类型的指针，尽管如此，指针本身是完全类型，因为不管什么指针在32位系统都是占用4个字节。而到后面定义结束，struct node成了一个完全类型，从而next就是一个指向完全类型的指针了。

### 4.3 结构体初始化和大小
结构体初始化比较简单，需要注意的是结构体中包含有指针的时候，如果要进行字符串拷贝之类的操作，对指针需要额外分配内存空间。如下面定义了一个结构体student的变量stu和指向结构体的指针pstu，虽然stu定义的时候已经隐式分配了结构体内存，但是你要拷贝字符串到它指向的内存的话，需要显示分配内存。

```
struct student {
	char *name;
	int age;
} stu, *pstu;

int main()
{
    stu.age = 13; //正确
    // strcpy(stu.name,"hello"); //错误，name还没有分配内存空间
        
    stu.name = (char *)malloc(6);
    strcpy(stu.name, "hello"); //正确
        
    return 0;
}
```

结构体大小涉及一个对齐的问题，对齐规则为：

- 结构体变量首地址为最宽成员长度（如果有```#pragma pack(n)```，则取最宽成员长度和n的较小值，默认pragma的n=8）的整数倍
- 结构体大小为最宽成员长度的整数倍
- 结构体每个成员相对结构体首地址的偏移量都是每个成员本身大小（如果有pragma pack(n),则是n与成员大小的较小值）的整数倍
因此，下面结构体S1和S2虽然内容一样，但是字段顺序不同，大小也不同，```sizeof(S1) = 8, 而sizeof(S2) = 12```. 如果定义了```#pragma pack(2)```，则```sizeof(S1)=8；sizeof(S2)=8```

```
typedef struct node1
{
    int a;
    char b;
    short c;
}S1;

typedef struct node2
{
	char b;
    int a;
    short c;
}S2;
```
### 4.4 柔性数组
柔性数组是指结构体的最后面一个成员可以是一个大小未知的数组，这样可以在结构体中存放变长的字符串。如代码中所示。**注意，柔性数组必须是结构体最后一个成员,柔性数组不占用结构体大小.**当然，你也可以将数组写成```char str[0]```,含义相同。

**updated：** 查看Python源码过程中，发现其柔性数组声明并不是用一个空数组或者```char str[0]```，而是用的```char str[1]```，即数组大小为1.这是因为C/C++标准不允许声明大小为0的数组，为了可移植性，所以常常看到的是声明数组大小为1。当然，很多编译器比如GCC等把数组大小为0作为了一个非标准的扩展，所以声明空的或者大小为0的柔性数组在GCC中是可以正常编译的。

```
struct flexarray {
        int len;
        char str[];
} *pfarr;

int main()
{
        char s1[] = "hello, world";
        pfarr = malloc(sizeof(struct flexarray) + strlen(s1) + 1);
        pfarr->len = strlen(s1);
        strcpy(pfarr->str, s1);
        printf("%d\n", sizeof(struct flexarray)); // 4
        printf("%d\n", pfarr->len); // 12
        printf("%s\n", pfarr->str); // hello, world
        return 0;
}
```

## 5 总结

- 关于const，c语言中的const不是常量，所以不能用const变量来定义数组，如```const int N = 3; int a[N];```这是错误的。
- 注意内存分配和释放，杜绝野指针。
- C语言中弱符号和强符号一起链接是合法的。
- 注意指针和数组的区别。
- typedef和#define是不同的。
- 注意包含指针的结构体的初始化和柔性数组的使用。

## 6 参考资料
- 《c专家编程》
- 《linux c一站式编程》
- 《高质量C/C++编程》
- [结构体字节对齐](http://www.cnblogs.com/dolphin0520/archive/2011/09/17/2179466.html)
- [柔性数组](http://www.cnblogs.com/Daniel-G/archive/2012/12/02/2798496.html)
