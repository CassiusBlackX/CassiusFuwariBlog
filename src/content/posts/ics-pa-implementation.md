---
title: ics-pa-implementation
published: 2024-05-07
description: ''
image: ''
tags: ["linux", "qemu"]
category: 'notes'
draft: false 
lang: ''
---
# NJU-ICS-PA 自学笔记
## PA1
### PA1.1 单步执行
了解strtok()的相关内容：（该函数用于用制定分隔符来分割字符串）

> 	char *strtok(char *str, const char *delim);
> 	   														
> 	strtok() 函数将字符串分解为由零个或多个非空标记组成的序列。第一次调用 strtok() 时，应在 str 中指定要解析的字符串。在随后每次解析同一字符串的调用中，str 必须为空。delim 参数指定了一组字节，用于分隔解析字符串中的标记。 调用者可以在解析同一字符串的连续调用中，在 delim 中指定不同的字符串。
> 	    														
> 	返回值： strtok() 和 strtok_r() 函数返回指向下一个标记的指针，如果没有更多标记，则返回 NULL。

利用strtok()函数可以读出si接续的参数N（如果有的话，没有则默认N值为1），并调用cpu_exec()函数来完成对于N步的执行。代码如下。

```c
static int cmd_si(char *args) {
  char *arg = strtok(NULL, " ");
  int n;
  if (arg == NULL) {
    n = 1;
  }
  else {
    n = atoi(arg);
  }
  cpu_exec(n);
  return 0;
}

```

### PA1.2 打印寄存器

同样是利利用strtok()函数可以读出info接续的参数arg，如果arg对应为字符串"r"，则表示目前执行的是打印寄存器的任务。代码框架中在`nemu/src/isa/riscv32/reg.c`目录中定义了isa_reg_display()函数，可以在其中完成对寄存器值的输出（基本为补充printf）。isa_reg_display()函数的补充和cmd_info的代码分别如下（info中包含了监视点的前置代码）。

```c
void isa_reg_display() {
  // printf("%4s  0x%08x\n", "pc",cpu.pc);
  for (int i = 0; i < 32; i++) {
    printf("%4s  0x%08x  %8d\n", regsl[i], cpu.gpr[i]._32, cpu.gpr[i]._32);
  }
}
```

```c
static int cmd_info(char *args){
  char *arg = strtok(NULL, " ");
  if(arg == NULL){
    printf("Unknown command, please input the subcmd!\n");
  }
  else if(strcmp(arg, "r") == 0){
    isa_reg_display();
  }
  else if(strcmp(arg, "w") == 0){

  }
  else{
    printf("Unknown command, please check the subcmd!\n");
  }
  return 0;
}
```



### PA1.3 扫描内存

利用了strtol()函数。其形式如下：

```c
long int strtol(const char *str, char **endptr, int base)
```

参数的含义：

- **str** -- 要转换为长整数的字符串。
- **endptr** -- 对类型为 char* 的对象的引用，其值由函数设置为 **str** 中数值后的下一个字符。
- **base** -- 基数，必须介于 2 和 36（包含）之间，或者是特殊值 0。如果 base 为 0，则会根据字符串的前缀来判断进制：如果字符串以 '0x' 或 '0X' 开头，则将其视为十六进制；如果字符串以 '0' 开头，则将其视为八进制；否则将其视为十进制。

利用该函数就可以将输入的地址字符串转换为对应地址数字，从而可以读取对应地址上的数值并且输出，同时根据输入的参数控制该过程的循环次数，扫描内存也就完成了。对应函数关键代码如下。

```c
static int cmd_x(char *args) { 
	char *N = strtok(NULL, " ");
  char *arg = strtok(NULL, " ");
  if(N == NULL || arg == NULL){
    printf("Unknown command, please input the N and the address!\n");
    return 0;
  }
  int n = atoi(N);
	paddr_t addr = strtol(arg, NULL, 16);
	for(int i=0; i<n; i++) {
		printf("0x%08x: ", addr);
		for(int j=0; j<4; j++) {
			printf("%02x ", paddr_read(addr, 1));
			addr++;
		}
		printf("\n");
	}
	return 0;
}
```

### PA1.4 表达式求值

PA实验中使用以下方法解决表达式求值的问题:

1. 首先识别出表达式中的单元
2. 根据表达式的归纳定义进行递归求值

因此要拆成两个步骤分别分析。

#### 1.4.1 词法分析

第一步的重点在于识别出一个表达式的所有token。token所包含的内容文档中有列举（实际上还包含很多其他内容，包括十六进制整数、八进制整数，其他各类运算符，寄存器等等）。这个过程即词法分析的过程，属于编译原理的范畴。框架在nemu/src/monitor/debug/expr.c中已经给出，我们需要完善的内容如下：

```c
/* PA1.4 表达式求值_task1_1: 补充token类型*/
enum {
  TK_NOTYPE = 256,    // spaces
  TK_EQ,              // equal
  TK_NEQ,             // not equal
  TK_NUM,             // number
  TK_HEX_NUM,         // hex number
  TK_AND,             // and
  TK_OR,              // or
  TK_REG,             // register
};
```

```c
static struct rule {
  char *regex;
  int token_type;
} rules[] = {
  // PA1.4 表达式求值_task1_2: 补充规则,词法分析
  {" +", TK_NOTYPE},    // spaces
  {"\\*", '*'},         // multiply
  {"\\/", '/'},         // divide
  {"\\+", '+'},         // plus
  {"\\-", '-'},         // minus
  {"==", TK_EQ},        // equal
  {"!=", TK_NEQ},       // not equal
  {"0x[0-9a-fA-F]+", TK_HEX_NUM}, // hex number
  {"[0-9]+", TK_NUM},   // number
  {"\\(", '('},         // left bracket
  {"\\)", ')'},         // right bracket
  {"&&", TK_AND},       // and
  {"\\|\\|", TK_OR},    // or
  {"\\$[a-zA-Z]+", TK_REG}, // register
};
```

```c
/* PA1.4 表达式求值_task1_3: 补充token类型
 * tokens数组中的每一个元素都是一个token结构体，包含两个成员：
 * 1. type: 表示token的类型，是一个整数，可以是上面定义的TK_NOTYPE、TK_EQ等
 * 2. str: 表示token的字符串，是一个字符数组，用于存储token的字符串
*/
switch (rules[i].token_type) {
      // TK_NOTYPE:break;
  case TK_NOTYPE:break;
  case TK_NUM: 
  {
    if (substr_len>32) { puts("The length of number is too long!"); return false; }
          tokens[nr_token].type='0';
    strncpy(tokens[nr_token].str,substr_start,substr_len);
    tokens[nr_token].str[substr_len]='\0';
    ++nr_token;
    break;
  }
  case TK_HEX_NUM:
  {
    if (substr_len>32) { puts("The length of number is too long!"); return false; }
    tokens[nr_token].type='6';
    strncpy(tokens[nr_token].str,substr_start+2,substr_len-2);
    tokens[nr_token].str[substr_len-2]='\0';
    ++nr_token;
    break;
  }

  // brackets
  case '(':
  case ')':

  // operators
  case '*':
  case '/':
  case '-':
  case '+':
  case TK_EQ:
  case TK_NEQ:
  case TK_AND:
  case TK_OR:{tokens[nr_token++].type=rules[i].token_type;break;}

  case TK_REG:
  {
    tokens[nr_token].type='r';  
    strncpy(tokens[nr_token].str,substr_start+1,substr_len-1);
    tokens[nr_token].str[substr_len-1]='\0';
    ++nr_token;
    break;
  }
  default: TODO();
}
```

以上的部分，成功将输入的长表达式拆分成若干个token。接下来就是根据这些拆分完毕的token进行相应的数据运算了。

#### 1.4.2 递归求值（包含扩展）

根据BNF定义, 一种解决方案已经逐渐成型了: 既然长表达式是由短表达式构成的, 我们就先对短表达式求值, 然后再对长表达式求值。 这种十分自然的解决方案就是分治法（一种非常经典的算法）。关于表达式求值的代码框架已经给出，我们需要将其具体实现。详细一点来说，需要进行两个步骤：

1. 是检查括号匹配性，保证表达式的合法性，不合法则不需要再继续进行；
2. 对当前唯一的运算式进行表达式求值。需要注意：

> 表达式有二元也有一元的，需要针对具体形式进行具体的分析。在此之前需要进行一次token的处理，即将单目运算符的'\*'（指针解引用）与双目运算符的'\*'（乘法运算符）区分开来。只需计算一次，因此可以在expr函数中，完成了make_token之后做这件事。

代码如下：

```c
/* PA1.4 表达式求值_task2_1: 检查括号匹配
 * 检查表达式中的括号是否匹配
*/
uint32_t check_parentheses(int start, int end){
  if (tokens[start].type != '(' || tokens[end].type != ')') {
    return false;
  }
  int i, cnt = 0;
  for (i = start + 1; i < end; i++) {
    if (tokens[i].type == '(') {
      cnt++;
    }
    else if (tokens[i].type == ')') {
      cnt--;
    }
    if (cnt < 0) {
      return false;
    }
  }
  return true;
}
```

```c
// PA1.4 表达式求值_task2_2_1: 定义运算符优先级
uint32_t operator_priority(int type) {
  switch (type) {
    case TK_OR: return 1;
    case TK_AND: return 2;
    case TK_EQ: case TK_NEQ: return 3;
    case '+': case '-': return 4;
    case '*': case '/': return 5;
    case TK_DEREF: return 6;
    default: return 0;
  }
}
```

```c
// PA1.4 表达式求值_task2_2_2: 寻找主运算符
uint32_t main_operator(int start, int end) {
  int i, cnt = 0, main_op = -1;
  uint32_t main_pri = 10;
  for (i = start; i <= end; i++) {
    if (tokens[i].type == '(') {
      cnt++;
    }
    else if (tokens[i].type == ')') {
      cnt--;
    }
    if (cnt == 0) {
      if (tokens[i].type == '+' || tokens[i].type == '-' || tokens[i].type == '*' || tokens[i].type == '/' 
                || tokens[i].type == TK_EQ || tokens[i].type == TK_NEQ || tokens[i].type == TK_AND || tokens[i].type == TK_OR
                || tokens[i].type == TK_DEREF) 
      {
        if (operator_priority(tokens[i].type) < main_pri) {
          main_pri = operator_priority(tokens[i].type);
          main_op = i;
        }
      }
    }
  }
  return main_op;
}
```

```c
// PA1.4 表达式求值_task2_3: 表达式求值
uint32_t eval(int start, int end, bool *success) {
  if (start > end) {
    /* Bad expression */
    *success = false;
    return 0;
  }
  else if (start == end) {
    // Single token.
    // printf("now token type is %c\n", tokens[start].type);
    // printf("now token is %s\n", tokens[start].str);
    int result = 0;
    if(tokens[start].type == '0'){
      result = atoi(tokens[start].str);
    }
    else if(tokens[start].type == '6'){
      result = strtol(tokens[start].str, NULL, 16);
    }
    else if(tokens[start].type == 'r'){
      result = isa_reg_str2val(tokens[start].str, success);
      // printf("now the address is 0x%x\n", result);
    }
    else assert(0);
    if (*success == false) {
      return 0;
    }
    // printf("now result is %d\n", result);
    return result;
  }
  else if (check_parentheses(start, end) == true) {
    // The expression is surrounded by a matched pair of parentheses.
    return eval(start + 1, end - 1, success);
  }
  else {
    // Find the dominant operator in the token expression.
    int op = main_operator(start, end);
    if (op == -1) {
      // printf("no main operator found\n");
      *success = false;
      return 0;
    }
    // PA1.4 表达式求值_task2_4: 补充对指针解引用的处理
    else if(tokens[op].type == TK_DEREF){
      uint32_t val = eval(op + 1, end, success);
      if (*success == false) {
        return 0;
      }
      return vaddr_read(val, 4);
    }
    uint32_t val1 = eval(start, op - 1, success);
    uint32_t val2 = eval(op + 1, end, success);
    if (*success == false) {
      return 0;
    }
    // printf("now operator is %c\n", tokens[op].type);
    switch (tokens[op].type) {
      case '+': return val1 + val2;
      case '-': return val1 - val2;
      case '*': return val1 * val2;
      case '/': return val1 / val2;
      case TK_EQ: return val1 == val2;
      case TK_NEQ: return val1 != val2;
      case TK_AND: return val1 && val2;
      case TK_OR: return val1 || val2;
      default: assert(0);
    }
    
  }
}
```

```c
uint32_t expr(char *e, bool *success) {
  if (!make_token(e)) {
    *success = false;
    return 0;
  }
  // PA1.4 表达式求值_task2_4: 补充对指针解引用的处理
  for(int i=0;i<nr_token;++i){
    if(tokens[i].type=='*'&&(i==0||(tokens[i-1].type!=')'&&tokens[i-1].type!='0'&&tokens[i-1].type!='6'&&tokens[i-1].type!='r'))){
      tokens[i].type=TK_DEREF;
    }
  }
  *success = true;
  return eval(0, nr_token - 1, success);

  return 0;
}
```

OK了！

#### 1.4.3 补充ui.c中函数框架

不必多言，和前面一样。

```c
static int cmd_p(char *args) {
  char *arg = strtok(NULL, " ");
  bool success = true;
  uint32_t result = expr(arg, &success);
  if(success) {
    printf("%s = %u\n", arg, result);
  }
  else {
    printf("Invalid expression!\n");
  }
  return 0;
}
```

至此，表达式求值（包含扩展）完成！

~~至于要不要补充测试代码……看后面进度orz~~

### PA1.5 监视点

文档中关于扩展表达式求值的内容已经在前一部分的内容中完成了补充和说明。详见前文。

在`nemu/include/monitor/watchpoint.h`中定义了监视点的数据结构，我们需要补充数据成员以方便后续的操作。这里添加了三个数据成员，具体如下：

```c
typedef struct watchpoint {
  int NO;
  struct watchpoint *next;

  /* TODO: Add more members if necessary */
  char expr[65536];             // watchpoint expression
  bool valueChanged;            // if the value of the watchpoint has changed
  uint32_t oldValue,nowValue;   // the value of the watchpoint

} WP;
```

由此来构成一个监视点的结点数据结构。易知，监视点列表是由单向链表构成的。以此作为线索，需要定义两个函数new_wp()和free_wp()，分别用于创建一个监视点以及释放一个监视点。这一部分就是单链表的操作，比较简单，需要注意的是定义了两个链表`*head, *free_`分别用于存储正在使用的监视点和空闲监视点，在创建和删除的时候需要分别进行相应的修改。代码如下。

```c
WP* new_wp(char* expre) {
  if (free_ == NULL) { assert(0); return NULL; }
  WP* res = free_;
  free_ = free_->next;
  res->next = head;
  head = res;
  if(strlen(expre) >= strlen(res->expr)) {assert("expression too long");}
  strcpy(res->expr, expre);

  bool success = true;
  res->nowValue = res->oldValue = expr(res->expr, &success);
  if(!success) {assert("Wrong expression");}

  return res;
}
```

```c
bool free_wp(int No){
  WP* p = head;
  WP* pre = NULL;
  while(p != NULL) {
    if(p->NO == No) {
      if(pre == NULL) {
        head = p->next;
      }
      else {
        pre->next = p->next;
      }
      p->next = free_;
      free_ = p;
      return true;
    }
    pre = p;
    p = p->next;
  }
  return false;
}
```

接下来需要做的事是实现监视点的相关功能。监视点的功能应该包括显示其表达式的值，以及监视点表达式的值发生变化时让程序因触发了监视点而暂停下来。因此需要再添加两个函数：`watchpoint_display()`用于打印监视点的信息（这个函数用于操作info w时的调用，这是容易得知的），`check_wp()`则用来检查监视点的表达式值是否发生改变（如果发生了改变输出相应的提示信息）。

```c
void watchpoint_display() {
  WP* p = head;
  while(p != NULL) {
    printf("watchpoint %d: %s\n", p->NO, p->expr);
    printf("old value: %u\n", p->oldValue);
    printf("new value: %u\n", p->nowValue);
    p = p->next;
  }
}
```

```c
bool check_wp() {
  WP* p = head;
  bool flag = false;
  while(p != NULL) {
    bool success = true;
    p->nowValue = expr(p->expr, &success);
    if(!success) {assert("Wrong expression");}
    if(p->nowValue != p->oldValue) {
      flag = true;
      printf("Hit watchpoint!\n");
      printf("watchpoint %d: %s\n", p->NO, p->expr);
      printf("old value: %u\n", p->oldValue);
      printf("new value: %u\n", p->nowValue);
      p->oldValue = p->nowValue;
    }
    p = p->next;
  }
  return flag;
}

```

`check_up()`函数的调用位置也值得思考。每当`cpu_exec()`执行完一条指令, 就对所有待监视的表达式进行求值。需要将`nemu_state.state`变量设置为`NEMU_STOP`来达到暂停的效果。由此其添加的位置也比较好确定了，即在`nemu/src/monitor/cpu-exec.c`中：

```c
    /* TODO: check watchpoints here. */
  if(check_wp()) {
    nemu_state.state = NEMU_STOP;
  }
```

使用`d`命令来删除监视点, 你只需要释放相应的监视点结构即可，因此可以直接调用free_wp。

最后，将`nemu/src/monitor/debug/ui.c`中的相应函数补充完毕：

```c
static int cmd_w(char *args) {
  char *arg = strtok(NULL, " ");
  if(arg == NULL){
    printf("Unknown command, please input the expression!\n");
    return 0;
  }
  WP* wp = new_wp(arg);
  printf("Set watchpoint %d for %s\n", wp->NO, wp->expr);
  return 0;
}
```

```c
static int cmd_d(char *args) {
  char *arg = strtok(NULL, " ");
  if(arg == NULL){
    printf("Unknown command, please input the watchpoint number!\n");
    return 0;
  }
  int n = atoi(arg);
  if(free_wp(n)){
    printf("Delete watchpoint %d\n", n);
  }
  else{
    printf("No watchpoint %d\n", n);
  }
  return 0;
}
```

### PA1-- 结语--

首先补充一下help（帮助文档）的内容：

```c
static struct {
  char *name;
  char *description;
  int (*handler) (char *);
} cmd_table [] = {
  { "help", "Display informations about all supported commands", cmd_help },
  { "c", "Continue the execution of the program", cmd_c },
  { "q", "Exit NEMU", cmd_q },
  { "si", "Execute N instructions in a single step", cmd_si},
  { "info", "Print the state of the program",cmd_info},
  { "x", "Scan memory", cmd_x},
  { "p", "Evaluate the expression", cmd_p},
  { "w", "Set watchpoint", cmd_w},
  { "d", "Delete watchpoint", cmd_d},

  /* TODO: Add more commands */

};
```

## PA2
### PA2.1 RTFSC，运行第一个C程序
我们真正需要完成补充的实际上只有为`lui, auipc, addi, jal, jalr`这几个，剩下的都是伪指令，事实上执行的事已经列出的这几种。接下来又要进行痛苦的STFSC和STFM了，因为我们完成了指令类型的判断，要开始进行后面辅助函数的补充了，而这需要我们知道要在哪写。

顺着藤摸了半天瓜，终于在`nemu/src/isa/riscv32/decode.c`和`nemu/src/isa/riscv32/exec/compute.c`摸到了想吃的瓜。前者是译码辅助函数，后者是执行辅助函数。瓜是找到了，吃起来可真不容易！

##### (1) lui指令

我们还是从lui指令开始分析，因为它是给出的参考框架。

`idex()`中的参数`e`为`IDEX(U,lui)`，我们接下来去`idex()`函数中分析，会发现需要用到`DHelper`和`EHelper`两个参数成员和相关调用。前者用于完成后续的译码工作，后者则用于辅助执行工作。

我们来到`nemu/src/isa/riscv32/decode.c`中，可以看到U型指令的操作数译码已经完成了。观察这个函数：
```c
make_DHelper(U)
{
  decode_op_i(id_src, decinfo.isa.instr.imm31_12 << 12, true);
  decode_op_r(id_dest, decinfo.isa.instr.rd, false);
  print_Dop(id_src->str, OP_STR_SIZE, "0x%x", decinfo.isa.instr.imm31_12);
}
```
基本还是可以比较清楚地知道它的作用，实际上就是对U型指令的两个操作数（立即数和寄存器）进行译码处理，辅助译码过程的完成。

接下来到`nemu/src/isa/riscv32/exec/compute.c`里，观察：
```c
make_EHelper(lui)
{
  rtl_sr(id_dest->reg, &id_src->val, 4);
  print_asm_template2(lui);
}
```
这个函数显然是辅助执行的。经过RTFM，我们知道实际上它就是完成lui的执行操作的。以上就是lui的整个执行过程。其余的指令也这样分析即可。

##### (2) auipc指令

U型指令，译码辅助函数已经给出来了，前面lui已经利用了。所以我们要做的就是完成auipc指令的实现。仿照lui指令相关的函数实现，根据auipc指令的规则来编写代码：

```c
make_EHelper(auipc) {
  rtl_add(&id_dest->val, &cpu.pc, &id_src->val);
  rtl_sr(id_dest->reg, &id_dest->val, 4);
  print_asm_template2(auipc);
}
```

auipc指令，很快搞定了。

##### (3) addi指令

I型指令。诶，出现了一个新的指令类型。其实如果对riscv熟悉的朋友应该已经开始有了一些警惕意识：I型指令（同样也包括S型指令，R型指令等等）数量很多，且是具有一些相似性的。如果一个一个写会不会产生什么严重后果，比如代码写出个几百上千行差不多的东西……OK，这个问题先放着，后面会处理的。我们先把运行第一个I型指令，最基础的addi指令给搞定。

不幸的是，I型指令的译码辅助函数没给，我们仿照`decode.c`里的函数来补充（如果对此没头绪，要结合PA2.0里的指令格式仔细分析）。

> 重要提醒！！！这里的函数，需要在`nemu/src/isa/riscv32/include/isa/decode.h`进行定义，不然用不了，识别不出来！一个非常简单的定义卡了我一万年……OTZ

```c
// pa2 added for I-type instructions
make_DHelper(I) {
  decode_op_r(id_src, decinfo.isa.instr.rs1, true);
  decode_op_i(id_src2, decinfo.isa.instr.simm11_0, true);
  print_Dop(id_src->str, OP_STR_SIZE, "%d(%s)", id_src2->val, reg_name(id_src->reg, 4));
  decode_op_r(id_dest, decinfo.isa.instr.rd, false);
}
```

接下来我们来搞定执行辅助函数：

```c
// PA2.1_3 addi
// 备注：这里是为了PA2.1的结果迅速得出，后面重构I型指令时非常有可能会删除这个函数
make_EHelper(addi) {
  rtl_add(&id_dest->val, &id_src->val, &id_src2->val);
  rtl_sr(id_dest->reg, &id_dest->val, 4);
  print_asm_template3(addi);
}
```

##### (4) jal指令

J型指令。又是一种新的指令类型。补充译码辅助函数：

```c
// pa2 added for J-type instructions
make_DHelper(J){
  int32_t offset = decinfo.isa.instr.simm20<<20 | decinfo.isa.instr.imm10_1<<1 | decinfo.isa.instr.imm11_<<11 | decinfo.isa.instr.imm19_12<<12;
  decode_op_i(id_src, offset, true);
  print_Dop(id_src->str, OP_STR_SIZE, "0x%x", offset);
  decode_op_r(id_dest, decinfo.isa.instr.rd, false);
}
```

补充执行辅助函数：

```c
// PA2.1_3: jal指令
make_EHelper(jal) {
  rtl_sr(id_dest->reg, &decinfo.seq_pc, 4);
  rtl_j(decinfo.seq_pc + id_src->val);

  print_asm_template2(jal);
}
```

##### (5) jalr指令

I型指令。

> 熟悉riscv指令集的小伙伴知道，jalr虽然归类于I型指令，但其实由于它改变了PC的值，所以我们一般将它归类在执行控制的指令中。

补充执行辅助函数：

> 关于辅助函数的补充说明：你可能不太清楚`difftest_skip_dut`的作用，可以在此处做一个标记，PA2.3会回来解答你的疑惑。

```c
// pa2.1_5 jalr
make_EHelper(jalr){
  uint32_t addr = cpu.pc + 4;
  rtl_sr(id_dest->reg, &addr, 4);

  decinfo.jmp_pc = (id_src->val+id_src2->val)&(~1);
  rtl_j(decinfo.jmp_pc);

  difftest_skip_dut(1, 2); //difftest_skip_dut(1, 2)表示跳过1个周期，2表示跳过的指令数

  print_asm_template2(jalr);
}
```

### PA2.2 完善AM，实现更多程序
#### 2.2.1 补充ID/EX辅助函数
首先补充B, R, I型指令中对应的译码辅助函数。
##### (1) B型指令

观察B型指令操作码，发现opcode全都是相同的，因此都处于同一个`opcode_table`中，具体位置为序号18。在此不再单独列出，最后进行汇总的补充。

译码辅助函数补充如下：

```c
// pa2 added for B-type instructions
make_DHelper(B)
{
  decode_op_r(id_src, decinfo.isa.instr.rs1, true);
  decode_op_r(id_src2, decinfo.isa.instr.rs2, true);
  print_Dop(id_src->str, OP_STR_SIZE, "%s", reg_name(id_src->reg, 4));
  print_Dop(id_src2->str, OP_STR_SIZE, "%s", reg_name(id_src2->reg, 4));

  int32_t offset = decinfo.isa.instr.simm12<<12 | decinfo.isa.instr.imm10_5<<5 | decinfo.isa.instr.imm4_1<<1 | decinfo.isa.instr.imm11<<11;
  decode_op_i(id_dest, offset, true);

}
```

接下来是重要的执行辅助函数。观察所有B型指令，可以发现区分它们的是14~12位的funct3操作数。这样就可以将每条指令的执行操作分开了，之后再根据各条指令的不同意义编写代码即可。仔细观察B型指令，可以发现都是通过一次比较，从而决定是否对对应的offset值完成跳转。对应修改，完成此部分。

```c
// PA2.2 B-type instructions
make_EHelper(B_ir_18){
  decinfo.jmp_pc = cpu.pc + id_dest->val;
  switch(decinfo.isa.instr.funct3){
    case 0b000:     // beq||beqz
      rtl_jrelop(RELOP_EQ, id_src->val, id_src2->val, decinfo.jmp_pc);
      print_asm_template3(beq);
      break;
    case 0b001:     // bne||bnez
      rtl_jrelop(RELOP_NE, id_src->val, id_src2->val, decinfo.jmp_pc);
      print_asm_template3(bne);
      break;
    case 0b100:     // blt||bltz
      rtl_jrelop(RELOP_LT, id_src->val, id_src2->val, decinfo.jmp_pc);
      print_asm_template3(blt);
      break;
    case 0b101:     // bge||bgez
      rtl_jrelop(RELOP_GE, id_src->val, id_src2->val, decinfo.jmp_pc);
      print_asm_template3(bge);
      break;
    case 0b110:     // bltu
      rtl_jrelop(RELOP_LTU, id_src->val, id_src2->val, decinfo.jmp_pc);
      print_asm_template3(bltu);
      break;
    case 0b111:   // bgeu
      rtl_jrelop(RELOP_GEU, id_src->val, id_src2->val, decinfo.jmp_pc);
      print_asm_template3(bgeu);
      break;
    default:
      assert(0);
      break;
  }
}
```

##### (2) I型指令

I型指令在opcode中体现为三种，但是通过观察指令格式可以发现，实际上只有opcode不一样。第一种opcode为`0000011`，其实是半字、字节等的存取指令，归进了ld/st指令中；第二种opcode为`0010011`，也就是addi所在的位置，这将是我们编写I型指令代码的重点关注部分；最后一种opcode为`1110011`，程序中应该是没有这一部分的相应指令的。综上，译码辅助函数不需要再额外进行一次编写。我们检查一下所有需要实现的I型指令（其中jalr作为一个特例，不计入更大范围内的I型指令）：

| 指令  | funct3 | funct7         |
| ----- | ------ | -------------- |
| addi  | 000    | 包含于立即数中 |
| slti  | 010    | 包含于立即数中 |
| sltiu | 011    | 包含于立即数中 |
| xori  | 100    | 包含于立即数中 |
| ori   | 110    | 包含于立即数中 |
| andi  | 111    | 包含于立即数中 |
| slli  | 001    | 包含于立即数中 |
| srli  | 101    | 0000000        |
| srai  | 101    | 0100000        |

这下很明确了。针对它们完成函数的改编即可。（要记得前文`opcode_table`的addi需要改成I型指令，不然识别不出其余的了）

```c
// PA2.2 I-type instructions
make_EHelper(I_ir_4) {
  switch (decinfo.isa.instr.funct3) {
    case 0b000: // addi
      rtl_add(&id_dest->val, &id_src->val, &id_src2->val);
      break;
    case 0b010: // slti
      id_dest->val = (int32_t)id_src->val < (int32_t)id_src2->val;
      break;
    case 0b011: // sltiu
      id_dest->val = (unsigned)id_src->val < (unsigned)id_src2->val;
      break;
    case 0b100: // xori
      rtl_xor(&id_dest->val, &id_src->val, &id_src2->val);
      break;
    case 0b110: // ori
      rtl_or(&id_dest->val, &id_src->val, &id_src2->val);
      break;
    case 0b111: // andi
      rtl_and(&id_dest->val, &id_src->val, &id_src2->val);
      break;
    case 0b001: // slli
      rtl_shl(&id_dest->val, &id_src->val, &id_src2->val);
      break;
    case 0b101:
      if((decinfo.isa.instr.funct7) == 0b0000000) // srli
        rtl_shr(&id_dest->val, &id_src->val, &id_src2->val);
      else // srai
        rtl_sar(&id_dest->val, &id_src->val, &id_src2->val);
      break;
    default:
      assert(0);
  }
  rtl_sr(id_dest->reg, &id_dest->val, 4);
  print_asm_template3(I_ir_4);
}
```

##### (3) R型指令

R型指令的opcode都是`0110011`，对应进`opcode_table`就是二进制的01100，很容易定位。至此完成了`opcode_table`的补充：

```c
static OpcodeEntry opcode_table [32] = {
  /* b00 */ IDEX(ld, load), EMPTY, EMPTY, EMPTY, IDEX(I, Imm), IDEX(U, auipc), EMPTY, EMPTY,
  /* b01 */ IDEX(st, store), EMPTY, EMPTY, EMPTY, IDEX(R, Reg_2), IDEX(U, lui), EMPTY, EMPTY,
  /* b10 */ EMPTY, EMPTY, EMPTY, EMPTY, EMPTY, EMPTY, EMPTY, EMPTY,
  /* b11 */ IDEX(B, Branch), IDEX(I, jalr), EX(nemu_trap), IDEX(J, jal), EMPTY, EMPTY, EMPTY, EMPTY,
};
```

然后是R型指令的辅助译码函数。这里涉及的指令特别多，建议仔细阅读和识别。

```c
// pa2 added for R-type instructions
make_DHelper(R)
{
  decode_op_r(id_src, decinfo.isa.instr.rs1, true);
  decode_op_r(id_src2, decinfo.isa.instr.rs2, true);
  print_Dop(id_src->str, OP_STR_SIZE, "%s", reg_name(id_src->reg, 4));
  print_Dop(id_src2->str, OP_STR_SIZE, "%s", reg_name(id_src2->reg, 4));
  decode_op_r(id_dest, decinfo.isa.instr.rd, false);
}
```

R型指令辅助执行函数：

```c
// PA2.2 R-type instructions
make_EHelper(Reg_2){
  switch (decinfo.isa.instr.funct3){
  case 0b000: {
    if(decinfo.isa.instr.funct7 == 0x00){       // add
      rtl_add(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(add);
      
    }
    else if(decinfo.isa.instr.funct7 == 0x20){  // sub
      rtl_sub(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(sub);
    }
    else{                                       // mul
      rtl_imul_lo(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(mul);
    }
    break;
  }
  case 0b001: {
    if(decinfo.isa.instr.funct7 == 0x00){       // sll
      rtl_shl(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(sll);

    }
    else{                                       // mulh
      rtl_imul_hi(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(mulh);
    }
    break;
  }
  case 0b010: {
    if(decinfo.isa.instr.funct7 == 0x00){       // slt
      id_dest->val = (signed)id_src->val < (signed)id_src2->val;
      print_asm_template3(slt);
    }
    else{                                       // mulhsu
      TODO();
    }
    break;
  }
  case 0b011: {
    if(decinfo.isa.instr.funct7 == 0x00){       // sltu
      id_dest->val = (unsigned)id_src->val < (unsigned)id_src2->val;
      print_asm_template3(sltu);
    }
    else{                                       // mulhu
      TODO();
    }
    break;
  }
  case 0b100: {
  if(decinfo.isa.instr.funct7 == 0x00){         // xor
      rtl_xor(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(xor);
    }
  else{                                         // div
      rtl_idiv_q(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(div);
    }
    break;
  }
  case 0b101: {
    if(decinfo.isa.instr.funct7 == 0x00){       // srl
      rtl_shr(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(srl);
    }
    else if(decinfo.isa.instr.funct7 == 0x20){  // sra
      rtl_sar(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(sra);
    }
    else{                                       // divu
      rtl_div_q(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(divu);
    }
    break;
  }
  case 0b110: {
    if(decinfo.isa.instr.funct7 == 0x00){       // or
      rtl_or(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(or);
    }
    else{                                       // rem
      rtl_idiv_r(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(rem);
    }
    break;
  }
case 0b111: {                                       
    if(decinfo.isa.instr.funct7 == 0x00){       // and
      rtl_and(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(and);
    }
    else{                                       // remu
      rtl_div_r(&id_dest->val, &id_src->val, &id_src2->val);
      print_asm_template3(remu);
    }
    break;
  }
  default: break;
  }
  rtl_sr(id_dest->reg, &id_dest->val, 4);
}
```

##### (4) Load/Store指令

存取指令。其实存取指令的代码已经几乎比较完善了，但还差一点。仔细观察会发现：

1. `load_table`和`store_table`怎么显得这么空？是不是还有数据成员没有完善？
2. 待测试的程序如果你现在再跑一下看看，不出意外`load-store.c`这个程序是还没有pass的。

好了，问题发现了，需要针对这个来进行修改。经过痛苦的RTFSC、RTFM和一些阅读技巧，你发现了你需要针对`nemu/src/isa/riscv32/exec/ldst.c`中的内容进行针对性补充，另外还发现了需要再去实现I型指令中我们略过去的第一种。哈哈！没想到吧，在这等着呢。你需要手动补充完毕`load_table`和`store_table`的内容，另外还需要把`lb, lh`两种指令进行针对性补充。不过，已经到了这里，这些想必已经不是难题。这里给出`load_table`和`store_table`的内容，至于其余的辅助函数，该给的基本给了，你一定可以自己写出来的。

```c
static OpcodeEntry load_table [8] = {
  EXW(lb,1), EXW(lh,2), EXW(ld, 4), EMPTY, EXW(ld, 1), EXW(ld, 2), EMPTY, EMPTY
};
static OpcodeEntry store_table [8] = {
  EXW(st, 1), EXW(st, 2), EXW(st, 4), EMPTY, EMPTY, EMPTY, EMPTY, EMPTY
};
```

#### 2.2.2 补充其余函数
> ##### 实现字符串处理函数
>
> 根据需要实现`nexus-am/libs/klib/src/string.c`中列出的字符串处理函数, 让测试用例`string`可以成功运行. 关于这些库函数的具体行为, 请务必RTFM.

> ##### 实现sprintf
>
> 为了运行测试用例`hello-str`, 你还需要实现库函数`sprintf()`. 实现`nexus-am/libs/klib/src/stdio.c`中的`sprintf()`, 具体行为可以参考`man 3 printf`. 目前你只需要实现`%s`和`%d`就能通过`hello-str`的测试了, 其它功能(包括位宽, 精度等)可以在将来需要的时候再自行实现.

好了，所有的重要信息都在这里了。接下来就是搓代码时间了。`string.c`异常简单，代码甚至已经手搓过很多次，就不给了。`stdio.c`则有一些难度，不过你在最开始的时候只需要解决%s和%d就可以通过hello-str测试了。这贴出最终的代码，里面有很多东西是后面涉及的，建议你先不要全部抄下来，而是在理解的基础上自己构建代码。

```c
#include "klib.h"
#include <stdarg.h>

#if !defined(__ISA_NATIVE__) || defined(__NATIVE_USE_KLIB__)


void add_string(char **s, char *str);
void add_char(char **s, char c);
void add_number(char **s, int num,char mode);
void add_special_number(char **s, const char *fmt, va_list ap,char c);

int printf(const char *fmt, ...) {
  va_list ap;
  va_start(ap,fmt);
  char buf[1000];
  int length=vsprintf(buf,fmt,ap);
  buf[length]='\0';
  for(int i=0;i<length;i++){
    _putc(buf[i]);
  }
  va_end(ap);
  return 0;
}

int vsprintf(char *out, const char *fmt, va_list ap) {
  char *temp=out;
  while(*fmt!='\0'){
    if(*fmt=='%'){
      fmt++;
      switch(*fmt){
        case 's':{
          char *str=va_arg(ap,char*);
          add_string(&temp,str);
          break;
        }
        case 'd':{
          int num=va_arg(ap,int);
          add_number(&temp,num,'d');
          break;
        }
        case 'x':{
          int num=va_arg(ap,int);
          add_number(&temp,num,'x');
          break;
        }
        case '0':{
          fmt++;
          add_special_number(&temp,fmt,ap,'0');
          fmt++;
          break;
        }
        case '1':
        case '2':
        case '3':
        case '4':
        case '5':
        case '6':
        case '7':
        case '8':
        case '9':{
          add_special_number(&temp,fmt,ap,' ');
          fmt++;
          break;
        }
        case 'c':{
          char c=va_arg(ap,int);
          add_char(&temp,c);
          break;
        }
      }
      fmt++;
    }
    else{
      *temp++=*fmt++;
    }
  }
  *temp='\0';
  return temp-out;
}

int sprintf(char *out, const char *fmt, ...) {
  va_list ap;
  va_start(ap,fmt);
  int length=vsprintf(out,fmt,ap);
  va_end(ap);
  return length;
}

int snprintf(char *out, size_t n, const char *fmt, ...) {
  return 0;
}


void add_string(char **s, char *str) {
  while (*str!='\0') {
    **s = *str;
    (*s)++;
    str++;
  }
}

void add_char(char **s, char c) {
  **s = c;
  (*s)++;
}

void add_number(char **s, int num,char mode) {
  if (num == 0) {
    add_char(s, '0');
    return;
  }
  char temp[100];
  int i = 0;
  if(num<0){
    add_char(s,'-');
    num=-num;
  }
  if(mode=='d'){
    while (num) {
      temp[i] = num % 10 + '0';
      num /= 10;
      i++;
    }
    while (i) {
      add_char(s, temp[i - 1]);
      i--;
    }
  }
  else if(mode=='x'){
    add_string(s,"0x");
    while (num) {
      temp[i] = num % 16 + (num % 16 < 10 ? '0' : 'a' - 10);
      num /= 16;
      i++;
    }
    while (i) {
      add_char(s, temp[i - 1]);
      i--;
    }
  }
}


void add_special_number(char **s, const char *fmt, va_list ap,char c) {
  int num_of_digit=0;
  while(*fmt!='d'&&*fmt!='x'){
    num_of_digit=num_of_digit*10+*fmt-'0';
    fmt++;
  }
  int num = va_arg(ap, int);
  char temp[100];
  if(*fmt=='d'){
    int k = num==0?1:0;
    if(num<0){
      add_char(s,'-');
      num=-num;
    }
    while (num) {
      temp[k] = num % 10 + '0';
      num /= 10;
      k++;
    }
    int i=k;
    while(i<num_of_digit){
      add_char(s,c);
      i++;
    }
    while(k){
      add_char(s,temp[k-1]);
      k--;
    }
  }
  else if(*fmt=='x'){
    int k = num==0?1:0;
    while (num) {
      temp[k] = num % 16 + (num % 16 < 10 ? '0' : 'a' - 10);
      num /= 16;
      k++;
    }
    int i=k;
    while(i<num_of_digit){
      add_char(s,c);
      i++;
    }
    while(k){
      add_char(s,temp[k-1]);
      k--;
    }
  }
}

#endif
```

### PA2.3 输入输出
#### 2.3.0 设备（准备工作）
修改文件，打开HAS_IOE宏定义。
```c
/* You will define this macro in PA2 */
#define HAS_IOE
```
代码是模块化的, 要在NEMU中加入设备的功能, 你只需要在`nemu/include/common.h`中定义宏`HAS_IOE`. 定义后, `init_device()`函数会对设备进行初始化. 重新编译后, 你会看到运行NEMU时会弹出一个新窗口, 用于显示VGA的输出(见下文). 需要注意的是, 终端显示的提示符`(nemu)`仍然在等待用户输入, 此时窗口并未显示任何内容.

#### 2.3.1 输入输出内容补充
##### (1) 串口

我们的实验选择的是ISA是riscv32，因此串口部分不需要额外编写其它代码，保证前文实验内容正确的前提下就可以通过在`nexus-am/tests/amtest`目录下输入

```shell
make mainargs=h run
```

就能得到十行"hello, world"的输出了。

接下来处理`printf()`函数。

> 有了`_putc()`, 我们就可以在klib中实现`printf()`了.
>
> 你之前已经实现了`sprintf()`了, 它和`printf()`的功能非常相似, 这意味着它们之间会有不少重复的代码. 你已经见识到Copy-Paste编程习惯的坏处了, 思考一下, 如何简洁地实现它们呢?

所以，你可以轻松地搓出`printf()`函数的代码了。整个`stdio.c`的文件内容已经贴到了前面，你可以仔细查阅。

串口部分的主要工作就是这些。

##### (2) 时钟

在实现该部分内容之前，如果你有兴趣，可以先在`nexus-am/tests/amtest`目录下跑一下native的时钟：

```shell
make ARCH=native mainargs=t run
```

然后你就可以看到虚拟机的当前时间，以及每秒递增的计数。我们要做的就是用riscv32-nemu也实现这个功能。阅读手册：

> #####  实现IOE
>
> 在`nexus-am/am/src/nemu-common/nemu-timer.c`中实现`_DEVREG_TIMER_UPTIME`的功能. 在`nexus-am/am/include/nemu.h`和`nexus-am/am/include/$ISA.h` 中有一些输入输出相关的代码供你使用.

于是我们在相应的文件中可以完成如下代码的编写：

```c
#include <am.h>
#include <amdev.h>
#include <nemu.h>

static uint32_t boot_time = 0;

size_t __am_timer_read(uintptr_t reg, void *buf, size_t size) {
  switch (reg) {
    case _DEVREG_TIMER_UPTIME: {
      _DEV_TIMER_UPTIME_t *uptime = (_DEV_TIMER_UPTIME_t *)buf;
      // PA2.3 TODO: calculate time using RTC (Real Time Counter)
      uint32_t current_time = inl(RTC_ADDR);
      uptime->hi = 0;
      uptime->lo = current_time - boot_time;
      return sizeof(_DEV_TIMER_UPTIME_t);
    }
    case _DEVREG_TIMER_DATE: {
      _DEV_TIMER_DATE_t *rtc = (_DEV_TIMER_DATE_t *)buf;
      rtc->second = 0;
      rtc->minute = 0;
      rtc->hour   = 0;
      rtc->day    = 0;
      rtc->month  = 0;
      rtc->year   = 2000;
      return sizeof(_DEV_TIMER_DATE_t);
    }
  }
  return 0;
}

void __am_timer_init() {
  // PA2.3 TODO: initialize RTC
  boot_time = inl(RTC_ADDR);
}
```

时钟功能完成。

> 如果你打算现在就对你所实现的时钟功能进行测试并跑分，请注意这条提示：
>
> 跑分时请注释掉`nemu/include/common.h`中的`DEBUG`和`DIFF_TEST`宏, 以获得较为真实的跑分。

##### (3) 键盘

阅读手册：

> #####  实现IOE(2)
>
> 在`nexus-am/am/src/nemu-common/nemu-input.c`中实现`_DEVREG_INPUT_KBD`的功能. 实现后, 在`$ISA-nemu`中运行`amtest`中的`readkey test`测试. 如果你的实现正确, 在程序运行时弹出的新窗口中按下按键, 你将会看到程序输出相应的按键信息, 包括按键名, 键盘码, 以及按键状态.

很清晰地知道了需要在哪里进行编程了。我们来到对应的位置，实现相应功能。

```c
size_t __am_input_read(uintptr_t reg, void *buf, size_t size) {
  switch (reg) {
    case _DEVREG_INPUT_KBD: {
      _DEV_INPUT_KBD_t *kbd = (_DEV_INPUT_KBD_t *)buf;
      // PA2.3 TODO: implement the code to read the current state of keyboard
      int key=inl(KBD_ADDR);
      kbd->keydown = key & KEYDOWN_MASK ? 1 : 0;
      kbd->keycode = key & ~KEYDOWN_MASK;
      return sizeof(_DEV_INPUT_KBD_t);
    }
  }
  return 0;
}
```

##### (4) VGA

阅读手册，这次没有给出非常明确的要做什么的信息，但我们可以概括出来是这样：

> 1. 在`nemu/src/device/vga.c`中完善`vga_io_handler()`函数让硬件(NEMU)支持同步寄存器的使用；
> 2. 由于屏幕大小寄存器硬件已经支持，可以直接使用，同步寄存器前一步完成了支持，将这三个寄存器使用的函数进行声明；
> 3. 在`nexus-am/am/src/nemu-common/nemu-video.c`文件的`__am_vga_init()`函数中添加手册中已经编写好的代码；
> 4. 实现 `_DEVREG_VIDEO_INFO` 和 `_DEVREG_VIDEO_FBCTL` 的功能。

因此所有代码如下。

```c
// function location: nemu/src/device/vga.c
static void vga_io_handler(uint32_t offset, int len, bool is_write) {
  // TODO: call `update_screen()` when writing to the sync register
  // TODO();
    if(is_write){
    update_screen();
  }
}
```

```c
// function location: nexus-am/am/src/nemu-common/nemu-video.c

// PA2.3 added for video
void draw_sync();
int screen_width();
int screen_height();

size_t __am_video_read(uintptr_t reg, void *buf, size_t size) {
  switch (reg) {
    case _DEVREG_VIDEO_INFO: {
      _DEV_VIDEO_INFO_t *info = (_DEV_VIDEO_INFO_t *)buf;
      // PA2.3 TODO: modify the right width and height
      info->width = 400;
      info->height = 300;
      return sizeof(_DEV_VIDEO_INFO_t);
    }
  }
  return 0;
}

size_t __am_video_write(uintptr_t reg, void *buf, size_t size) {
  switch (reg) {
    case _DEVREG_VIDEO_FBCTL: {
      _DEV_VIDEO_FBCTL_t *ctl = (_DEV_VIDEO_FBCTL_t *)buf;
      // PA2.3 TODO: implement the code to write to frame buffer
      int x=ctl->x,y=ctl->y,h=ctl->h,w=ctl->w;
      int W=screen_width();
      int H=screen_height();
      uint32_t *pixels=ctl->pixels;
      uint32_t *fb=(uint32_t *)(uintptr_t)FB_ADDR;
      for(int i=0;i<h;i++){
        for(int j=0;j<w;j++){
          fb[(y+i)*W+x+j]=pixels[i*w+j];
        }
      }
      if (ctl->sync) {
        outl(SYNC_ADDR, 0);
      }
      return size;
    }
  }
  return 0;
}

// PA2.3 TODO: implement the following functions
void __am_vga_init() {
  int i;
  int size = screen_width() * screen_height();
  uint32_t *fb = (uint32_t *)(uintptr_t)FB_ADDR;
  for (i = 0; i < size; i ++) fb[i] = 0;
  draw_sync();
}
```

结束了，PA2.3的代码内容！

## PA3
### PA3.1 穿越时空的旅程：准备工作

#### 3.1.1 触发自陷操作

##### (1) 定义相关宏

> 首先是按照ISA的约定来设置异常入口地址, 将来切换执行流时才能跳转到正确的异常入口. 这显然是架构相关的行为, 因此我们把这一行为放入CTE中, 而不是让Nanos-lite直接来设置异常入口地址. 你需要在`nanos-lite/include/common.h`中定义宏`HAS_CTE`。

以上这一步的内容很明确。将对应位置的宏定义解注释即可。

##### (2) 实现`raise_intr()`函数

这个函数的实现仅仅看设置异常入口地址这一部分的手册可能会很迷茫。前文也要进行仔细阅读，手册没有哪一部分全是废话。

> riscv32提供`ecall`指令作为自陷指令, 并提供一个stvec寄存器来存放异常入口地址. 为了保存程序当前的状态, riscv32提供了一些特殊的系统寄存器, 叫控制状态寄存器(CSR寄存器). 在PA中, 我们只使用如下3个CSR寄存器:
>
> - sepc寄存器 - 存放触发异常的PC
> - sstatus寄存器 - 存放处理器的状态
> - scause寄存器 - 存放触发异常的原因
>
> riscv32触发异常后硬件的响应过程如下:
>
> 1. 将当前PC值保存到sepc寄存器
> 2. 在scause寄存器中设置异常号
> 3. 从stvec寄存器中取出异常入口地址
> 4. 跳转到异常入口地址

所以需要为`ISADecodeInfo`添加数据成员如下（在`nemu/src/isa/x86/include/isa/decode.h`）中：

```c
struct ISADecodeInfo {
  Instr instr;
  uint32_t stvec;
  uint32_t sepc, sstatus, scause;
};
```

然后是`raise_intr()`函数（在`nemu/src/isa/riscv32/intr.c`中）：

```c
#include "cpu/exec.h"
void raise_intr(uint32_t NO, vaddr_t epc) {
  decinfo.isa.sepc = epc;
  decinfo.isa.scause = NO;
  decinfo.jmp_pc = decinfo.isa.stvec;
  rtl_j(decinfo.jmp_pc);
}
```

##### (3) 实现系统调用指令

查表知系统调用指令通用一个opcode：1110011，据此填入`opcode_table`的对应位置：

```c
static OpcodeEntry opcode_table [32] = {
  /* b00 */ IDEX(ld, load), EMPTY, EMPTY, EMPTY, IDEX(I, Imm), IDEX(U, auipc), EMPTY, EMPTY,
  /* b01 */ IDEX(st, store), EMPTY, EMPTY, EMPTY, IDEX(R, Reg_2), IDEX(U, lui), EMPTY, EMPTY,
  /* b10 */ EMPTY, EMPTY, EMPTY, EMPTY, EMPTY, EMPTY, EMPTY, EMPTY,
  /* b11 */ IDEX(B, Branch), IDEX(I, jalr), EX(nemu_trap), IDEX(J, jal), IDEX(SYSTEM,syscall), EMPTY, EMPTY, EMPTY,
};
```

补充译码辅助函数（不要忘记头文件中的内容）：

```c
// pa3 added for system instructions
make_DHelper(SYSTEM)
{
  decode_op_r(id_src, decinfo.isa.instr.rs1, true);
  decode_op_i(id_src2, decinfo.isa.instr.csr, true);
  print_Dop(id_src->str, OP_STR_SIZE, "%d", decinfo.isa.instr.csr);
  decode_op_r(id_dest, decinfo.isa.instr.rd, false);
}
```
> #####   riscv32的特权模式简化
>
> 如果你RTFSC, 你会发现上述CSR寄存器都是S-mode(supervisor-mode, 监控者模式)的, 而标准的riscv32在开机后应该位于M-mode, 因此应该使用M-mode的CSR. 在这里我们选择S-mode, 是为了配合PA4的分页机制而进行的简化, 因此我们在NEMU中实现的并不是标准的riscv32(当然x86也不是), 不过这并不影响你的这些知识的理解. 如果你不理解这一段话的内容, 你可以忽略它.

<img src="https://i-blog.csdnimg.cn/blog_migrate/43ca3cffcd414292f5cf00100b49eb8e.png#pic_center" alt="在这里插入图片描述" style="zoom:67%;" />

所有的代码如下，清楚`csrr`等指令的作用的话应该不太难理解：

> 再给一个提示吧：`ecall`是有关系统调用方面的内容，如果你不是特别清楚，可以查看`navy-apps/libs/libos/src/nanos.c`和`nexus-am/am/include/arch/riscv32-nemu.h`，这样你就知道为什么要用

```c
#include "cpu/exec.h"

extern void raise_intr(uint32_t NO, vaddr_t epc);

int32_t readcsr(int i){
    switch(i){
        case 0x105:
            return decinfo.isa.stvec;
        case 0x142:
            return decinfo.isa.scause;
        case 0x100:
            return decinfo.isa.sstatus;
        case 0x141:
            return decinfo.isa.sepc;
        default:
            assert(0 && "Unfinished readcsr");
    }
}

void writecsr(int i, int32_t val){
    //TODO
    switch(i){
        case 0x105:
            decinfo.isa.stvec = val;
            break;
        case 0x142:
            decinfo.isa.scause = val;
            break;
        case 0x100:
            decinfo.isa.sstatus = val;
            break;
        case 0x141:
            decinfo.isa.sepc = val;
            break;
        default:
            assert(0 && "Unfinished readcsr");
    }
}

make_EHelper(syscall){
    Instr instr = decinfo.isa.instr;
    switch(instr.funct3){
        //ecall
        case 0b0:
            if((instr.val & ~(0x7f))==0){
                raise_intr(reg_l(17), decinfo.seq_pc-4);
            }
            else if(instr.val == 0x10200073){
                decinfo.jmp_pc = decinfo.isa.sepc + 4;
                rtl_j(decinfo.jmp_pc);
            }
            else{
                assert(0 && "system code unfinish");
            }
            break;
        // csrrw
        case 0b001:
            s0 = readcsr(instr.csr);
            writecsr(instr.csr, id_src->val);
            rtl_sr(id_dest->reg, &s0, 4);
            break;
        case 0b010:
            s0 = readcsr(instr.csr);
            writecsr(instr.csr, s0 | id_src->val);
            rtl_sr(id_dest->reg, &s0, 4);
            break;
        default:
            assert(0 && "Unfinished system op");
    }
}
```

#### 3.1.2 保存上下文

首先是完成指令实现，这一部分上一小节已经搞定了。那么就剩下重新组织_Context结构体这个任务了。

> 理解上下文形成的过程并RTFSC, 然后重新组织`nexus-am/am/include/arch/$ISA-nemu.h` 中定义的`_Context`结构体的成员, 使得这些成员的定义顺序和 `nexus-am/am/src/$ISA/nemu/trap.S`中构造的上下文保持一致.

首先看`trap.S`的内容：

```asm

#define concat_temp(x, y) x ## y
#define concat(x, y) concat_temp(x, y)
#define MAP(c, f) c(f)

#define REGS(f) \
      f( 1)       f( 3) f( 4) f( 5) f( 6) f( 7) f( 8) f( 9) \
f(10) f(11) f(12) f(13) f(14) f(15) f(16) f(17) f(18) f(19) \
f(20) f(21) f(22) f(23) f(24) f(25) f(26) f(27) f(28) f(29) \
f(30) f(31)

#define PUSH(n) sw concat(x, n), (n * 4)(sp);
#define POP(n)  lw concat(x, n), (n * 4)(sp);

#define CONTEXT_SIZE ((31 + 4) * 4)
#define OFFSET_SP     ( 2 * 4)
#define OFFSET_CAUSE  (32 * 4)
#define OFFSET_STATUS (33 * 4)
#define OFFSET_EPC    (34 * 4)

.globl __am_asm_trap
__am_asm_trap:
  addi sp, sp, -CONTEXT_SIZE

  MAP(REGS, PUSH)

  mv t0, sp
  addi t0, t0, CONTEXT_SIZE
  sw t0, OFFSET_SP(sp)

  csrr t0, scause
  csrr t1, sstatus
  csrr t2, sepc

  sw t0, OFFSET_CAUSE(sp)
  sw t1, OFFSET_STATUS(sp)
  sw t2, OFFSET_EPC(sp)

  mv a0, sp
  jal __am_irq_handle

  lw t1, OFFSET_STATUS(sp)
  lw t2, OFFSET_EPC(sp)
  csrw sstatus, t1
  csrw sepc, t2

  MAP(REGS, POP)

  addi sp, sp, CONTEXT_SIZE

  sret

```
认真理解一下，不难得出顺序应当是下面这样的：

```c
struct _Context {
  uintptr_t gpr[32], cause, status, epc;
  struct _AddressSpace *as;
};
```

#### 3.1.3 事件分发

> #####  实现正确的事件分发
>
> 1. 在`__am_irq_handle()`中通过异常号识别出自陷异常, 并打包成编号为`_EVENT_YIELD`的自陷事件.
> 2. 在`do_event()`中识别出自陷事件`_EVENT_YIELD`, 然后输出一句话即可, 目前无需进行其它操作.

不难得到（这两个函数在`nexus-am/am/src/riscv32/nemu/cte.c`和`nanos-lite/src/irq.c`）：

```c
// __am_irq_handle()中添加的内容
    switch (c->cause) {
        // PA3 added for system instructions
        case -1:
            ev.event = _EVENT_YIELD;
            break;
```

```c
// do_event()中
  switch (e.event) {
    case _EVENT_YIELD: printf("Yield event happened!\n"); break;
```

#### 3.1.4 恢复上下文

> #####  恢复上下文
>
> 你需要实现这一过程中的新指令. 重新运行Nanos-lite, 如果你的实现正确, 你会看到在`do_event()`中输出的信息, 并且最后仍然触发了`main()`函数末尾设置的`panic()`.

这里其实要做的是完善之前的指令，幸运的是你已经完成了这一步。

### PA3.2 用于程序和系统调用

#### 3.2.1 加载第一个用户程序

> #####  实现loader
>
> 你需要在Nanos-lite中实现loader的功能, 来把用户程序加载到正确的内存位置, 然后执行用户程序. `loader()`函数在`nanos-lite/src/loader.c`中定义, 其中的`pcb`参数目前暂不使用, 可以忽略, 而因为`ramdisk`中目前只有一个文件, `filename`参数也可以忽略.
>
> 实现后, 在`init_proc()`中调用`naive_uload(NULL, NULL)`, 它会调用你实现的loader来加载第一个用户程序, 然后跳转到用户程序中执行. 如果你的实现正确, 你会看到执行`dummy`程序时在Nanos-lite中触发了一个未处理的1号事件. 这说明loader已经成功加载dummy, 并且成功地跳转到dummy中执行了. 关于未处理的事件, 我们会在下文进行说明.

`loader()`函数代码：

```c
static uintptr_t loader(PCB *pcb, const char *filename) {
  // TODO();
  // PA3.2 加载用户程序到正确位置
  Elf_Ehdr ehdr;
  ramdisk_read(&ehdr, 0, sizeof(Elf_Ehdr));
  assert((*(uint32_t *)ehdr.e_ident == 0x464c457f));

  Elf_Phdr phdr[ehdr.e_phnum];
  ramdisk_read(phdr, ehdr.e_phoff, sizeof(Elf_Phdr)*ehdr.e_phnum);
  for (int i = 0; i < ehdr.e_phnum; i++) {
    if (phdr[i].p_type == PT_LOAD) {
      ramdisk_read((void*)phdr[i].p_vaddr, phdr[i].p_offset, phdr[i].p_memsz);
      memset((void*)(phdr[i].p_vaddr+phdr[i].p_filesz), 0, phdr[i].p_memsz - phdr[i].p_filesz);
    }
  }
  return ehdr.e_entry;
}
```

#### 3.2.2 系统调用

##### (1) 识别系统调用

> #####  识别系统调用
>
> 目前`dummy`已经通过`_syscall_()`直接触发系统调用, 你需要让Nanos-lite识别出系统调用事件`_EVENT_SYSCALL`.

`__am_irq_handle`函数补充内容：

```c

_Context* __am_irq_handle(_Context *c) {
  _Context *next = c;
  if (user_handler) {
    _Event ev = {0};
    // printf("c->cause: %d\n", c->cause);
    switch (c->cause) {
        // PA3 added for system instructions
        case -1:
            ev.event = _EVENT_YIELD;
            break;
        case 0:
        case 1:
            ev.event = _EVENT_SYSCALL;
            break;
        default: ev.event = _EVENT_ERROR; break;
    }
```

##### (2) 实现SYS_yield系统调用和SYS_exit系统调用

> ##### 实现SYS_yield系统调用
>
> 你需要:
>
> 1. 在`nexus-am/am/include/arch/$ISA-nemu.h`中实现正确的`GPR?`宏, 让它们从上下文`c`中获得正确的系统调用参数寄存器.
> 2. 添加`SYS_yield`系统调用.
> 3. 设置系统调用的返回值.
>
> 重新运行dummy程序, 如果你的实现正确, 你会看到dummy程序又触发了一个号码为`0`的系统调用. 查看`nanos-lite/src/syscall.h`, 你会发现它是一个`SYS_exit`系统调用. 这说明之前的`SYS_yield`已经成功返回, 触发`SYS_exit`是因为dummy已经执行完毕, 准备退出了.
>
> #####  实现SYS_exit系统调用
>
> 你需要实现`SYS_exit`系统调用, 它会接收一个退出状态的参数, 用这个参数调用`_halt()`即可. 实现成功后, 再次运行dummy程序, 你会看到`HIT GOOD TRAP`的信息.

看不懂要求没关系，慢慢来，先看手册这一段话：

> 添加一个系统调用比你想象中要简单, 所有信息都已经准备好了. 我们只需要在分发的过程中添加相应的系统调用号, 并编写相应的系统调用处理函数`sys_xxx()`, 然后调用它即可. 回过头来看`dummy`程序, 它触发了一个`SYS_yield`系统调用. 我们约定, 这个系统调用直接调用CTE的`_yield()`即可, 然后返回`0`.
>
> 处理系统调用的最后一件事就是设置系统调用的返回值. 对于不同的ISA, 系统调用的返回值存放在不同的寄存器中, 宏`GPRx`用于实现这一抽象, 所以我们通过`GPRx`来进行设置系统调用返回值即可.
>
> 经过CTE, 执行流会从`do_syscall()`一路返回到用户程序的`_syscall_()`函数中. 代码最后会从相应寄存器中取出系统调用的返回值, 并返回给`_syscall_()`的调用者, 告知其系统调用执行的情况(如是否成功等).

通过以上内容，我们就不难写出如下内容（关于参数和返回值的相关信息可以看下面关于`GPR?`的相关说明）：

```c
#include "common.h"
#include "syscall.h"

void sys_yield();
void sys_exit(int code);

_Context* do_syscall(_Context *c) {
  uintptr_t a[4];
  a[0] = c->GPR1;
  a[1] = c->GPR2;
  a[2] = c->GPR3;
  a[3] = c->GPR4;


  switch (a[0]) {
    case SYS_yield:
      sys_yield();
      c->GPRx = 0;
      break;
    case SYS_exit:
      sys_exit(a[1]);
      break;
    default: panic("Unhandled syscall ID = %d", a[0]);
  }

  return NULL;
}

void sys_yield() {
  _yield();
}

void sys_exit(int code) {
  _halt(code);
}
```

正确的`GPR?`宏：观察`_syscall_`的代码，发现是从`a0`寄存器取得系统调用的返回结果，因此修改`GPRx`的宏定义，将其改成寄存器`a0`的下标，就可以在操作系统中通过`c->GPRx`根据实际情况设置返回值了。想必你还记得下图，易知`a0`就是`gpr[10]`。然后GPR1是已经使用了，这个和前文也有联系；GPR2-4则是设置成通用寄存器后面来使用；GPRx作为返回值来使用。

```c
#define GPR1 gpr[17]
#define GPR2 gpr[10]
#define GPR3 gpr[11]
#define GPR4 gpr[12]
#define GPRx gpr[10]
```

#### 3.2.3 标准输出

> ##### 在Nanos-lite上运行Hello world
>
> Navy-apps中提供了一个`hello`测试程序(`navy-apps/tests/hello`), 它首先通过`write()`来输出一句话, 然后通过`printf()`来不断输出.你需要实现`write()`系统调用, 然后把Nanos-lite上运行的用户程序切换成`hello`程序并运行

RTFM，知道我们需要完成`write()`系统调用。那么就是基本和前文一样的步骤补充。如下（截取了该部分补充的内容，其余省去）：

```c
#include "common.h"
#include "syscall.h"

int sys_write(int fd, void *buf, size_t count);

_Context* do_syscall(_Context *c) {
  switch (a[0]) {
    case SYS_write:
      c->GPRx = sys_write(a[1], (void*)(a[2]), a[3]);
      break;
  }
  return NULL;
}

int sys_write(int fd, void *buf, size_t count) {
  if(fd == 1 || fd == 2) {
    for(int i = 0; i < count; i++) {
      _putc(((char*)buf)[i]);
    }
    return count;
  }
  return 0;
}
```

然后是修改`nano.c`函数调用：

```c
int _write(int fd, void *buf, size_t count) {
  // _exit(SYS_write);
  // return 0;
  return _syscall_(SYS_write, fd, (intptr_t)buf, count);
}
```

#### 3.2.4 堆区管理

> ##### 实现堆区管理
>
> 根据上述内容在Nanos-lite中实现`SYS_brk`系统调用, 然后在用户层实现`_sbrk()`. 

实现`SYS_brk`系统调用:

```c
#include "common.h"
#include "syscall.h"
int sys_brk(intptr_t addr);
static int program_break;
_Context* do_syscall(_Context *c) {
  switch (a[0]) {
    case SYS_brk:
      c->GPRx = sys_brk(a[1]);
      break;
    default: panic("Unhandled syscall ID = %d", a[0]);
  }
  return NULL;
}
int sys_brk(intptr_t addr) {
  program_break = addr;
  return 0;
}
```

在用户层实现`_sbrk()`（即在`nano.c`中）:

```
extern uint32_t _end;
void *_sbrk(intptr_t increment) {
  static int program_break = &_end;
  int ret = program_break;
  if(!_syscall_(SYS_brk, program_break + increment, 0, 0)){
    program_break += increment;
    return (void *)ret;
  }
  return (void *)-1;
}
```

### PA3.3 文件系统

#### 3.3.1 简易文件系统
需要为每一个已经打开的文件引入偏移量属性`open_offset`, 来记录目前文件操作的位置. 每次对文件读写了多少个字节, 偏移量就前进多少（这个结构体的定义也在`fs.c`中）：
```c
typedef struct {
  char *name;
  size_t size;
  size_t disk_offset; // 文件在ramdisk中的偏移
  // PA3.3 added
  size_t open_offset; // 文件被打开之后的读写指针
  ReadFn read;
  WriteFn write;
} Finfo;
```

然后仔细RTFM，我们知道我们需要在`fs.c`中实现以下函数（最好是把它们的声明放在头文件中，这样会更清晰一些）：

```c
int fs_open(const char *pathname, int flags, int mode);
size_t fs_read(int fd, void *buf, size_t len);
size_t fs_write(int fd, const void *buf, size_t len);
size_t fs_lseek(int fd, size_t offset, int whence);
int fs_close(int fd);
```

实现它们的时候要注意以下内容：

> - 由于简易文件系统中每一个文件都是固定的, 不会产生新文件, 因此"`fs_open()`没有找到`pathname`所指示的文件"属于异常情况, 你需要使用assertion终止程序运行.
> - 为了简化实现, 我们允许所有用户程序都可以对所有已存在的文件进行读写, 这样以后, 我们在实现`fs_open()`的时候就可以忽略`flags`和`mode`了.
> - 使用`ramdisk_read()`和`ramdisk_write()`来进行文件的真正读写.
> - 由于文件的大小是固定的, 在实现`fs_read()`, `fs_write()`和`fs_lseek()`的时候, 注意偏移量不要越过文件的边界.
> - 除了写入`stdout`和`stderr`之外(用`_putc()`输出到串口), 其余对于`stdin`, `stdout`和`stderr`这三个特殊文件的操作可以直接忽略.
> - 由于我们的简易文件系统没有维护文件打开的状态, `fs_close()`可以直接返回`0`, 表示总是关闭成功.

接下来就是具体的代码了：

```c
// PA3.3 added
extern size_t ramdisk_read(void*, size_t, size_t);          // for fs_read
extern size_t ramdisk_write(const void*, size_t, size_t);   // for fs_write

// PA3.3 updated
int fs_open(const char *pathname, int flags, int mode){
  // printf("open %s\n", pathname);
  for(int i = 0; i < NR_FILES;i++){
    if(strcmp(pathname, file_table[i].name) == 0){
        // printf("open %s OK\n", pathname);
        return i;
    }
  }
  assert(0 && "Can't find file");
}

size_t fs_read(int fd, void *buf, size_t len) {  
//   printf("open_offset = %d, disk_offset = %d\n", file_table[fd].open_offset, file_table[fd].disk_offset);
  assert(fd >= 3 && fd < NR_FILES);
  if (file_table[fd].open_offset + len >= file_table[fd].size) {
    if (file_table[fd].size > file_table[fd].open_offset)
      len = file_table[fd].size - file_table[fd].open_offset;
    else
      len = 0;
  }
//   printf("read %d bytes\n", len);
  if (!file_table[fd].read) {
    ramdisk_read(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  }
  else {
    len = file_table[fd].read(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  }
  file_table[fd].open_offset += len;
//   printf("open_offset = %d, disk_offset = %d\n", file_table[fd].open_offset, file_table[fd].disk_offset);
  return len;
}

size_t fs_write(int fd, const void *buf, size_t len) {
  // printf("open_offset = %d, disk_offset = %d\n", file_table[fd].open_offset, file_table[fd].disk_offset);
  if(fd == 1 || fd == 2) {
    for(int i = 0; i < len; i++) {
      _putc(((char*)buf)[i]);
    }
    return len;
  }
  if (fd >= 3 && (file_table[fd].open_offset + len > file_table[fd].size)) {
    if (file_table[fd].size > file_table[fd].open_offset)
      len = file_table[fd].size - file_table[fd].open_offset;
    else
      len = 0;
  }
  // printf("write %d bytes\n", len);
  if (!file_table[fd].write) {
    ramdisk_write(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  }
  else {
    file_table[fd].write(buf, file_table[fd].disk_offset + file_table[fd].open_offset, len);
  }
  file_table[fd].open_offset += len;
  return len;
}

size_t fs_lseek(int fd, size_t offset, int whence) {
  // printf("lseek %d %d %d\n", fd, offset, whence);
  assert(fd >= 3 && fd < NR_FILES);
  switch(whence) {
    case SEEK_SET: file_table[fd].open_offset = offset; break;
    case SEEK_CUR: file_table[fd].open_offset += offset; break;
    case SEEK_END: file_table[fd].open_offset = file_table[fd].size + offset; break;
    default: assert(0 && "Invalid whence");
  }
  return file_table[fd].open_offset;
}

int fs_close(int fd){
    file_table[fd].open_offset = 0;
    return 0;
}
```

除了完成`fs.c`之外，我们还需要另外去实现`SYS_open`,`SYS_close`,`SYS_read`,`SYS_lseek`系统调用，并将`SYS_write`系统调用更新为调用`fs_write()`函数。

`__am_irq_handle()`：

```c
_Context* __am_irq_handle(_Context *c) {
  if (user_handler) {
    switch (c->cause) {
        // PA3 added for system instructions
        case 0: // exit
        case 1: // yield
        case 2: // open
        case 3: // read
        case 4: // write
        case 7: // close
        case 8: // lseek
        case 9: // sbrk
            ev.event = _EVENT_SYSCALL;
            break;
    }
  }
}
```

`nanos-lite/src/syscall.c`相关函数：

```c
#include "common.h"
#include "fs.h"
#include "syscall.h"

int sys_write(int fd, void *buf, size_t count);
// PA3.3 updated
int sys_open(const char *pathname, int flags, int mode);
int sys_read(int fd, void *buf, size_t count);
int sys_lseek(int fd, size_t offset, int whence);
int sys_close(int fd);

static int program_break;

_Context* do_syscall(_Context *c) {
  uintptr_t a[4];
    
  switch (a[0]) {
    case SYS_write:
      c->GPRx = sys_write(a[1], (void*)(a[2]), a[3]);
      break;
    case SYS_open:
      c->GPRx = sys_open((const char*)a[1], a[2], a[3]);
      break;
    case SYS_read:
      c->GPRx = sys_read(a[1], (void*)(a[2]), a[3]);
      break;
    case SYS_lseek:
      c->GPRx = sys_lseek(a[1], a[2], a[3]);
      break;
    case SYS_close:
      c->GPRx = sys_close(a[1]);
      break;
    default: panic("Unhandled syscall ID = %d", a[0]);
  }
  return NULL;
}

int sys_write(int fd, void *buf, size_t count) {
  // if(fd == 1 || fd == 2) {
  //   for(int i = 0; i < count; i++) {
  //     _putc(((char*)buf)[i]);
  //   }
  //   return count;
  // }
  // return 0;
  return fs_write(fd, buf, count);
}

int sys_open(const char *pathname, int flags, int mode) {
  return fs_open(pathname, flags, mode);
}

int sys_read(int fd, void *buf, size_t count) {
  return fs_read(fd, buf, count);
}

int sys_lseek(int fd, size_t offset, int whence) {
  return fs_lseek(fd, offset, whence);
}

int sys_close(int fd) {
  return fs_close(fd);
}
```

更新`loader()`函数：

```c
#include "fs.h"
static uintptr_t loader(PCB *pcb, const char *filename) {
  // PA3.3 updated: use fs_read instead of ramdisk_read
  Elf_Ehdr head;
  int fd = fs_open(filename, 0, 0);

  fs_lseek(fd, 0, SEEK_SET);
  fs_read(fd, &head, sizeof(head));

  for (int i = 0; i < head.e_phnum; i++) {
    Elf_Phdr temp;
    fs_lseek(fd, head.e_phoff + i * head.e_phentsize, SEEK_SET);
    fs_read(fd, &temp, sizeof(temp));
    if (temp.p_type == PT_LOAD) {
      fs_lseek(fd, temp.p_offset, SEEK_SET);
      fs_read(fd, (void *)temp.p_vaddr, temp.p_filesz);
      memset((void *)(temp.p_vaddr + temp.p_filesz), 0, temp.p_memsz - temp.p_filesz);
    }
  }
  return head.e_entry;
}
```

最后不要忘记更新`nanos.c`：

```c
int _open(const char *path, int flags, mode_t mode) {
  // _exit(SYS_open);
  // return 0;
  return _syscall_(SYS_open, (intptr_t)path, flags, mode);
}

int _write(int fd, void *buf, size_t count) {
  // _exit(SYS_write);
  // return 0;
  return _syscall_(SYS_write, fd, (intptr_t)buf, count);
}

int _read(int fd, void *buf, size_t count) {
  // _exit(SYS_read);
  // return 0;
  return _syscall_(SYS_read, fd, (intptr_t)buf, count);
}

int _close(int fd) {
  // _exit(SYS_close);
  // return 0;
  return _syscall_(SYS_close, fd, 0, 0);
}

off_t _lseek(int fd, off_t offset, int whence) {
  // _exit(SYS_lseek);
  // return 0;
  return _syscall_(SYS_lseek, fd, offset, whence);
}
```
#### 3.3.2 一切皆文件：操作系统上的IOE
##### (1) 串口抽象为文件

RTFM，很容易知道这一步我们要做的是修改 `file_table`，并且完善`serial_write()`函数。具体来说是这样：

`fs.c`文件：

```c
// PA3.3 added
extern size_t serial_write(const void*, size_t, size_t);    // for stdout and stderr

/* This is the information about all files in disk. */
static Finfo file_table[] __attribute__((used)) = {
  {"stdin", 0, 0, 0, invalid_read, invalid_write},
  {"stdout", 0, 0, 0, invalid_read, serial_write},
  {"stderr", 0, 0, 0, invalid_read, serial_write},
#include "files.h"
};
```

`device.c`文件：

```c
size_t serial_write(const void *buf, size_t offset, size_t len) {
  for (size_t i = 0; i < len; i++) {
    _putc(((char *)buf)[i]);
  }
  return 0;
}
```

然后`fs_write()`函数就不需要单独判断fd为1或者2的情况了。

##### (2) 设备输入抽象成文件

> ##### 把设备输入抽象成文件
>
> 你需要在Nanos-lite中
>
> - 实现`events_read()`(在`nanos-lite/src/device.c`中定义), 把事件写入到`buf`中, 最长写入`len`字节, 然后返回写入的实际长度. 其中按键名已经在字符串数组`names`中定义好了. 你需要借助IOE的API来获得设备的输入.
> - 在VFS中添加对`/dev/events`的支持.
>
> 让Nanos-lite加载`/bin/events`, 如果实现正确, 你会看到程序输出时间事件的信息, 敲击按键时会输出按键事件的信息.

补充的内容如下，基本均可以参考手册中的内容完成编写：

```c
// fs.c
extern size_t events_read(void*, size_t, size_t);           // for events
static Finfo file_table[] __attribute__((used)) = {
  {"/dev/events", 0, 0, 0, events_read, invalid_write},
#include "files.h"
};
```

```c
// device.c
size_t events_read(void *buf, size_t offset, size_t len) {
  int kc = read_key();
  char tmp[3] = "ku";
  if ((kc & 0xfff) == _KEY_NONE) {
    int time = uptime();
    len = sprintf(buf, "t %d\n", time);
  }
  else {
    if (kc & 0x8000)
      tmp[1] = 'd';
    len = sprintf(buf, "%s %s\n", tmp, keyname[kc & 0xfff]);
  }
  return len;
}
```