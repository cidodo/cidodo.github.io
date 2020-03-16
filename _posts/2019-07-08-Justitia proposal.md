---
layout:     post
title:      Hyperledger Justitia Proposal
subtitle:   Justitia(fabric-admin)开源提案
date:       2019-07-08
author:     Arain
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - fabric
    - fabric-admin
    - justitia
---
# Hyperledger Justitia Proposal
## Project Identifier
Hyperledger Justitia可以帮助使用者创建和维护Fabric联盟  版本0.1

## Sponsor(s)
试金石信用服务有限公司, business@shijinshi.cn
<如果您希望作为赞助者加入其中，请通过邮件联系项目维护人员 liurui@shijinshi.cn>

## Abstract 
Justitia可以帮助联盟成员生成和维护自己的全部证书、简单方便的部署和维护自己的节点。另一方面，它可以帮助使用者管理联盟和通道的相关配置，其中包括成员和策略等配置。

## Motivation
最初，我们设想创建一个大的联盟链，联盟的所有参与者相互平等。他们各自维护自己在网络上的相关节点和证书，并且利用fabric多通道的特性联盟中部分或全部成员可以在不同通道内进行独立的业务合作。于是我们便开始尝试去构建这样一个网络，但是在实际操作过程中我们发现以运行脚本的方式去部署fabric节点的过程中配置复杂且容易出错，并且整个联盟难以扩展和维护。因此我们迫切需要一个工具来帮我们管理Fabric网络上的身份、节点、通道和联盟成员。
开发Justitia是为了帮助客户更加简单的部署fabric节点，以便于客户更加方便的加入到我们的业务中来。Justitia不仅可以帮助联盟成员管理他在联盟中全部的身份数据，还可以帮助使用者部署fabric节点、动态创建通道和管理通道以及联盟的成员。
当整个项目开发接近尾声的时候，我们考虑将其贡献给社区去帮助更多和我们有同样需求的用户。在着手准备贡献的过程中，我们在社区看到了一个类似的项目cello。在对其进行了深入的了解和对比之后，发现cello和Justitia对于fabric网络节点的部署方式存在很大差异。Cello可以非常简单方便的部署一个fabric网络，但是cello对于证书的管理和节点部署的灵活性不够好，另外它无法完成联盟成员和通道成员的管理。我们认为Justitia刚好可以弥补cello在一些场景下的不足，这更加坚定了我们想要开源这个项目的决心。当然，在后续的版本中我们也会尽可能学习cello的优秀设计。

## Status
这个项目将从孵化阶段开始。
本项目整体上已经实现了基本功能，但是还有部分功能不够完善有待优化，我们将在后续跟进这些部分。

## Solution
本项目使用Fabric ca管理和维护组织身份信息，借助Docker的http api维护和监控Fabric节点，通过fabric-sdk-java对fabric网络上的数据进行维护。整个项目的结构设计上分为四个部分，如下图所示。其中DB/File用于存储身份数据（包括证书和私钥）。Service定义了一些业务模块，提供给Scheduler调度。Scheduler进行身份校验，并定义了一系列REST接口；WEB通过Scheduler提供的REST接口进行数据展示和服务调用。
![justitia架构图](https://raw.githubusercontent.com/cidodo/image-host/master/imgs_for_notes/structure.png)
1.	Dao server
提供数据存储和读取的接口，可以根据实际需求使用不同数据库，默认使用MYSQL。
2.	Fabric server
封装了fabric-sdk-java，提供一些简单的服务接口。
3.	Chaincode server
通过调用Fabric SDK提供安装、实例化和升级等链码维护功能。链码安装支持上传本地合约和从代码仓库获取代码两种方式。
4.	Node server
通过调用Docker Rest API来创建和管理节点容器，使用者可根据需求动态调整节点数量。在部署节点时，系统为节点提供一个默认配置，使用者也可以根据实际情况覆盖这些配置。系统监控节点状态信息，当节点状态异常时系统通过邮件或短信等方式通知管理员处理相关情况。
5.	Identity server
身份信息维护借助于fabric ca来完成，包括证书的登记、续期和吊销。在数据库中我们会保存证书和私钥，以便当用户需要部署一个新的节点或使用一个新的客户端用户时，此服务可以为其提供所需的全部证书和私钥。当证书因为私钥泄露等原因需要使原有证书失效时，系统可以帮用户吊销相关证书，并将CRL更新到通道配置中去。
6.	Channel server
通道服务通过提交通道配置来管理联盟和通道成员。
由联盟管理员（掌握orderer节点管理员用户私钥），通过修改系统通道中的联盟信息，以达到修改联盟配置和增删联盟成员的目的。申请人通过本系统生成包含本组织身份信息的通道配置数据，然后将次配置数据作为申请材料通过邮件等其他形式向联盟管理员发起加入申请。若联盟管理员同意将其加入联盟（相应的审批机制有具体业务决定），便从orderer节点获取最新的系统通道配置区块，将申请者提交的配置信息增加到联盟中生成配置更新交易。此后申请者便作为联盟成员与其他成员拥有相同的权力，也可以创建通道。删除联盟成员则以相同操作，从联盟配置中去除于此成员的相关配置。
![Add member to consortium.png](https://raw.githubusercontent.com/cidodo/image-host/master/imgs_for_notes/Add%20member%20to%20consortium.png)
通道成员和通道配置的管理与管理联盟的方式类似，由通道成员的管理员用户发起修改通道配置的交易来实现。但是与修改系统通道不同的是，在默认策略下想要修改通道配置需要通道内超过半数的组织的管理员对修改提案背书，当然这个策略是可修改的。另一方面，组织管理员用户背书的过程无法在链上完成，如果引入一个中心化的服务帮助完成这个签名流程又太过复杂。为了在链上解决这个问题，我们在peer节点镜像中增加了一个系统链码CMSCC（channel manager system chaincode）辅助完成这个签名的审批流程。成员加入通道的过程如下图所示，过程与组织加入联盟类似。申请者向介绍人（通道中原有的任意成员）发起加入申请，介绍人获取通道最新配置区块生成通道更新交易。借助于CMSCC将此交易广播给通道中的其他成员，其他成员可以选择接受或者拒绝此申请。直到接受的成员数量满足通道策略时，介绍人将签名结果和更新交易发送给Orderer节点更新通道配置。此后申请者便作为通道成员与其他成员拥有相同的权力。删除通道成员则以相同操作，从通道配置中去除对应成员的配置。
![Add member to channel.png](https://raw.githubusercontent.com/cidodo/image-host/master/imgs_for_notes/Add%20member%20to%20channel.png)
 
## Efforts and Resources
目前我们已经将最初版本的代码托管到了GitHub( https://github.com/shijinshi/justitia)。虽然它现在很多地方使用了中文，而且文档也还不够完善。但是我们会根据本文提到的设计尽快重构出下一个完善的版本。另外，我们希望在下一个版本的时候能够将代码转移到github的hyperledger用户下，以便更多的人能够看到这个项目。值得关注的是，对于此项目我们已经成立专门的团队去管理和维护直到整个项目孵化结束。当然，我们也非常欢迎其他有兴趣的人员加入我们一起把它做得更好。

## How To
本项目是一个Spring boot + React实现的前后端分离项目。使用者可以选择通过编译源代码和运行docker镜像的方式来部署。部署完成以后可以通过浏览器访问它，第一次访问时我们提供了引导流程帮助使用者初始化系统配置。

## Closure
项目成功可以通过验证Solution部分的功能，检查这些功能是否完整和稳定。另外，随着后续版本的跟进，我们也可能在Justitia中加入新的功能。

## Reviewed By
- [ ] Arnaud Le Hors
- [ ] Baohua Yang
- [ ] Binh Nguyen
- [ ] Christopher Ferris
- [ ] Dan Middleton
- [ ] Hart Montgomery
- [ ] Kelly Olson
- [ ] Mark Wagner
- [ ] Mic Bowman
- [ ] Nathan George
- [ ] Silas Davis
