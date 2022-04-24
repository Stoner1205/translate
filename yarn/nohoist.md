# nohoist in Workspaces（翻译）

原文地址：https://classic.yarnpkg.com/blog/2018/02/15/nohoist/

与[yarn workspaces](https://classic.yarnpkg.com/blog/2017/08/02/introducing-workspaces/)(工作区)一样，社区仍未完全理解monorepo(单仓多包)架构的hoisting(依赖提升)特性。这里介绍yarn官方提供的一种简单易用的[nohoist](https://github.com/yarnpkg/yarn/pull/4979)(禁用依赖提升)配置，允许workspaces之间存在相同的依赖冲突也能正常工作。

我们希望这个功能能减轻monprepo开发者的痛苦，并在效率(尽可能的hoisting)和可用性(锁定那些workspaces适用的依赖)之间取得平衡。

## 问题是什么？

首先，让我们看下hoisting在独立项目中是如何工作的：

为了减少代码冗余，多数依赖管理工具都会自带hoisting特性在安装依赖模块时将其进行提炼和扁平化，并将其安装在一个统一的目录。在项目中，依赖树将会被简化如下图：

![tree](https://classic.yarnpkg.com/assets/posts/2018-02-15-nohoist/standalone-2.svg)

通过提升，我们可以在同样的根目录`package-1/node_modules`下排除重复的“A@1.0”和“B@1.0”，同时保存不同的版本(B@2.0)。大部分的依赖检索器/加载器/打包工具可以从项目根目录开始通过遍历“node_modules”树来高效的定位本地依赖模块。

接下来我们看看monorepo项目，它引入的新分层结构不需要通过“node_modules”的硬链接。在这样的项目中，依赖可以被分散在多个不同的独立目录中。

![monotree](https://classic.yarnpkg.com/assets/posts/2018-02-15-nohoist/monorepo-2.svg)

yarn workspaces通过hoisting将依赖模块提升到项目根目录的“node_modules”中，达到在子项目/子包中共享依赖的目的。当考虑到这些子包之间也可能相互依赖(这也是monorepo存在的主要原因)，在解决这种代码的高度冗余问题时，这种提升优化显得更有必要。

## 依赖找不到了！！

尽管我们可以通过项目根目录的node_modules访问到所有的依赖，但我们经常在每个子包中构建其项目，而依赖模块可能在其自身的node_modules下并不存在，此外，并不是所有的检索器都会遍历[symlinks](https://github.com/facebook/metro/issues/1)。

最后，workspaces开发者们经常在构建子项目的时候会碰到找不到依赖相关的错误：

- 无法从项目根目录“monorepo”下找到依赖模块“B@2.0”(无法检索symlink)
- 无法从“package-1”下找到依赖模块“A@1.0”(无法识别“monorepo”中的依赖模块树)

对于monorepo项目来说，需要稳定的在任何位置找到依赖模块，它需要遍历每个node_modules树：“monorepo/node_modules”和“monorepo/packages/package-1/node_modules”。

## 为什么他们不能修复？

确实有很多方法可以让库作者解决上述问题，比如multi-root(译者没搞清楚multi-root是怎么解决此类问题的)，自定义模块映射，巧妙的遍历方案等...然而，

1. 并不是所有的三方库都有资源来兼容monorepo环境
2. 短板问题：JavaScript拥有庞大的三方库。然而，这也意味着复杂工具链的健壮性取决于其最薄弱的环节。只要在工具链深处存在一个不兼容的依赖将会导致整个工具无法使用。
3. 引导问题：例如，react-native提供了一种通过`rn-cli.config.js`配置multi-root的方法。但他并不支持`react-native init`或`create-react-native-app`这样的引导程序，后者在创建/安装应用前无法使用该功能。

令人沮丧的是适用于独立项目的依赖解决方案并不适用于monorepo环境。理想的解决方法在于解决底层实现，但现实很骨感，我们的项目等不起。

## 什么是“nohoist”？

是否有一种简单但通用的机制允许这些不兼容的依赖在monorepo环境中正常的工作？

事实证明是有的，并被贴切的命名为“nohoist”。这也在其他的monorepo工具(比如[lerna](https://github.com/lerna/lerna/blob/main/doc/hoist.md))中得到了验证。

“nohoist”能让workspaces正常消费那些并不兼容hoisting特性的三方依赖。原理是通过禁用指定的依赖模块被提升至项目根目录来实现。相对的他们将会被放置在实际的子项目中，就像一个独立的、non-workspaces的项目一样。

因为多数三方库已经能在独立项目中正常运行，因此在workspaces中模拟此类环境的能力应该能够解决许多已知的兼容性问题。

### 友情提示

尽管nohoist很有用，它也带来了缺点。最明显的一点就是nohoist的依赖会在本地重复存在，这会与上面提到的hoisting带来的好处相悖。

所以，我们建议在项目中尽可能的缩小和明确nohoist的范围。

## 什么时候可用？

该功能计划于1.4.2版本发布

## 如何使用？

我们可以很直观的使用nohoist。它通过在package.json中配置nohoist规则来实现。从1.4.2版本开始，yarn将使用一个包含可选nohoist设置的新的workspaces配置格式：

```
// flow type definition:
export type WorkspacesConfig = {
  packages?: Array<string>,
  nohoist?: Array<string>,
};
```

例如：

- 在项目根目录下使用nohoist：

    ```
    "workspaces": {
        "packages": ["packages/*"],
        "nohoist": ["**/react-native", "**/react-native/**"]
    }
    ```

- 在项目根目录下不使用nohoist：

    ```
    "workspaces": {
        "packages": ["packages/*"],
    }
    ```

- 在子项目中使用nohoist：

    ```
    "workspaces": {
        "nohoist": ["react-native", "react-native/**"]
    }
    ```

*注意：对于那些并不需要nohoist配置的童鞋也不用担心，原有的workspaces格式仍然支持*

nohoist规则是一个[glob patterns](https://github.com/isaacs/minimatch)的集合，用来在依赖树中匹配模块路径。模块路径在依赖树中是一个虚拟路径，并非真实的文件路径，所以并不需要在nohoist规则指定“node_modules”或“packages”。

## 图例

让我们来看一个简单的例子，这个例子解释了nohoist是如何在我们的monorepo项目中预防react-native被提升的。在“monorepo”项目下有3个子包，A、B和C:

![monorepo](https://classic.yarnpkg.com/assets/posts/2018-02-15-nohoist/monorepo-example-1.svg)

在执行`yarn install`之前的文件结构：

![fs](https://classic.yarnpkg.com/assets/posts/2018-02-15-nohoist/monorepo-example-file-1-a.svg)

项目根目录下的package.json文件：

```
// monorepo's package.json
  ...
  "name": "monorepo",
  "private": true,
  "workspaces": {
    "packages": ["packages/*"],
    "nohoist": ["**/react-native", "**/react-native/**"]
  }
  ...
```

让我们仔细看下这个配置：

### 范围：私有

nohoist只在私有包中可用因为workspaces只在私有包中可用。

### 匹配模式

yarn会基于初始(在hoisting之前)的子包依赖为每个依赖模块提供一个可用的模块路径。如果这个路径与nohoist提供的模式相匹配，那这个依赖将只会被安装在与其最近的子项目/子包下。

#### 模块路径

A

- monorepo/A
- monorepo/A/react-native
- monorepo/A/react-native/metro
- monorepo/A/Y

B

- monorepo/B
- monorepo/B/X
- monorepo/B/X/react-native
- monorepo/B/X/react-native/metro

C

- monorepo/C
- monorepo/C/Y

#### nohoist模式

“**\*\*/react-native**”：这个配置告诉yarn不要提升react-native包自身，不管它在哪。（shallow）

- “**”用来匹配react-native之前的0到n个元素，这意味着他会匹配任何的react-native不论它出现在路径的什么位置。

- 结尾带着“react-native”的模式决定了react-native下面的依赖，比如“react-native/metro”，将不会进行匹配，因此我们称其为“shallow”。

“**\*\*/react-native/\*\***”：这个配置告诉yarn不要提升任何与react-native相关的依赖库，以及这些依赖库相关的依赖。（deep）

- 结尾带着“\*\*”的模式，并不像上面提到的前置\*\*，该模式匹配react-native之后的1到n个元素，意味着只有react-native的依赖匹配这个模式，但不包括react-native自身。

- 并不只有react-native的直接依赖匹配该模式，他们的依赖也包括在内，因此我们称其为“deep”

合并这两种模式(shallow + deep)，他们会告诉yarn不要提升react-native和与它有关的所有相关依赖。

让我们试试其他模式：

- 如果我们只希望子包A下的react-native不做提升？

```
"nohoist": ["A/react-native", "A/react-native/**"]

```

- 如果在构建react-native应用的时候子包A需要包含子包C？

```
"nohoist": ["A/react-native", "A/react-native/**", "A/C"]

```

这将会在子包A的node_modules下创建一个symlink链接到子包C

### 提升后的文件结构

在执行`yarn install`之后，文件结构将会像下面这样：

![structure](https://classic.yarnpkg.com/assets/posts/2018-02-15-nohoist/monorepo-example-file-2-a.svg)

模块X和Y将会被提升到根目录因为“monorepo/A/Y”，“monorepo/B/X”和“monorepo/C/Y”并不匹配任何的nohoist模式。值得注意的是尽管“monorepo/B/X/react-native”匹配nohoist模式。“monorepo/B/X”也不匹配。所以react-native模块将会被留在子包“B”中而它的父级“X”将会被提升到项目根目录。

react-native和metro将会分别安装在子包A和B下，因为它们匹配react-native的nohoist模式。注意尽管B并不直接依赖react-native，它们仍将只提升到“B”下，就像一个独立项目一样。

## 如何关闭nohoist？

nohoist默认开启，如果yarn在一个私有package.json下读取到nohoist配置，它将会使用该配置。

想要关闭nohosit，你只需移除package.json中的nohoist配置，或者在.yarnrc配置文件中设置`workspaces-nohoist-experimental false`，也可以在执行命令的时候设置`yarn config set workspaces-nohoist-experimental false`。

## 举些栗子

现在你已经了解nohoist是如何工作的了，是时候上点真家伙了。。。

下面的测试项目是我们在开发nohoist功能的时候使用的。你们可以在[yarn-nohoist-examples](https://github.com/connectdotz/yarn-nohoist-examples)中使用：

1. 在yarn workspaces中创建react-native：=> 确认我们可以像在独立项目环境中一样遵循react-native的引导。

2. 创建一个同时包含react和react-native的monorepo项目：=> 确保nohoist可用，并尝试更多的高级用例。

上述测试项目，你可以跟随指示下载并运行它，如果它无法工作，请告知我们。

## 如何调查

如果得不到预期的效果怎么办？比如你本地的node_modules下有超多模块或者什么都没有？yarn提供了一个强大的命令“[why](https://classic.yarnpkg.com/en/docs/cli/why)”用来报告其hoisting原因供你调查。

这里有个[例子](https://github.com/connectdotz/yarn-nohoist-examples/tree/master/workspaces-examples/react-native#under-the-hood)给出了一个很好的说明。

## 结论

nohosit是一个新功能，它需要经过几次的打磨。如果发现有不对的地方请告知我们。它已经让我们的workspaces更加的简单易用了，希望对你来说也一样。

我们同时呼吁库作者们能够改写你们的库，使它能够兼容monorepo环境。或许有一天我们能够不再需要nohosit功能，同时所有的依赖都能被安装在正确的位置上。


## 参考

- nohoist原始提案：[RFC #86](https://github.com/yarnpkg/rfcs/pull/86)
- nohoist PR：[#4979](https://github.com/yarnpkg/yarn/pull/4979)
- workspaces介绍：[Workspaces in Yarn](https://yarnpkg.com/blog/2017/08/02/introducing-workspaces/)
