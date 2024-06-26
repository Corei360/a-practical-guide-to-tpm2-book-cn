# PCR
平台配置寄存器是TPM必需的特性之一。它们最初的应用是用于提供一种密码学的方式记录（测量）软件的状态：包括平台上运行的软件和软件使用的配置数据。PCR的更新方法叫做扩展，扩展是一种单向的哈希操作，从而保证测量值不被篡改。这些用于测量的PCR可以被读取来报告这些软件的状态。这些PCR的值可以被签名然后用于更加安全的报告，这被称作认证（或者说引用）。PCR还可以被用于扩展的授权policy从而限制其他TPM对象的使用。

TPM从来不对测量的结果做任何判定。单单根据TPM内部的信息来看，并不能确定测量结果的好坏，或者结果是否安全可信。在测量软件时，TPM仅仅用PCR来记录测量值。至于是否安全，这要到应用程序真正使用PCR用于policy授权的时候，或者是远程请求者请求一个签名认证（quote，引用）然后判定可信性。

TPM2.0新增加的关于PCR的特性是，TPM不再将PCR的哈希算法固定为SHA-1。哈希算法现在可以修改了。有一些TPM的实现包含bank的概念，每一个bank实现一种不同的算法。

一个TPM会实现一定数量的PCR：比如说，PC上使用的TPM实现了24个。这些PCR按照惯例被分配给各种各样的软件层使用，软件类型从早期启动代码到操作系统和应用。这些PCR的分配还可以分为以下两类：需要运行的软件（通常是偶数序号的PCR），和用于定制启动过程的配置文件（通常是奇数序号的PCR）。

## PCR的值
PCR最初的应用场景是用于表示平台软件的状态，重要软件运行到当前阶段时的历史信息（包括配置信息）。TPM上电时会初始化所有的PCR，初始值由TPM平台相关的规范定义，通常是全0或者全1。命令调用者不能直接向PCR写值。相反，PCR的值是通过被TPM成为extend（扩展）的操作，我们在第2章已经讨论过了。从密码学的角度来看，这个过程如下：

PCR new value = Digest of (PCR old value || data to extend)

TPM将会把需要加进来的数据连接到旧的PCR值中。要被扩展到PCR中的数据几乎总是哈希值，尽管TPM并没有限制一定是哈希值。然后TPM会对这个刚刚连接好的值做哈希，然后将新的哈希值存储到PCR中。

系统重启以后，整个平台的运行由CRTM（Core Root of Trust Measurement）开始。CRTM会测量接下来将要运行的软件并将测量值扩展到一个偶数索引的PCR中。然后CRTM还会将软件的配置信息扩展到一个奇数索引的PCR中。这个被测量的软件，可能是BIOS，反过来又会测量和扩展它的下一级软件，这或许是MBR。这个测量链就这样继续下去直到早期的系统内核代码或者这个之后。这个过程中，重要的安全配置文件也会被测量。

最终测量的结果就是PCR的值表示了所有扩展到PCR中的测量历史。因为安全摘要机制的单向属性，索引没有办法撤销一个测量。（将PCR的值变成过去的某一个值）

表12-1是 PC客户端规范为PCR分配的功能。

表12-1. Example PCR Allocation
|PCR Number|Allocation|
|:---------|:---------|
|0|BIOS|
|1|BIOS configuration|
|2|Option ROMs|
|3|Option ROM configuration|
|4|MBR (master boot record|
|5|MBR configuration|
|6|State transitions and wake events|
|7|Platform manufacturer specific measurements|
|8-15|Static operating system|
|16|Debug|
|23|Application support|

测量过程的安全性取决于CRTM的安全性。CRTM作为第一个运行的软件，它是不能被测量或者验证的。CRTM就是所谓的信任根。平台制造商可以通过将CRTM存储于ROM中，让它变得不可修改来保护CRTM，如果不在ROM中也可以设置禁止升级来保护。但是因为有可能因为bug需要修改CRTM，另一种方法就是要更新的代码要附带签名，CRTM在更新自己之前先验证这个签名。

Linux的完整性测量架构（Integrity Measurement Architecture）在内核中集成了启动时测量功能。IMA策略决定了哪些软件需要被测量。这些被测量的软件通常包括库文件和启动时以root权限来执行的代码，以及决定Linux启动路径的配置文件。IMA通常情况下不会测量用户态的应用程序。

### PCR的数量
在实际应用种，一个TPM设备会有多个PCR。PC客户端平台要求TPM实现最少24个PCR，这也是PC平台实际具有的数量。自动化设备的TPM可能会有更多PCR。平台相关的TPM规范会指定PCR的属性，平台相关的软件规范规定各个PCR用于测量哪些软件。

平台相关的规范可能会给用户软件分配几个PCR。还有一个PCR（16），叫做调试PCR，用于软件测试。因为是调试的原因，这个PCR的值可以不在TPM重新上电的情况下被复位。


第11章我们已经介绍过，TPM2.0允许用户定义NV扩展索引，这实际上就是PCR。它们在哈希算法，口令和policy等方面可以灵活地被独立配置。虽然这种索引的元数据是非易失性的，但是可以通过使用混合索引让实际的数据在大多数情况下都存储在易失性内存中。

本章的后续小节的描述仅针对规范架构定义的PCR。

### PCR命令
PCR相关的命令如下：
* TPM2_PCR_Extend：几乎是PCR最常用的命令，它用户将一个摘要值扩展到（extend）到PCR中。
* TPM2_PCR_Event：让TPM来做哈希并将哈希值extend到PCR中。这个命令要求输入消息的长度最长为1024字节。
* TPM_PCR_Read：读取一个PCR的值，这在我们后面要介绍的验证事件记录中很有用。
* TPM2_PCR_Reset：复位一个PCR，主要用于为用户层软件分配的PCR（比如前面刚刚介绍的debug用PCR）。大多数的PCR在TPM一个上电周期内是不允许被复位的。
* TPM_PCR_Allocate：为PCR设定哈希算法。如果需要修改默认的哈希算法，需要使用这个命令，并且大多数情况下只需要执行一次就足够了。
* TPM2_PCR_SetAuthPolicy：为一个PCR组设定一个授权policy。在PC客户端上并不需要这个命令。
* TPM2_PCR_SetAuthValue：为一个PCR组设定一个授权口令。在PC客户端上并不需要这个命令。

### PCR用于授权
PCR的一个常用功能是授权。一个TPM实体可以拥有一个如下的policy，只有当特定的PCR的值是一个特定值时才允许使用这个TPM实体。第14章将详细介绍这个功能。这个policy可以选择一组PCR，每个PCR制定不同的值。如果PCR的值与设定的值不同，policy就不会满足，因此相关的TPM实体就不能被访问。

*************************
* #### 应用案例：将硬盘加密秘钥与平台的状态绑定
*************************

把全磁盘加密软件的秘钥存储在TPM中比存储在磁盘上并且仅使用口令保护要安全很多。首先，TPM能够防止暴力攻击（第8章有详细介绍TPM防止字典攻击的细节），这使得针对口令的暴力攻击不会成功。如果软件使用一个强度较弱的口令保护秘钥，那这个秘钥会很容易受到攻击。其次，将秘钥存储在磁盘上很容被窃取。拿到硬盘也就意味着拿到了秘钥。而如果使用TPM存储秘钥，如果要想窃取秘钥就需要整个首先窃取包含TPM的整个平台，或者至少先窃取硬盘和主板。

密封（sealing）操作可以让秘钥不仅可以被一个口令保护，还可以被一个policy保护。一个典型的policy会将秘钥锁定在sealing操作时的一个PCR值上（代表了软件的状态）。这个方案还假设系统启动的状态没有变化。任何事先植入的恶意软件都将在启动过程中被测量到PCR中，这样一来秘钥就会在不安全的状态下继续被密封。一个有较低可信度的公司可能会有一个自己的磁盘镜像，它可以将秘钥密封到代表这个镜像状态的PCR中（可以理解为镜像启动以后PCR的值）。这些PCR的值可以在更安全的平台上事先被计算好。一个更加复杂的方案是公司使用TPM2_PolicyAuthorize命令，并提供用于授权一组可信PCR值得凭据（Tickets）。参考第14章的内容来了解更多关于使用Policy授权来解决PCR脆弱带来的问题。

尽管使用一个口令也可以保护秘钥，但是即使没有TPM秘钥口令（以上的方案）可以增加系统的安全性。一个攻击者可能不需要一个TPM秘钥的口令就能启动系统，但是没有用户明和登录密码他还是不能登录。OS的安全特性可以用于保护数据。但是攻击者可以启动另外一个OS，比如通过DVD或者USB设备而不是硬盘启动，这样一来就可以越过OS登录的安全保护。还好在使用TPM的情况下，这种不同的启动配置（不从硬盘启动）和不同的OS软件将会改变PCR的值。又因为这些被改变过的PCR值不能和之前正确的PCR值匹配，所以TPM不会释放解密磁盘的秘钥，进而硬盘上的数据文件也就不会被解密。

以下是密封操作的步骤：
```
1. 构建一个Policy，使用TPM2_PolicyPCR，选择将来解封（UNseal）秘钥时的PCR值作为输入。
2. 执行以下操作（与TPM1.2类似）
  * 使用TPM2_GetRandom的结果作为对称秘钥，秘钥在TPM之外使用。
  * 使用TPM2_Create命令创建一个TPM秘钥，将刚刚的对称秘钥作为输入的Secret信息，并选择第1步的policy授权。
  * 或者（new TPM 2.0 alternative）
  * 使用TPM2_Create，仅仅使用Policy授权，通过改变命令的配置信息，让TPM自己生成对称秘钥并密封到当前创建的对象中。
```
 如下的操作用于解封秘钥：
```
   * TPM2_Load加载TPM2_Create创建的对象。
   * TPM2_PolicyPCR来满足Policy。
   * TPM2_Unseal返回被密封的对称秘钥。
```
------------------------------------------------------

*************************
* #### 应用案例：VPN秘钥
*************************

与之前的案例类似，一个VPN私钥可以被锁定（密封）到一个PCR上。只有在平台处于正确的状态时（没有被非法篡改启动）TPM才会允许VPN连接公司专用网络。

------------------------------------------------------

*************************
* #### 应用案例：将一个秘钥由OS环境下安全地传递到一个没有OS的环境
* *************************

一个平台管理员可能想授权终端用户修改BIOS设置，这可能是修改启动顺序。这是BIOS就需要管理员口令。这个时候管理员必须向BIOS（没有OS的环境）传递一个访问权限很高的口令，但是同时又不希望用户能看到。

管理员可以将口令密封到一个PCR中，PCR中存储了BIOS正在运行时的平台状态。管理员可以将用于密封管理员口令的口令在OS中传递给用户。用户不能再OS环境下解密管理员口令，但是BIOS却因为有正确的PCR值而可以解密（当然用户需要输入不那么重要的“用于密封管理员口令的口令”提供给BIOS）。

以下OS中的操作步骤：
```
1. 使用TPM2_PolicyPCR构建一个policy，选择PCR[2]全零作为解封时的PCR。这个PCR只有在启动早期的值为0，这时CRTM刚刚把控制权交给BIOS的第一部分代码。
2. 使用TPM2_Create，输入一个口令和刚刚的policy来创建一个密封对象。口令通过一个加密会话传递（参见17章），实际上就是通向TPM设备的安全通道。
```
以下是BIOS阶段的操作：
```
3. 使用TPM2_Load加载对象。
4. 使用TPM2_PolicyPCR来满足Policy。
5. 使用TPM2_Unseal来返回管理员的口令。
```

------------------------------------------------------

尽管PCR的典型的应用就是将一个TPM实体的使用授权与平台软件的状态绑定，但是PCR还有其他可能的应用。比如说，可以将一个口令扩展到PCR中，从而解锁一个实体的访问。当不需要访问时，就把PCR复位（如果允许复位操作的话）或者扩展其他的值。

## PCR用于认证
PCR用于认证是一种高级的应用案例。在没有TPM的平台上，远程应用软件通常不能决定平台软件的状态。如果平台软件状态是通过软件来报告的，如果这个软件被攻击，它就可以欺骗远程应用软件。

TPM认证功能为软件状态提供基于密码学的证据。回想一下，我们曾今介绍过，大部分的PCR（除去debug PCR）中的测量状体是不能被撤销的。具体来说就是PCR不能被撤回到曾经的某一个值。认证的过程就是一个TPM引用（Qoute）操作：将一组PCR的值做哈希，然后使用TPM密钥对哈希进行签名。如果远程一方可以验证这个用于签名的密钥确实来自一个真实的TPM，那就可以确认平台报告的PCR摘要没有被篡改过。

我们说这个认证过程是一个高级的应用案例是因为，仅仅验证签名和签名密钥的证书是不够的。远程的一方接下来还要验证PCR的摘要值能与报告的PCR匹配。这是显而易见的。

再接下来，远程的一方还要读取一个事件记录，记录包含很多测量过的软件等信息，以及所有的测量值。然后会执行类似于PCR Extend的操作得到最终的哈希值，然后与报告过来的PCR值比较。这些操作仍然不算难，它仅仅包含了一些数学计算。

TCG基础设施工作组（Infrastructure Work Group）和PC客户端工作组指定了事件记录的详细格式。IWG的平台可信服务规范定义了通过可信网络连接（Trusted Network Connect）报告测量值的方法。将记录和报告的格式标准化有利于标准软件在认证过程中解析和验证记录。

Linux的完整性测量架构模块（IMA）指定了一个事件记录文件格式。通常的格式包含如下内容：PCR索引，一个模版哈希，一个模板名称，文件的哈希值，文件的完整路径：

10 88da93c09647269545a6471d86baea9e2fa9603f ima
a218e393729e8ae866f9d377da08ef16e97beab8 /usr/lib/systemd/systemd

10 e8e39d9cb0db6842028a1cab18b838d3e89d0209 ima
d9decd04bf4932026a4687b642f2fb871a9dc776 /usr/lib64/ld2.16.so

10 babcdc3f576c949591cc4a30e92a19317dc4b65a ima
028afcc7efdc253bb69cb82bc5dbbc2b1da2652c /etc/ld.so.cache

10 68549deba6003eab25d4befa2075b18a028bc9a1 ima
df2ad0965c21853874a23189f5cd76f015e348f4 /usr/lib64/libselinux.so.1

接下来是最困难的部分。远程软件通过一个TPM签名过的认证了解了平台的软件状态。现在它需要决定这个软件的状态是否是安全的。这时候它需要将测量的哈希值与一个白名单比较，这就潜在地需要和第三方软件提商合作。

这就是可信计算概念本质。PCR仅仅提供一种可信的用于表示平台软件状态的方法。但是它们本身并不会做关于软件是否安全的判定。


*************************
* #### 应用案例：引用（Quote）
*************************

一个网络设备想要决定是否允许客户端平台连接一个网络。它需要知道客户端平台上运行的软件是否打了完整的软件补丁（patch）。此时网络设备就可以引用TPM的PCR然后与一个打过patch的软件模块的（哈希）白名单比较。如果平台是正常的，它就允许连接网络。如果不是，平台的网络连接将被路由到一个特殊的patch服务器，但是不能访问网络。

开源的VPN解决方案StrongSwan可以使用TCG TNC标准，从而结合TPM引用和一个policy为VPN的连接增减访问控制。

------------------------------------------------------

杀毒软件Kaspersky的用户协议（End User License Agreement）允许软件报告处理过的软件，软件的版本以及其他信息。这个协议允许使用TPM来认证这个报告，当然是平台有TPM设备的情况下。

### PCR引用详解
详细地检查引用数据是很有意思的。通过这些数据，用户可以明白引用（Quote）操作的安全属性。一个引用的数据结构（数据结构是指被做哈希和签名的结构）包含以下内容：
* Magic Number TPM_GENERATED：这可以阻止攻击者使用限制性的签名密钥签名任意数据，然后生成这是一个TPM引用操作。参考第10章了解关于限制性签名密钥和TPM_GENERATED相关的交互。
* Qualified name of the signing key(签名密钥的Qualified名称)：一个强度很高的密钥可以被一个使用相对较弱的算法的父密钥保护。Qualified名称代表了密钥的整个祖先（ancestry）。
* Extra data provided by the caller：这个数据通常是一个用于防止重放攻击的nonce，这个nonce用于证明这个引用操作只适用于现在这一次操作。
* TPM firmware version：包含这个数据以后，调用者就可以自行决定是否信任某一个版本的TPM固件。
* TPM clock state：resetCount对于下一个应用案例是很重要的。基于隐私保护的考虑，时钟信息被非背书组织架构下的密钥签名时会被做混淆处理。这不是漏洞，因为认证申请者只想要知道resetCount是否已经改变，而不需要读具体的值。
* 认证数据结构的类型（在这个案例中是Quote）。
* Quote操作中使用的PCR。
* 这些PCR的哈希值。


*************************
* #### 应用案例：检测两次数据传输之间的重启
*************************

假设一个平台正在执行财务相关的传输。一个监测设备每隔15分钟就会Quote一次用于检测平台软件状态的修改。但是，攻击者可能在两次Quote之间偷偷做一些事情，通过重启来执行恶意软件，然后发起一次未授权的数据传输，最后重启让平台恢复可信状态。这样一来，下一次正常的Quote将还会报告可信的PCR值。但是，resetCount能够检测到重启的发生，并且会（通过Quote）报告给监听软件。

------------------------------------------------------

### PCR属性
每个PCR都具备一些属性。TPM软件库规范定义了这些属性，但是平台相关的规范最终决定那个PCR拥有什么样的属性。通常来说，大部分的PCR都按照惯例分配给不同的特定软件来使用，但是也有一少部分没有被分配，它们是留给应用软件使用的。

PCR的Reset属性用于表示PCR的值是否可以通过TPM2_PCR_Reset命令复位。通常来说复位值是全0.大多数的PCR是不能被复位的，因为允许复位将可能导致恶意软件将PCR复位成一个已知的正常状态。有一些PCR只能在特定的Locality下被复位，动态信任根测量（Dynamic Root of Trust Measurement）过程有相关的要求。

PCR的Extend属性用于表示一个PCR是否可以通过TPM2_PCR_Extend或者TPM2_PCR_Event命令来扩展它的值。很显然，如果一个PCR不能被扩展那这个PCR就是没有任何用处的，只是有一些PCR只能在某些特定的Locality下被扩展（参考最后一章中介绍的TXT技术）。

通过DRTM来复位PCR的属性表明一个PCR是否可以通过直接向TPM的接口写数据来扩展它的值，而不是通过正常的TPM命令格式。这个属性既是平台相关的，又和具体的TPM硬件接口相关。这个属性通常在不同的Locality下有不同的设置。

当系统重启后使用CLEAR参数执行TPM2_Startup命令时，所有的PCR会被复位。大多数的PCR值会被复位成0，但是有一些时不同的值，比如说全1或者与执行startup命令时的Locality有关系。

No Increment属性是与TPM2_PolicyPCR绑定的。将一个Policy与PCR绑定是一个立即断言。执行TPM2_PolicyPCR命令时PCR当前的值会被增加到Policy会话的摘要中。但是，一个PCR在这个立即断言（PolicyPCR命令）之后有可能被改变，“正常”情况下，这样就会导致这个policy会话无效。“无效”是通过一个用于记录PCR值变化的计数器来实现的。Policy会话会记录执行TPM2_PolicyPCR命令时的计数值，然后在使用这个policy会话时检查它。如果计数值不相等，TPM就知道PCR的值已经改变了，Policy会话就会失败。

需要注意的是上一段中打引号的“正常”。TPM规范提供了No Increment属性。具有这个属性的PCR在PCR的值发生变化时不会增加这个计数器的值，因此也就不会产生前面说的Policy会话无效的情况。需要说明的是，大部分PCR是没有这个属性的，但是PC客户端规范给debug PCR和一少部分保留给应用软件使用的PCR设置了这个属性。


*************************
* #### 应用案例：PCR的No Increment属性用于VM
*************************

一个应用相关的PCR可以用于测量一个虚拟机。一个VM在例化时可以将PCR复位，在VM后续生命周期内会被频繁地扩展。如果每次Extend操作都使Policy会话变得无效，那TPM2_PolicyPCR命令就没有任何用处了。


------------------------------------------------------

*************************
* #### 应用案例：PCR的No Increment属性用于审计（Audit）
*************************

一个应用相关的PCR可以用于让一个审计记录（Audit Log）更加安全。参考第16章来了解这个案例的详细内容。当审计记录初始化时这个PCR的值会被复位，然后当log更新时PCR值也会被更新。同样的道理，如果每次Extend操作都使Policy会话变得无效，那TPM2_PolicyPCR命令就没有任何用处了。


------------------------------------------------------

### PCR授权和policy
与其他的实体一样，一个PCR可以拥有授权值或者Policy。软件库规范允许单独或者成组地对PCR设置授权信息。

但是，PC客户端的PCR并没有这个内容。访问PCR不需要授权。这样做的理由时增加授权会增加启动时间，启动时间对于PC来说通常是一个重要的参数。

### PCR算法
促成TPM2.0的第一个需求就是去掉TPM1.2中固定的SHA-1哈希算法。因为PCR与哈希算法关系很紧密，TPM2.0通过TPM2_PCR_Allocate命令理论上为PCR提供了许多算法相关的可能性。

这里的关键词是“理论上”。PCR可以按照bank来分配，每一个bank对应一种哈希算法。这个命令允许PCR按照任意组合来分配，一个PCR可以被分配到多个bank中，并且可以有多种哈希算法。如果是具有多种算法，除去PCR索引和摘要值以外，TPM2_Extend命令还要提供哈希算法参数。如果输入的算法与PCR配置的所有算法都不匹配，这个命令将会被忽略。

所以，理论上软件可以使用不同的算法做测量来产生不同的摘要，然后将这写摘要分别扩展到相应的bank中。那么PC客户端规范在实际中是如何做得呢？

这个规范要求一个包含所有PCR得bank。bank的算法默认被设置成SHA-1，但是可以被改成SHA-256.尽管一个TPM厂商可以自由实现更多复杂的组合，我们建议大部分的TPM都仅仅使用SHA-1或者SHA-256算法。这样相关的配套软件只要知道TPM的算法，然后就可以直接测量，得到摘要，然后将摘要扩展到PCR中。

更进一步讲，我们不希望TPM频繁修改哈希算法。实际上大多数的情况是TPM出厂配置成SHA-256，然后就永远使用SHA-256，或者是出厂时配置成SHA-1，然后当相关的软件更新到SHA-256时，同时把TPM的哈希算法更新成SHA-256。

## 总结
PCR有两个最基本的应用。它们的值可以通过一个签名的认证引用被报告出去，这样允许一个依赖方决定平台的软件状态是否可信。它们还可以用于policy，基于PCR的值来授权其他TPM对象的使用。相对于TPM1.2PCR的算法被固定成SHA-1，TPM2.0做了改进，允许使用其他的哈希算法。
