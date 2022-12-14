# Game Boy 模拟器:设计 CPU

> 原文：<https://www.inspiredpython.com/course/game-boy-emulator/game-boy-emulator-designing-the-cpu>

是时候充实我们的框架类了，这样模拟器的核心就可以成形了。让我们从收银机开始。



### 代表寄存器和标志

正如我前面提到的，你需要把寄存器看作仅仅是变量。然而，唯一的两个障碍是 16 位字的高低概念，标志是存储在`AF`低位字中的位域。

因此，如果你有一个 16 位的数字`0xABCD`并且你想要高位和低位字分开，你将需要旋转一些位，正如表达式所说。



#### 钻头操作快速入门

我将快速介绍我们需要的内容，但是一旦我们深入查看一些说明，就会有更深入的探讨。话虽如此，我还是建议你尝试一下，因为直觉理解是必不可少的。

因此，要获取或设置任意位，我们需要了解一些关于二进制和位运算的基本知识。考虑数字`0xAB`的二进制表示:

```
+-----+-----+-----+-----+-----+-----+-----+-----+
| 128 | 64  | 32  | 16  |  8  |  4  |  2  |  1  |
+-----+-----+-----+-----+-----+-----+-----+-----+ = 0xAB
|  1  |  0  |  1  |  0  |  1  |  0  |  1  |  1  |
+-----+-----+-----+-----+-----+-----+-----+-----+
```

假设我想要`0xAB`的`0xA`。我可以通过右移四位来实现这一点，因为我希望 8 位(即 4 位)中的*高电平部分*:

因为大多数计算机从右向左计数，并且因为移位*将*位 *N* 位向左或向右移动，所以右移位有效地擦除了它所替换的位:

```
+-----+-----+-----+-----+-----+-----+-----+-----+
| 128 | 64  | 32  | 16  |  8  |  4  |  2  |  1  |
+-----+-----+-----+-----+-----+-----+-----+-----+ = 0xA
|  0  |  0  |  0  |  0  |  1  |  0  |  1  |  0  |
+-----+-----+-----+-----+-----+-----+-----+-----+
Shift right 4 ->        \-----------------------/
```

当我们向右移动时，这些值被填充为 0，因此是多余的，就像用十进制写`00042`和`42`一样。

与之相反的是左移:

向左移动，然后再向你移动的方向移动。不过，请注意新值`0xA0`。就像十进制数乘以 10 加一个零一样，左移也是如此。

那么，左移相当于乘法，右移相当于 2 的幂的除法*。所以，一个除了位移位和加减运算之外什么都不会的有事业心的程序员，即使 CPU 不支持这样的运算，也能模拟乘除运算。*

 试着用 2 的乘方数进行除法和乘法来复制位移。 

这就是高潮词。如果我想要更低的单词呢？我不能移动，因为那会把号码抹掉。相反，我需要一种提取这些信息的方法——一种能让我们更好地前进的方法。

获取您想要的位的一种方法是询问我们想要的位是否被设置，然后返回它们。多亏了按位`&`运算符和一个叫做*位屏蔽*的概念，这很容易做到。



##### 比特屏蔽

位运算的工作方式很像在 if 语句中使用的逻辑运算。逐位运算对每一位进行操作，并对每一对位进行逻辑`and`运算。

```
+-----+-----+-----+-----+
|  8  |  4  |  2  |  1  |  Bit
+-----+-----+-----+-----+
|  0  |  1  |  1  |  0  |  A
+-----+-----+-----+-----+  &
|  1  |  0  |  1  |  1  |  B
+-----+-----+-----+-----+  =
|  0  |  0  |  1  |  0  |  C
+-----+-----+-----+-----+
```

想想当你把这两个数按位`&`在一起时会发生什么。因为逻辑`and`按顺序应用于每个位，所以结果是这些按位运算的结果。

然后一个聪明的程序员可以使用一个*掩码*——一个数字——只返回他们需要的比特。从例子中可以看出，`A`和`B`中只有位 2 为`1`，所以结果为`2`，

所以一个面具是有用的，如果你想得到(或设置！)位。

面具通常放在右手边

这不是一个严格的规则，但是如果你硬编码常量(你将在后面看到),习惯上使用右边的掩码和左边的值进行掩码。

如果你因为风格的原因或者为了复制一个算法而反过来做也没问题，但是要保持一致。





##### 检查是否用按位 AND 设置了一个或多个位

因此，如果我想要低位，我可以使用按位`&`操作符挑选出我想要的位。

给定数字`0xAB`，我可以用设置了 1、2、4 和 8 位的掩码得到较低的四位:

当我用 T1 表示 T0 时:

```
>>>  hex(0xAB  &  0b1111) '0xb'
```

这就是我们想要的答案。





##### 用按位“或”设置位

问问你自己什么是逻辑`or`——如果左边或右边的一个或两个是`True`,它返回`True`。这也适用于按位`|`。

再次应用掩码的概念，您可以使用相同的概念来*设置*位:

```
+-----+-----+-----+-----+
|  8  |  4  |  2  |  1  |  Bit
+-----+-----+-----+-----+
|  0  |  0  |  1  |  0  |  A
+-----+-----+-----+-----+  |
|  1  |  0  |  1  |  1  |  B
+-----+-----+-----+-----+  =
|  1  |  0  |  1  |  1  |  C
+-----+-----+-----+-----+
```

给定一个掩码，我们现在可以用`|`任意设置位。

```
>>>  bin(0b0010  |  0b1011) '0b1011'
```

其中`0b1011`实际上是两个输入的按位`|`。





##### 用按位异或切换位

XOR 或 eXclusive OR 在 Python 中没有逻辑对应物。它的用途主要是，但不总是，归入逐位运算或作为算法中的专业工具，因为它拥有一个有趣的特性  。

由于其独特的转换位的能力，这一特性使得创造密码无法破解的一次性密码本成为可能。

也就是说，XOR ( `^`)的结果只有*非零*(记住，这里没有布尔运算！)如果**左侧或右侧的**之一非零。即，与按位 AND 不同，两边都不能非零或为零；与按位“或”不同，只有一边可以为零，另一边必须非零。

再次应用遮罩的概念，让我们切换一些位

```
+-----+-----+-----+-----+
|  8  |  4  |  2  |  1  |  Bit
+-----+-----+-----+-----+
|  0  |  0  |  1  |  0  |  A
+-----+-----+-----+-----+  ^
|  1  |  0  |  1  |  1  |  B
+-----+-----+-----+-----+  =
|  1  |  0  |  0  |  1  |  C
+-----+-----+-----+-----+
```

```
>>>  bin(0b0010  ^  0b1011) '0b1001'
```

请注意我们如何切换这些位，除了值和掩码都是`0`的位。

按位运算符返回运算结果

我们得到的不是`True`或`False`的答案，而是运算的结果。很有用，那个！

按位运算符和掩码是获取、设置、重置或切换位的有用工具

掩码是有用的工具，任何东西都可以是掩码，包括您想要测试的另一个值。

按位 AND 检查是否设置了某些内容；按位“或”用于设置位；按位异或用于切换位

您可以将移位与按位运算符结合使用

这允许您将单个位移动到您想要检查的位置。试试`1 << 7`，看看它的二进制，十进制，十六进制形式。









### 寄存器和标志

好了，有了位的基础知识，现在是表示寄存器的时候了。你可以使用一个简单的字典，或者一个类的显式变量或属性——这取决于你。

我将使用修改过的字典式符号，因为它便于阅读和推理。但是无论如何，选择一个你认为对你最有意义的方法。

让我们从我们需要的寄存器和标志的一些映射开始，以及它们如何相互关联。

```
REGISTERS_LOW =  {"F":  "AF",  "C":  "BC",  "E":  "DE",  "L":  "HL"} REGISTERS_HIGH =  {"A":  "AF",  "B":  "BC",  "D":  "DE",  "H":  "HL"} REGISTERS =  {"AF",  "BC",  "DE",  "HL",  "PC",  "SP"} FLAGS =  {"c":  4,  "h":  5,  "n":  6,  "z":  7}
```

这些常量将低位寄存器(如`C`)映射到`BC`。这样，我们的代码可以通过一个简单的 if 语句链来检查寄存器是低位、高位、16 位寄存器还是标志。

```
from collections.abc import MutableMapping   @dataclass class  Registers(MutableMapping):   AF:  int BC:  int DE:  int HL:  int PC:  int SP:  int   def  values(self): return  [self.AF, self.BC, self.DE, self.HL, self.PC, self.SP]   def  __iter__(self): return  iter(self.values())   def  __len__(self): return  len(self.values())
```

在这里，我将为一个定制的字典风格的类打下基础。我继承了一个抽象基类`MutableMapping`，它确保我覆盖了所有正确的方法来提供字典式的访问(`registers["PC"] = 42`)。)

我还使用了一个数据类，这样每个主寄存器都是构造函数的一部分。

接下来，是时候定义访问器了:

```
def  __getitem__(self, key):   if key in REGISTERS_HIGH: register = REGISTERS_HIGH[key] return  getattr(self, register)  >>  8 elif key in REGISTERS_LOW: register = REGISTERS_LOW[key] return  getattr(self, register)  &  0xFF elif key in FLAGS: flag_bit = FLAGS[key] return self.AF >> flag_bit &  1 else: if key in REGISTERS: return  getattr(self, key) else: raise KeyError(f"No such register {key}")
```

该守则是故意说教；但是意图很简单。我依次检查每个寄存器集合，当我找到它时，我根据需要应用正确的位运算:

1.  高位寄存器向右移位 8 位，得到我们想要的值

2.  低位寄存器与掩码`0xFF`进行逐位“与”运算，因为它与低位 8 位相匹配。

3.  相反，标志是根据它们在`AF`中的位置移动的，所以我们关心的位被放在最右边的位置，在那里我可以用`1`进行位与运算，以检查它是否被置位。

4.  对 16 位寄存器的请求只是简单地返回该值，无需修改。

5.  其他一切都是一个`KeyError`

设置一个寄存器或多或少与获取是一样的，但是按位运算符现在是`|`。

```
def  __setitem__(self, key, value):   if key in REGISTERS_HIGH: register = REGISTERS_HIGH[key] current_value = self[register] setattr(self, register,  (current_value &  0x00FF  |  (value <<  8))  &  0xFFFF) elif key in REGISTERS_LOW: register = REGISTERS_LOW[key] current_value = self[register] setattr(self, register,  (current_value &  0xFF00  | value)  &  0xFFFF) elif key in FLAGS: assert value in  (0,  1),  f"{value} must be 0 or 1" flag_bit = FLAGS[key] if value ==  0: self.AF = self.AF &  ~(1  << flag_bit) else: self.AF = self.AF |  (1  << flag_bit) else: if key in REGISTERS: setattr(self, key, value &  0xFFFF) else: raise KeyError(f"No such register {key}")   def  __delitem__(self, key):   raise NotImplementedError("Register deletion is not supported")
```

1.  设置高电平寄存器是这样一种情况:取值并将其左移 8 位到高电平位置，然后用掩码清除以前的值。我使用按位`|`来确保我们保留寄存器中的其他内容(`current_value`

2.  对于低位寄存器，我简单地应用按位 or，因为不需要移位。

3.  对于 16 位寄存器，设置值很简单。

4.  对于标志，我们将需要复位的位(`value == 0`)移动到正确的位置，然后使用*补码* ( `~`)翻转所有位，然后用`AF`屏蔽。如果`value == 1`那么我使用按位或。

5.  其他一切都是一个`KeyError`。

你可能已经注意到了每个作业末尾的`& 0xFFFF`。这是防止值溢出主寄存器 16 位大小限制的安全措施。我们正在编写的仿真器每个寄存器有 16 位，但是 Python 并不关心这个。屏蔽`0xFFFF`确保我们永远不会溢出。

 为什么那个口罩会起作用，它是如何防止溢出的？试着给`0xFFFF`加 1，看看屏蔽前后的二进制表示。 

现在做一些测试。我再次使用 [假设来生成策略](https://www.inspiredpython.com/course/testing-with-hypothesis/testing-your-python-code-with-hypothesis) :

```
import hypothesis.strategies as st from hypothesis import given   @pytest.fixture(scope="session") def  make_registers():   def  make(): return Registers(AF=0, BC=0, DE=0, HL=0, PC=0, SP=0) return make   @given(   value=st.integers(min_value=0, max_value=0xFF), field=st.sampled_from(sorted(REGISTERS_HIGH.items())), ) def  test_registers_high(make_registers, field, value):   registers = make_registers() high_register, full_register = field registers[high_register]  = value assert registers[full_register]  == value <<  8
```

测试很简单。我创建了一个`Registers`类，并通过手动移动它并检查 16 位大小的寄存器来测试我们设置的值——它是由假设随机抽取的——是我们所期望的。(这就是为什么保持寄存器分离并在测试中易于重用是值得的。)

既然您已经熟悉了测试的结构以及如何验证您的结果，那么测试低尺寸和全尺寸寄存器就足够容易了。试着自己写。

测试标志也同样简单:

```
@given(   starting_value=st.integers(min_value=0, max_value=0xFF), field=st.sampled_from(sorted(FLAGS.items())), ) def  test_flags(make_registers, starting_value, field):   flag, bit = field registers = make_registers() outcome =  [] for value in  (0,  1):  registers.AF = starting_value registers[flag]  = value outcome.append(registers["F"]) assert registers["F"]  >> bit &  1  == value assert outcome.pop()  != outcome.pop()
```

最终的结果是一个简单的 dict 风格的类，允许您使用它们的名称查询低位、高位、标志和满寄存器。





### 中央处理器

CPU 是模拟器的核心部分，所以它必须是可扩展的，因为我们会随着时间的推移添加它；为了便于测试，部件必须易于换入或换出；它应该封装 CPU 的状态，这将包括一些我们还没有谈到的东西。

除了函数之外，您可以什么都不用做，但是为了保持到目前为止的风格，我将使用一个 dataclass 来表示 CPU。

```
class  InstructionError(Exception):   pass     @dataclass class  CPU:   registers: Registers decoder: Decoder   def  execute(self, instruction: Instruction):    match instruction: case Instruction(mnemonic="NOP"): pass case _: raise InstructionError(f"Cannot execute {instruction}")   def  run(self): while  True: address = self.registers["PC"] try: next_address, instruction = self.decoder.decode(address) except IndexError: break self.registers["PC"]  = next_address self.execute(instruction)
```

很简单。我再次使用了*依赖注入*模式。这意味着 CPU 不应该知道如何创建或实例化它所依赖的现存类。要使用它，你必须传递`Decoder`(你在上一章创建的)和`Registers`的`CPU`构造器实例。

我喜欢这种模式，因为它分离了关注点，所以每个类只知道它需要知道的东西，*，仅此而已！*因此，如果我们愿意，我们可以传入`FakeRegisters`或`FakeDecoder`来模拟我们想要测试的特定行为，甚至是替代实现。

但是如您所见，实现相当稀疏。然而，这足以开始执行指令。目前还没有内存条控制器，也没有显示屏或我们需要的任何其他花哨功能，但这很好。我们以后再去找他们。但是因为我们一丝不苟地一次实现了一个部分，所以现在需要将我们到目前为止构建的部分整合在一起:

指令提取和解码(大部分)已经完成

多亏了单一的`decode`方法，我们现在可以从字节流中获取指令；解码指令；将流推进到下一个位置；并返回解码后的指令。

CPU 现在处于利用这些信息的最佳位置。我们知道下一个逻辑指令(`next_address`)的地址，并且可以相应地更新`PC`寄存器。

指令执行是下一个重要步骤

我将使用 Python 3.10 的 [匹配和 case 关键字](https://www.inspiredpython.com/course/pattern-matching/mastering-structural-pattern-matching) ，但是如果你没有那个版本或者不想使用那个特性，也不用担心。您可以使用很多很多 if 语句，或者使用字典分派给执行的函数。

考虑如何存储从操作码文件加载的指令。您现在需要一种方法来调用能够处理该指令的函数。

我添加了一个单独的指令，`NOP`，它不做任何事情(它意味着*不操作*)

`run()`方法有一个无限的`while`循环:一旦我们开始，我们就不会停止，除非我们用完了所有的指令——因此有了`IndexError`检查——或者 CPU 被指令或人告知停止。

没有太多要测试的，所以让我们看看是否可以执行一系列任意的`NOP`指令:

```
@pytest.fixture(scope="session") def  make_cpu(make_registers, make_decoder):   def  make(data, pc=0): cpu = CPU(registers=make_registers(pc=pc), decoder=make_decoder(data=data)) return cpu   return make     @given(count=st.integers(min_value=0, max_value=100)) def  test_cpu_execute_nop_and_advance(make_cpu, count):   cpu = make_cpu(b"\x00"  * count) assert cpu.registers["PC"]  ==  0 cpu.run() assert cpu.registers["PC"]  == count
```

仅此而已。我们的骨架 CPU 完成了。它可以根据需要读写每个寄存器和标志；它可以执行指令。

