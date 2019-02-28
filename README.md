# NFChacker


[NFC基础](https://xuanxuanblingbling.github.io/nfc/2018/07/29/NFC/)
[Android HCE开发](https://xuanxuanblingbling.github.io/ctf/android/2018/07/29/AndriodHCE/)


对于NFC的攻击可以分为攻击卡与攻击读卡器：

- 攻击卡就是用android模拟读卡器，给卡发送payload
- 攻击读卡器就是用android模拟卡，给读卡器发送payload

目前只通过了android的HCE功能实现了模拟卡，即攻击读卡器，并且可以根据某公司泄露的内部技术手册可以了解常用的APDU指令


## 代码结构

### Activity

- MainActivity：主界面
- EditActivity：编辑查表界面
- RuleActivity：编辑规则界面
- LogActivity：查看日志

### Service

- PaymentServiceHost：HCE服务实现

### 工具类

- ToolsUnit：自己写的工具类，重要函数都在这里
- ListViewAdapter：查表界面与规则界面中的listview的适配器
- ItemBean：查表界面与规则界面中的listview的适配器需要的数据结构

### 配置文件

- host_aid_list.xml：配置HCE路由表，已经添加AID:a000000632010105

### 资源文件

assets目录下的初始HCE实例文件，最终会被释放到/sdcard/NFChacker/data/目录下

## 存储实现

所有数据的存储均为txt文本文件，在程序初始化时会在设备的sdcard目录下新建一个NFChacker文件夹，其中有两个子文件夹：log文件夹保存每次记录的日志信息，data文件夹保存一个HCE通信示例的配置文件

### 配置文件

/sdcard/MTYNFChacker/data/*.txt

内容是一个json数组的字符串，数组的第一个json对象是查表数据，第二个json对象是特殊规则，如测试用例

```JSON
[{
"00a4040008a000000632010105":"6a82",
"00b0840023":"10007511856076490102003000000000000000000000000020170911201909130000009000",
"00b0870009":"ffffffffffffffffff9000",
"00a40000021001":"6f0484024f439000",
"00b2019c00":"00015e000d1600011806231102308000180000000000009000",
"805c000204":"000003209000",
"0084000004":"d97767e69000",
"04d6940012180708182601ba90011f0201632668b774ca":"9000",
"00b094000e":"180708182601ba90011f020163269000",
"00a40000023f00":"6f10840e315041592e5359532e44444630319000",
"04d6850016000000004500000000201807012214300160df891739":"9000",
"00b0850012":"0000000045000000002018070122143001609000"
},{
"04d694-1-5-14":"-0-0-9000",
"00b094-0-0-0":"-1-180708182601ba90011f02016326-9000",
"04d685-2-5-18":"-0-0-9000",
"00b085-0-0-0":"-2-000000004500000000201807012214300160-9000"
}]

```

可以通过软件界面中的第一行的保存来修改或者添加配置文件，也可以通过手工修改文件的方式来更改

### 日志文件

/sdcard/NFChacker/log/*.txt

每次当我们保存日志时，则会记录到这个文件夹中，就是普通的txt格式，没有做什么特殊的处理

## 特殊规则

我们正常的查表数据如下：

```JSON
{
"00a4040008a000000632010105":"6a82",
"00b0840023":"10007511856076490102003000000000000000000000000020170911201909130000009000",
"00b0870009":"ffffffffffffffffff9000",
"00a40000021001":"6f0484024f439000",
"00b2019c00":"00015e000d1600011806231102308000180000000000009000",
"805c000204":"000003209000",
"0084000004":"d97767e69000",
"04d6940012180708182601ba90011f0201632668b774ca":"9000",
"00b094000e":"180708182601ba90011f020163269000",
"00a40000023f00":"6f10840e315041592e5359532e44444630319000",
"04d6850016000000004500000000201807012214300160df891739":"9000",
"00b0850012":"0000000045000000002018070122143001609000"
}
```

这种数据的保存方式只能针对固定的读卡器数据来进行返回，功能不灵活，实际应用中有如下情景：

- 匹配读卡器发的前缀来返回固定值
- 保留读卡器发的数据并当作之后的返回数据
- 返回数据中的随机数替换，例如随机金额，随机卡号

所以为了实现这些特殊的情景，我编写了一种规则，类似上面的JSON数据。当我们填写规则时，读卡器与卡的数据都被三个短线（-）分成了四个部分，例如

```JSON
{
    "80dc-0-0-0":"-0-0-9000",
    "8054-0-0-0":"8654a9484a88eb1a-0-0-9000"
}
```

于是我便可以通过每一条读卡器以及卡返回的特殊配置中，设置8个字段，则实现成上述的情景。

```JSON
{
	"匹配前缀-变量序号-变量起始位置-变量长度":"返回前缀-变量序号-变量初始值-返回后缀"
}
```



### 匹配前缀

当读卡器发送8050XXXXXXXXXXXXXXXX或者80dcXXXXXXXXXXXXXX这样的指令时，我们不关心XXXX究竟是什么，只需要按照检测的前几位来进行比配就可以了：

```
用了8个字段中的3个字段

匹配：匹配前缀
返回：返回前缀+返回后缀
```

- 读卡器字段只需要配置第一个字段，即匹配前缀，读卡器后三个字填0就也可以。
- 卡返回字段需要配置返回前缀与后缀，如果只是返回9000则不需要配置前缀

JSON可以写成如下格式：

```json
{
    "80dc-0-0-0":"-0-0-9000",
    "8050-0-0-0":"000162af0036000320010084176ad7-0-0-9000",
    "8054-0-0-0":"8654a9484a88eb1a-0-0-9000"
}
```

以上三条的意思是：

- 匹配到前缀80dc，则返回9000
- 匹配到前缀8050，则返回000162af0036000320010084176ad79000
- 匹配到前缀8054，则返回8654a9484a88eb1a9000

注：前缀不一定非得是四个，是偶数个即可

### 记录变量

在某个过程中有可能这样一个情景：

- POS机先通过04d6命令去写一个东西到94这个文件中
- 然后POS又通过00b0把刚才写的东西读出来

![image](https://xuanxuanblingbling.github.io/assets/pic/nfcsave.png)

所以这里就需要记录04d6中包含的数据，然后当收到00b0的时候再发出去：

```
用了8个字段中的8个字段

匹配：匹配前缀
记录：变量序号，变量起始位置，变量长度
返回：返回前缀+变量/变量初始值+返回后缀

```

```JSON
{
"04d694-1-5-14":"-0-0-9000",
"00b094-0-0-0":"-1-180708182601ba90011f02016326-9000",
"04d685-2-5-18":"-0-0-9000",
"00b085-0-0-0":"-2-000000004500000000201807012214300160-9000"
}
```

- 匹配到前缀04d694，记录第1个变量，内容这条指令的从第5个字节开始长度为14字节的数据部分，返回9000
- 匹配到前缀00b094，返回记录的第1个变量后接上9000，若此时没有记录到第1个变量则返回180708182601ba90011f020163269000
- 匹配到前缀04d685，记录第2个变量，内容这条指令的从第5个字节开始长度为18字节的数据部分，返回9000
- 匹配到前缀00b085，返回记录的第2变量后接上9000，若此时没有记录到第2个变量则返回0000000045000000002018070122143001609000

### 随机卡号

通过随机卡号可以绕过黑名单机制，这里原来的卡返回字段被我重新规定：

```json
{
	"匹配前缀-变量序号-变量起始位置-变量长度":"返回前缀-*-随机数字位数-返回后缀"
}
```

如果在卡返回的第二个字段上填写*（星号），则卡返回的第三个字段则会被当作随机数字的位数来进行随机数的生成（十进制），例如我们知道在某读卡器号如下指令以及返回：

![image](https://xuanxuanblingbling.github.io/assets/pic/nfcid.png)

其中`1000751160360389`就是卡号，例如我们想替换卡号后四位，于是我们按照如下书写规则：

```json
{
"00b084-0-0-0":"100075116036-*-4-0102003000000000000000000000000020170911201909130000009000"
}
```