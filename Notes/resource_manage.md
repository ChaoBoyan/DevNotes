# iOS 开发中的资源管理问题
## 前言

在 iOS 开发中，资源文件管理是必不可少的一件事情。今天我和大家聊一聊下面这几个问题。

- 如何更高效的同步素材图?
- 如何更安全的读取素材图?
- 如何解决组件之间的重名问题？

---

## 更高效的同步素材图

说起 **更高效**，前不久特地找了几个朋友问了一下他们项目中同步素材图的方式，我大致列举一下:

- 乞丐版 
![](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage1.jpg)

- 入门版 
![](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage2.jpg)

- 升级版 
![](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage3.jpg)

- 极客版 
![](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage4.jpg)

总结了那么多，我只想说一句话，那就是... 

![(图片与本文无关)](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage5.jpg)

...那就是，我们可以借助**脚本**去优化上面的工作流，从而达到更高效。大致流程如下

![](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage6.png)

实际上工作原理很简单，就是借助了 **Xcode** 的 **Build Phases** 的功能，编写一个将图片从一个同步盘同步到项目中的 **Asset Catalog** 的脚本，在编译的时候 **增量** 同步，就可以很方便的管理图片文件了。

我用 Swift 实现了这个脚本，放在了 [GitHub - AutoAsset](https://github.com/Damonvvong/AutoAsset) 上，感兴趣的可以下载源码了解一下，当然你也可以直接使用到你的项目中，然后再也不用操心同步图片的事情啦。上面有使用方法和示例Demo，所以脚本的使用细节我就不展开讲了。

可能有人会担心这样的方式会减慢编译速度，但是我通过缓存上次同步成功的图片，再借助 diff 找出不一样的文件而去进行增量同步。就不会存在什么减慢编译速度了。

> 说了那么多，搞的好像不加这个你编译 Swift 会很快一样。😂

---

## 更安全的读取素材图

项目中读取一张图片大致有两种类型:

- 通过 **字符串** 读取

```Swift
let image = UIImage(named: "damonwong"）
```
对于这种方式而言，当图片删除时，可能会因为沟通不及时 **导致图片读取失败** 而没有发现，相对来说并不是一个 **十分安全** 的方式。

- 通过 **字面量** 读取

```Swift
let image = #imageLiteral(resourceName: "damonwong")
```
虽然通过字面量读取，在 Xcode 里面可以直接看到图片特别的形象，但是一旦图片删除，你又没有及时修改而上线了，**直接会导致奔溃** 🙃, 所以是一种 **极度不安全** 的方式。

**那么，如何才能更安全的读取一张素材图呢？**

> 用 **Strongly typed identifiers**

首先，先说一下什么是 **Strongly typed identifiers**。
- 对于一个字符串而言，你可以是任何字符比如 `damonwong` 或者 `damonvvong`, 编译器并不会帮你去检查这个字符串有没有什么问题。但是我们可以利用 **enum** 或者 **struct** 对字符串进行一次包装，强行让编译器去检查一下字符串有没有什么问题。比如：

```Swift
enum StronglyString: String {
    case damonwong
    case damonvvong
}

print(StronglyString.damonwong.rawValue)
print(StronglyString.damonvvong.rawValue)
```
这就是 **strongly typed identifiers**。

同时，对于 **图片名** 而言，是一个很好的 **Strongly typed identifiers** 应用例子。将所有的图片名都以 **enum** 的方式去包装，不但能检查图片重名问题，还能及时发现被删除的图片名到底用在哪些地方。

所以要怎么去做呢？照着 Asset Catalog 目录一个一个去码？No，你错了。像这种事情，还是可以用脚本搞定。

![](https://github.com/Damonvvong/DevNotes/blob/master/images/resource_manage7.jpg)

在 github 上有两个比较好用的库 [R.Swift](https://github.com/mac-cain13/R.swift) 和 [SwiftGen](https://github.com/SwiftGen/SwiftGen)，都很好的解决了从 **字符串** 到 **Strongly typed identifiers** 的自动工作。

两个库我都尝试了一下，大体功能都差不多，非要说一点不同的话，那就是:

- **SwiftGen** 相对来说依赖少一点，可配置空间更大，适合老项目引入并资源局部管理。
- **R.Swift** 配置简单且管理全面，适合新项目引入并资源全局管理。

最终的使用效果如下：

```Swift
// SwiftGen
let image = Asset.damonwong.image
// R.Swift
let icon = R.image.damonwong()
```

下面简单的描述一下如何使用 **SwiftGen**

- 首先，你要根据 [README.md](https://github.com/SwiftGen/SwiftGen#installation) 安装 **SwiftGen** 它可以用 CocoaPods 集成也可以直接安装命令行工具。
- 在安装好之后，在 **Xcode** 的 **Build Phases** 新增一个脚本。这里要注意的是脚本执行是有顺序的，先要执行上面的图片同步脚本 **SwiftGen**，再执行 **SwiftGen**。而且这两步是需要在编译之前执行的。
- 如果有定制化要求的朋友，也可以自己去修改 **SwiftGen** 的模板文件。 

结合 **更高效的同步素材图** 和 **更安全的读取素材图**，我们现在的流程就变成了:
> 1. 同步图片到 **Asset Catalog** 
> 2. **SwiftGen/R.Swift** 生成 **Strongly typed identifiers** 文件 
> 3. 编译器编译检验 - 如有问题抛出异常。

整个流程就相对更加高效和安全了。我也写了一个[示例 Demo - ResourceManageDemo](https://github.com/Damonvvong/DevNotes/tree/master/Demo/ResourceManageDemo)

---

## 组件之间的重名问题

最近项目在做组件化，遇到一个问题「如何解决组件之间的重名问题」。和小伙伴讨论了一下，大致有下面几种的解决方案：

- 1. 不用管，反正现在还有没重名文件
- 2. 将一个项目的所有文件都到放到一起，及时发现重名文件并修改
- 3. 通过命名前缀比如 `PodA_icon_image` 方式去约束
- 4. 组件之间单独管理资源文件

**方案 1** 是上来凑数的，肯定是不考虑的。
**方案 2** 是一个相对不错的解决方案，但是维护起来是相对于**一个项目而不是一个组件**，也就是说如果有一个新项目，那你需要知道这个项目所依赖的组件都需要哪些素材图。
**方案 3** 就是一个部门之间的协同与规范问题了，出于 **求人补不如求己** 的原则，最后也放弃了。
**方案 4** 就是我们最后的选择

> 剩余 50% 需要到 [SwiftOldDriver 精选](https://xiaozhuanlan.com/topic/0194576283) 查看

