Java的原始类型里没有无符号类型，c语言int类型表示有符号整型，unsigned int无符号整型

int占4个字节，每个字节8位，占32位，取值范围 -231~231-1，有32个0-1的二进制位。

左起第一位是符号位， 0表示正数，1表示负数 其余后面31位是数值位。

**0 0000000000000000000000000000010**

数字0的表示： 按照上面提到的符号，有两种0的表示方法，即“+0”和“-0”。 实际上，在32位系统下int类型中，计算机已经强行规定数字0采用“+0”的表示方法，即00000000 00000000 00000000；而“-0”这个特殊的数字被定义为了-231。 因此我们看到32位系统下int类型的取值范围中，负数部分比正数部分多了一个数字，正数的最大取值是231-1，而负数的最小取值是-231。正数部分之所以要减去1，是因为被数字0占用了，而负数部分不需要用来表示0，因此原本的“-0”就用来表示-231这个数字。

那么-1按照上面描述就是： **10000000 00000000 00000001** 了？左起第一位是符号位1表示负数？右起第一位是数值1？ 实际不是这样的。这里就需要引入“补码”了。

计算机中的符号数有三种表示方法，即原码、反码和补码。三种表示方法均有符号位和数值位两部分，符号位都是用0表示“正”，用1表示“负”，而数值位，三种表示方法各不相同。

#### **原码**

最高位1表示负，0表示正，剩余位表示数值。

原码的1： 00000000000000000000000000000001

原码的-1：10000000000000000000000000000001

[原码](https://baike.baidu.com/item/原码)不能直接参加运算，可能会出错。例如数学上，1+(-1)=0 ，原码计算则是

00000000000000000000000000000001+10000000000000000000000000000001=-2 是错的。

原码的符号位不能直接参与运算。

#### **反码**

正数的反码与其原码相同；负数的反码是对其原码逐位取反，但符号位除外。

对于正数和“+0”而言，其原码本身就是反码，例如 8位二进制“+1”，其原码与反码都是00000001；

对于负数和“-0”而言，符号位与原码中一样，保持不变，其余位数逐位取反，1换成0,0换成1，例如 “-1”，其8位二进制原码是1000 0001，其反码是1111 1110；

那么我们是否已经可以正常进行运算了呢？

我们举个三个例子：

例一：1+2=3（以8位二进制表示）

| 十进制         | 原码      | 反码      |
| -------------- | --------- | --------- |
| 1              | 0000 0001 | 0000 0001 |
| 2              | 0000 0010 | 0000 0010 |
| 结果（反码）   |           | 0000 0011 |
| 结果（原码）   |           | 0000 0011 |
| 结果（十进制） |           | 3         |

计算结果正确。

例二：1+（-2）=-1

| 十进制         | 原码      | 反码      |
| -------------- | --------- | --------- |
| 1              | 0000 0001 | 0000 0001 |
| -2             | 1000 0010 | 1111 1101 |
| 结果（反码）   |           | 1111 1110 |
| 结果（原码）   |           | 1000 0001 |
| 结果（十进制） |           | -1        |

计算结果正确。

例三：1+（-1）=0

| 十进制         | 原码      | 反码      |
| -------------- | --------- | --------- |
| 1              | 0000 0001 | 0000 0001 |
| -1             | 1000 0001 | 1111 1110 |
| 结果（反码）   |           | 1111 1111 |
| 结果（原码）   |           | 1000 0000 |
| 结果（十进制） |           | -0        |

计算结果为-0，问题来了，由于-0的存在，使得二进制与十进制的互换不再是一一对应的关系。

总结：由于-0这个问题的存在，会使得计算机需要增加额外的物理硬件配合运算，所以在计算机发展的早期就已经抛弃了使用反码储存数据。

#### 补码

补码正是基于反码的“-0”问题诞生的，可以解决这个问题。

补码的计算方法是：正数和+0的补码是其本身，负数则先计算其反码，然后反码加上1，得到补码。

补码换算为原码的过程中，如果补码是正数或者+0的补码，则其原码就是补码本身；如果补码是负数或者-0的补码，则其原码的计算方法是，先将补码减掉1，得到反码，再将反码取反，得到原码。

例一：1+（-1）=0

| 十进制         | 原码      | 反码      | 补码      |
| -------------- | --------- | --------- | --------- |
| 1              | 0000 0001 | 0000 0001 | 0000 0001 |
| -1             | 1000 0001 | 1111 1110 | 1111 1111 |
| 结果（补码）   |           |           | 0000 0000 |
| 结果（反码）   |           |           | 0000 0000 |
| 结果（原码）   |           |           | 0000 0000 |
| 结果（十进制） |           |           | +0        |

计算结果正确，+0即是数字0的唯一表示。

例二：1+2=3

| 十进制         | 原码      | 反码      | 补码      |
| -------------- | --------- | --------- | --------- |
| 1              | 0000 0001 | 0000 0001 | 0000 0001 |
| 2              | 0000 0010 | 0000 0010 | 0000 0010 |
| 结果（补码）   |           |           | 0000 0011 |
| 结果（反码）   |           |           | 0000 0011 |
| 结果（原码）   |           |           | 0000 0011 |
| 结果（十进制） |           |           | 3         |

计算结果正确。

特别地，我们加入例四：（-1）+（-127）=-128

我们知道8位二进制的符号数的取值范围是(-27)～(２7-1),即-128～127。

| 十进制         | 原码      | 反码      | 补码      |
| -------------- | --------- | --------- | --------- |
| -1             | 1000 0001 | 1111 1110 | 1111 1111 |
| -127           | 1111 1111 | 1000 0000 | 1000 0001 |
| 结果（补码）   |           |           | 1000 0000 |
| 结果（反码）   |           |           |           |
| 结果（原码）   |           |           |           |
| 结果（十进制） |           |           | -128      |

由于补码1000 0000具有特殊性，计算机在编写底层算法时，将其规定为该取值范围中的最小数-128，其值与（-1）+(-127)的计算结果正好符合。

补充一点，8位二进制补码1000 0000没有对应的反码和原码，其他位数的二进制补码与此类似。

**int类型在内存中，以补码的形式存储。而且我们还知道了为何int类型的取值范围中负数的最小值的绝对值比正数的最大值大1的原因，即-232的补码是10000000 00000000 00000000，原本-0的位置被-232取代了**

**二进制数在内存中以补码的形式存储**

**正数：原码、补码和反码都相同**

**负数：补码符号位不变，其它位是对应正数的原码取反加一**