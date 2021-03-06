#iOS 开发框架的重要性

##背景
最近公司打算开发一个新 IM 通讯的项目, 借着这个机会我觉得自己必须得在框架整体框架搭建上做一下深入研究与反思了。不论是前端,后台,移动端,结合到现实生活中,处处都会涉及到框架的应用,一个初期就设计优良的架构,素容置疑在项目的可维护性,扩展性,健壮性肯定都有很大的提升。所以我们必须要在开发之前在架构上面多下功夫! 

对于一个项目来说 **架构** 一直是个值得关注的大话题, 这里我将其划分为几个部分去思考,设计。
**文件目录模块架构**,**Coding 设计模式层面架构**,**网络层框架架构**,**数据持久化架构**,**UIKit 层面架构**,暂时先以这几个重要的层面去考虑,和设计。我们来依次讨论这几个模块的理解和设计方案。

##文件目录模块架构

对于一个项目来说,最直观的设计就是在文件目录结构上,一个好的文件目录结构划分能让我们自己在开发过程中更加快速编写代码,也能让新加入的同事更快速的融入到项目中来,你想一下如果文件目录划分清晰明了,和一个杂乱无章的目录结构,那个阅读起来舒服？其就像一本书的编写排版,读完排序之后读者基本读整本书内容有一个清晰的认识,以及在后面阅读起来随时能通过排序来找到相应的章节.

所以在开发前期我们都应该根据我们当前项目所有涉及到的业务场景,需要使用到的模块心里都要有一个准备,参考了一些开源项目,以及网上关于项目目录划分的文章, 主流的划分逻辑大概是分为两种.

1. 按项目内每个大的模块功能来进行文件夹划分. 
2. 按主要的设计模式来进行划分,例如: ViewController. View.Model VS ViewModel等. 

个人更倾向于第一种, 举个栗子. 以**支付宝**为例,项目 TabBar 分为 5 个大模块, 一个是首页,财富,口碑,朋友,我的. 然后在依次建立 5 个相应的文件夹,当然还会有一些涉及功能很多的模块,比如项目里面的商城,生活号,小程序等,虽然可能都包含在这 5 个大的模块里面, 我们可以根据实际场景结合业务, 如果某个模块业务涉及场景和业务比较复杂, 都可以单独为其建立一个独立的文件夹, 与其他 5 个模块进行平级别的划分。
除了对模块进行划分外,我们在对项目里面肯定都会包含的网络层,数据持久层,工具类库层,三方资源层等等...这些在项目中使用率高的核心层面,也单独划分成一个文件夹的形式,方便以后新增代码的时候,将其按功能点来添加和划分。 
这样我们开发过程中查找相应类文件,新增的时候也不会浪费太多的时间,也方便其他人阅读。

## 设计模式层面架构

这里主要是涉及到 设计模式的选择上, 目前比较主流的有 MVC.MVVM.MVP。然后我参考了知名应用 **小猿搜题库** 出的设计模式架构篇, 其主要是围绕 MVC 和 MVVM 将其优缺点进行一个优良改造版. 在两种架构中权衡而产生的架构**MVVM without Binding with DataController**, 我觉得很值得大家去参考,学习.

关于在设计模式上, 我觉得还是应该结合自己实际业务场景去进行选择最好, 不需要太死板要求为了遵守这种设计模式而遵守。每一种设计都有其优缺点, 我们应该权衡其利与弊抱着学习的态度来对待, 只要其能真正解决我们的痛点就是一个好的设计模式, 在选择上面我比较倾向于项目中多种设计模式公用的做法。 具体根据当前业务场景来。

##网络框架架构

为什么要对网络层进行封装. 网络层的代码几乎遍布我们每一个页面, 想象一下假如我们不对网络层进行一个良好的封装, 每次调用原生库 NSURLSession 来请求一段网络大家会是什么心情？ 在想象一个每次我们请求网络的时候基本都会添加一些公共参数,缓存处理,统一错误码等处理吧, 还有请求返回格式化设置等等. 这些都是基本很少变动的配置, 我们对其都做好封装,上层直接传入请求方法,URL,进来即可. 

上面说的比较浅, 相信大家在项目中对网络层封装至少都 2 层以上, 本人主要是对 AFNetWorking 进行的一个封装, 第一层封装主要对 AFHttpSeesionManage 的初始化设置以及安全策略,证书校验,还有请求超时处理等. 第二层则是直接获取网络的类,对其做了一些添加公共参数,处理返回数据公共逻辑,返回结果错误码,添加缓存,缓存处理等。 

这样做的好处有几点. 
1. 假如有一天 AFNetWorking 不更新维护了, Apple 弃用了其最新版本的基于 NSURLSession 的网络请求类, 我们可能需要考虑更换一个新的底层网络库. 我们只需要更改第一层封装库的代码即可。 **大大降低了对 三方库的依赖关系;**
2. 通过第一层封装不仅完成了 降低依赖关系, 我们还做了公共的配置, 方便后续在做网络请求时, 不需要再重新配置, 实现只配置一次即可共全局使用的效果。 类似的第二层添加公共参数也是如此。
3. 第二层封装主要是我们在实际业务场景中,基本每个网络请求都会携带一些公共参数, 比如当前项目版本号,设备号,时间等,后台要求每次请求都要携带上这些公共参数, 我们将其封装起来。
4. 处理请求结果错误码,基本同理。 
#### 总结:不写重复的代码。

##数据持久化架构

由于项目是主打 IM 即时通讯的, 会涉及到很多数据都需要保存到本地, 以及读取缓存的操作, 所以后期肯定要在数据缓存处理,数据读取,数据存储上做精心设计. 这一块目前自己能力有限, 主要我们先参考大神文章为主. 留个坑, 后续我们在来补上, 希望有数据结构设计,缓存处理方案的大神能指一二。

##UIKit 层面架构

UIKit 层面涉及到的范围也比较广, 我主要从视图控制器下手进行封装, 采用 BaseViewController 的方式, 对 ViewController 和 UINavigationController 以及 UITabBarController 等进行一个 基类控制器的封装, 将一些统一的参数,业务等进行配置, 方便后续在项目中类似的控制器中, 直接通过集成 BaseViewController 就可以轻松实现的功能.

##结语

作者水平有限,文中可能有说的不对,或者错误的地方欢迎大家指正。

主要是通过参考网上比较出名的文章,结合自己项目实际场景来进行,后续我会把自己看到的相关资料统统放下面供大家参考! 非常感谢这些大神,前辈为我们提供如此精辟的技术分享,如同一盏黑暗里的明灯,不断指引我们走向光明大道！



参考资料: 
[iOS应用架构谈 view层的组织和调用方案](https://casatwy.com/iosying-yong-jia-gou-tan-viewceng-de-zu-zhi-he-diao-yong-fang-an.html)
[猿题库 iOS 客户端架构设计](http://gracelancy.com/blog/2016/01/06/ape-ios-arch-design/)  
[被误解的 MVC 和被神化的 MVVM](http://blog.devtang.com/2015/11/02/mvc-and-mvvm/)
[浅谈 MVC、MVP 和 MVVM 架构模式](https://draveness.me/mvx)
[浅析 iOS 应用网络层设计](https://skyline75489.github.io/post/2016-3-13_ios_networking_layer_design.html) 
[iOS应用架构谈 网络层设计方案](https://casatwy.com/iosying-yong-jia-gou-tan-wang-luo-ceng-she-ji-fang-an.html)
[做一个 App 前需要考虑的几件事](http://www.cocoachina.com/ios/20161025/17849.html)
[onmyway133/fantastic-ios-architecture 设计模式架构思想汇总](https://github.com/onmyway133/fantastic-ios-architecture)
[老码农冒死揭开行业黑幕：如何编写无法维护的代码](http://blog.jobbole.com/80241/)
[不错开源项目](https://www.zhihu.com/question/26292022)
[iOS 视图控制器转场详解](http://blog.devtang.com/2016/03/13/iOS-transition-guide/)
[今日头条：iOS 架构设计杂谈](https://juejin.im/post/5b2b1a73e51d4558b27782c0)
[优秀的 iOS 应用架构：MVVM、MVC、VIPER，孰优孰劣](https://academy.realm.io/cn/posts/krzysztof-zablocki-mDevCamp-ios-architecture-mvvm-mvc-viper/)
[iOS应用架构现状分析](http://mrpeak.cn/blog/ios-arch/)

##希望此篇文章对您有所帮助，如有不对的地方，希望大家能留言指出纠正。谢谢！！！！！
##学习的路上, 与君共勉!!!


