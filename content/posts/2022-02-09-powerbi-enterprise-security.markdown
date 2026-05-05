---
title: "Power BI 企业数据安全"
date: 2022-02-08T08:03:50+00:00
draft: false
tags:
  - Power BI
  - Azure Key Vault
  - Data Security
  - Sensitive Label
categories:
  - 技术笔记
author: "Davis Zhang"
featured_image: "/images/30.jpg"
avatar: "/images/Davis.jpg"
description: ""
---

# 前述

数据不仅是企业对资产，也是企业的命脉，其重要性不言而喻。而随着越来越多的企业采用了Power BI作为其商务智能解决方案之一，用户对微软BI体系中数据的安全与合规性的要求也愈加严格。因文章篇幅问题，下文将以PBI及M365管理员的角度，节选Power BI平台中较为核心的数据安全问题，即“数据寄存，数据呈现及数据导出”几个方面进行图文讲解。

# 一、PBI数据安全之数据寄存

当Power BI数据集发布到PBI Service后，其数据实际上存储在Azure Blob Storage, 在默认情况下，数据加密由Microsoft托管，而密钥的值，轮换规则以及加密方式则不受企业或组织的控制。尽管Microsoft使用了足够强大的256位AES加密算法，但在一些企业严苛的信息安全政策及合规部门严谨的审查工作面前，该说辞恐怕无济于事。为了使组织掌控密钥并使数据加密方式与轮换规则符合其政策，在Power BI实施BYOK（Bring your own Key）成为了唯一方案。

## 在PBI实施BYOK

> **注意：**
> 
>  - BYOK目前仅适用于PBI Premium (包括Capacity与Per User)
>  - BYOK主要支持以关系型数据库位数据源的Import数据集    (了解具体限制需参阅[此文档](https://docs.microsoft.com/en-us/power-bi/admin/service-encryption-byok#data-source-and-storage-considerations))


1.登录Azure创建Key Vault, 并分配Unwrap Key及Wrap Key权限

![在这里插入图片描述](https://img-blog.csdnimg.cn/cc3225f7c69346fd8f78f34e2a4e4235.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

2.在Access Policy添加你要用于BYOK的Power BI工作区管理员账户以及Power BI Service服务主体

![在这里插入图片描述](https://img-blog.csdnimg.cn/6ea985987cf543359da2b1bea7142036.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)


3.接下来按如下配置创建密钥

![在这里插入图片描述](https://img-blog.csdnimg.cn/79309571380f43e19421b137c34b8a2c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

4.现在，将上一步创建好的密钥应用到PBI容量，目前没有界面可供设置，需要在PowerShell中键入命令

```Python
--安装Cmdlets（已装跳过）
Install-Module -Name MicrosoftPowerBIMgmt
--登录
Connect-PowerBIServiceAccount 
--在整个PBI租户中启用BYOK
Add-PowerBIEncryptionKey -Name '<输入Key名称>' -KeyVaultKeyUri '<输入Key URI>'
--注：其中Key URI可在Azure Key Vault中查询复制
--获取容量ID
Get-PowerBICapacity -Scope Individual
--最后，为该容量设置BYOK
Set-PowerBICapacityEncryptionKey -CapacityId <输入容量ID> -KeyName '<输入Key名称>'
```

至此，BYOK即设置完成，输入以下命令检验：


```Python
Get-PowerBIEncryptionKey
```

如图显示即配置成功：

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd751b7a7d9a44a5991c526013cee56a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

需要注意的点是，目前BYOK的实施是在整个容量层面的，而非Workspace层面的。 一旦为容量设置了BYOK，则该容量下所有的工作区都继承容量的BYOK设置。

6.秘钥轮换，可使用如下命令，若要实现定期轮换，可将其配置到已有应用程序，或直接使用Azure Key Vault 中 "Rotation Policy" 预览功能。

```Python
Switch-PowerBIEncryptionKey -Name '<输入Key名称>' -KeyVaultKeyUri '<输入新的Key URI>'
```

此处需要注意，无论是新版秘钥还是当前秘钥，都必须处于启用状态，否则将遇到如下报错。确保秘钥都启用后，问题解决，如图：

![请添加图片描述](https://img-blog.csdnimg.cn/1c3f4a5898c24d95a68b50f0694ef9c6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

# 二、PBI数据安全之数据呈现

数据呈现阶段的安全，顾名思义，是指用户在访问PBI报表时，如何基于不同的自定义角色，给予不同用户或不同组设置对数据不同的访问权限，虽然我们可以在工作区决定是否为某用户授予Viewer角色，但无法控制该用户能看哪些数据以及不能看哪些数据。

幸运的是，在Power BI 导入模型中 ，微软提供了两种数据安全设置，RLS和OLS, 前者即PBI用户所熟悉的行级别安全性，其可以基于角色对不同表不同字段利用DAX定义过滤规则，也可以结合USERPRINCIPALNAME() 动态定义访客权限（由于资料众多，此处不展开）； 而后者OLS (Object Level Security) 则是针对字段本身设置的安全性，它可基于角色控制不同字段（包括字段名）对特定用户是否可见，我们可以形象理解为“列级别安全性”。

## 在PBI实施OLS

与RLS相同，OLS同样要先在PBID创建角色，但具体表或字段可见性的安全控制则需要在外部工具[tabular editor](https://tabulareditor.com)中完成。要对指定角色隐藏字段，只需使用该工具连接到模型后，选择需要设置的表或字段，将OLS中对应角色的值设置为None即可，如图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/1685a10a0d794ac8b9c1e4c3ff509c7d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

以下是关于OLS需要注意的点：

1. 如果对某角色隐藏的表或字段在前端报表中有使用，则该角色查看报表时，与之相关的所有可视化都不可用。

2. 虽然无法针对特定度量值实施OLS，但若度量值所引用的字段包含隐藏的字段，则该度量值对指定角色依然不可见。

下图展示的简单例子很好反映了上述两点。由于对角色“User”在字段“OrderQty”的OLS值设为了None，因此左侧的可视化完全不可用，右侧的表由于未使用该字段，也未使用任何引用该字段的度量值，因此显示不受影响。

![在这里插入图片描述](https://img-blog.csdnimg.cn/782f2bfcd49e4af88fcbb60cdaf1c33c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

由于受底层技术限制，当前PBID还无法做到在同一个可视化内，在隐藏特定字段或值的同时不影响其他字段或值的展现，因此，OLS的应用价值主要体现在了用户自助分析方面。

# 三、PBI数据安全之数据防护

通过在工作区指定角色配合RLS以及OLS，我们已经实现了对PBI数据内容的权限控制，但我们无法控制的是当数据从Power BI导出后，如何持续地保护数据，以及，如何对PBI数据进行监控以使得每当探测到敏感数据时，可以及时向管理员发出警报。

应对此情况的解决方案是对Power BI实施DLP (Data Lose Prevention) 。它包括以下两个阶段：

1.Microsoft 365管理员可以为Power BI创建和定义敏感度标签，不同的敏感度标签可以设置不同的权限定义，PBI各工作区的管理员可以据情况为不同的数据集，数据流以及报表设置对应敏感度标签，以实现敏感数据即使离开了 Power BI，也能得到保护。引用[MS文档](https://docs.microsoft.com/en-us/power-bi/admin/service-security-sensitivity-label-overview#introduction)的话讲：

> When labeled data leaves Power BI, either via export to Excel, PowerPoint, PDF, or .pbix files, or via other supported export scenarios such as Analyze in Excel or live connection PivotTables in Excel, Power BI automatically applies the label to the exported file and protects it according to the label's file encryption settings. This way your sensitive data can remain protected, even when it leaves Power BI.

2.Microsoft 365管理员可以为Power BI创建DLP策略，该策略可以基于PBI内容的敏感度标签对敏感数据进行监控 (该功能尚处于预览阶段)。

> 注：实施DLP需要Microsoft 365 E5 订阅

## 在PBI实施DLP

#### 创建并发布敏感度标签

1.在M365合规中心按下图创建标签。

> 注：如果Information protection选项卡不可见，则需要检查下你的账户权限以及M365 Lisence

![在这里插入图片描述](https://img-blog.csdnimg.cn/fe4904afd34640f58ec2478915a9fd89.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

2.选择Files & emails

![在这里插入图片描述](https://img-blog.csdnimg.cn/5af6a2837d204b449ecb45434ef46136.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

3.此步骤，即是决定敏感度标签应用后的权限设置，可以添加用户，安全组甚至域，为他们分配不同的角色。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c5a9957966ae428385ac8edf6ca8c3f5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

下图明确了不同角色的权限范围，也可以通过勾选具体项自定义角色的权限。
![在这里插入图片描述](https://img-blog.csdnimg.cn/f089eefdcc9444f2bb9a860edf000d1a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

4.标签创建完成后，发布标签：

> 注：可以在已有敏感度标签创建子标签，子标签可以在继承父级的权限范围的基础上对权限进行进一步定义，这将对大型组织十分有用

![在这里插入图片描述](https://img-blog.csdnimg.cn/72f77f5414cb48b6b5ee2dc07591d952.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

在发布标签过程中，可以勾选此项，这将强制Power BI开发者在发布数据集与报表前对其应用敏感度标签，下文“PBI敏感度标签强制实施验证”部分将展示其效果：

![在这里插入图片描述](https://img-blog.csdnimg.cn/f6416e2f99e8436691d3954048379ccf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

回到Power BI Service, 发布的标签已经可用：

![在这里插入图片描述](https://img-blog.csdnimg.cn/43938550ed2149689cb628ac7c7fa387.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)


> **注意：**
> - 敏感度标签发布后，通常不会立即在各个应用中显示。如下图1已经说明“It can take up to 24 hours...”，因此，如果没有在PBI看到发布的标签，并不一定代表你的设置不成功，只需耐心等待。
> - Power BI管理员需要在租户设置中启用敏感度标签（见图2）。

![图1](https://img-blog.csdnimg.cn/5b30a66fe4e34053b9220627f7dc5d1b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

![图2](https://img-blog.csdnimg.cn/a26ef59258664f72b1d83ca4eaec22c1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)


#### 验证

1. PBI敏感度标签强制实施验证。

	如下用户必须先为PBIX应用敏感度标签，才可发布报表。
![在这里插入图片描述](https://img-blog.csdnimg.cn/a601224cd874434fa119d85516f70375.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

2. 数据导出保护效果验证

	如图，我已为数据集，报表以及分页报表指定了敏感度标签，其中HIGH SENSITIVE只有特定用户才能查看导出的数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/0160bed5bba948f0bf21ad03b1f65127.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

【示例1-1】Power BI报表导出验证（使用有权限的账户）

![在这里插入图片描述](https://img-blog.csdnimg.cn/e24b1194735d4cc3ac8a998d871a4a0e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

【示例1-2】Power BI报表导出验证（使用无权限的账户）

![在这里插入图片描述](https://img-blog.csdnimg.cn/183444e8d6254cf39e7fefe793d88b2d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

【示例2】Power BI分页报表（Paginated Report）导出验证

![在这里插入图片描述](https://img-blog.csdnimg.cn/765c9eb7e2b0492387ff5111792180c7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)


#### 创建DLP策略

DLP策略在Power BI尚为预览功能，部分特性暂时不支持，如果公测在今年四月，那么GA可能又要往后推，下文仅就策略的创建部分做部分说明。

![aaa](https://img-blog.csdnimg.cn/ec99a4ee3f0a443b989b22dc8b002728.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

1. 在M365合规中心，从Custom Policy开始创建：

![在这里插入图片描述](https://img-blog.csdnimg.cn/44b79b92e90f4b468f16dfb62c9b0989.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

2. 选择Power BI，DLP默认应用到整个容量。如果仅希望DLP应用在个别工作区，可以输入对应的WorkspaceID:

![在这里插入图片描述](https://img-blog.csdnimg.cn/75033d2569b6452db233fda4ae3c7c0b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

3. 创建DLP策略警报触发规则，可以仅就标记了某特定敏感度标签的PBI内容实施：

![在这里插入图片描述](https://img-blog.csdnimg.cn/cdb97b6f35c844339f1a97da63a5c0d8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

4. 管理员可在此处进行数据监控：

![在这里插入图片描述](https://img-blog.csdnimg.cn/e44cf4f830ec4e02a9ebf04c494279d7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_RC1CSSB8IERhdmlzIG9uIEJJ,size_20,color_FFFFFF,t_70,g_se,x_16)

## 其他

除上述内容外，有关Power BI安全性的技术点还有很多，如网关安全性，私有链接集成，Azure Log Analytics集成等等，要逐一讲清这些内容，足够写完一本书了，而国内外关于该领域的研究非常少，一些坑，一些bug，即使搜遍谷歌也难以有对应解决办法，还是要依赖于读者的耐心探索，并在必要时获取微软技术人员的支持。

## [附]相关链接
   
 -  [Power BI security white paper](https://docs.microsoft.com/en-us/power-bi/guidance/whitepaper-powerbi-security).



