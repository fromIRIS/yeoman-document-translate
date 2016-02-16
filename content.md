## YEOMAN 文档翻译
### 第三节-用户交互
你的脚手架会和用户进行很多交互。默认情况下yeoman是跑在命令行里的，但这也支持不同工具能提供的传统用户界面。举个例子，没什么能阻止yeoman跑在一些IDE里或者跑在一些独立的APP里。

为了实现这种便利性，yeoman提供了一系列用户界面抽象元素。仅仅使用这些抽象元素与用户进行交互是你作为作者的责任。因为使用其他的方式可能会阻止yeoman正确的跑在不同的yeoman工具里。

举个例子，永远不使用`console.log（）`或者`process.stdout.write()`去输出内容是重要的。使用它们会导致用户在使用非命令行情况下无法看到输出信息。取而代之的是使用UI通用方法`generator.logo（）`,当`generator`是当前yeoman的上下文环境。

#### 用户交互
**Prompts**
`prompts`是跟用户交互的主要方法。`prompts`模块由[inquirer.js](https://github.com/SBoudrias/Inquirer.js)提供，你应该参考它的所有可用的API。

你可以这样调用prompt方法：
```
module.exports = generators.Base.extend({
  prompting: function () {
    var done = this.async();
    this.prompt({
      type    : 'input',
      name    : 'name',
      message : 'Your project name',
      default : this.appname // Default to current folder name
    }, function (answers) {
      this.log(answers.name);
      done();
    }.bind(this));
  }
})
```
这里我们使用`prompting`[队列机制](http://yeoman.io/authoring/running-context.html)取得用户的反馈信息。

**记住用户的参数选择**
一个用户每次执行脚手架的时候可能都会给出相同的输入，对于这类问题，你可能想要记住用户之前回答过的问题并且将这个输入当做新的默认输入。

yeoman拓展inquirer.js的API并增加了一个`store`属性。这个属性允许你指定用户提供的答案作为之后新的默认值。如下能实现：
```
this.prompt({
  type    : 'input',
  name    : 'username',
  message : 'What\'s your Github username',
  store   : true
}, callback);
```
note：提供一个默认值能防止用户提供空的答案。
如果你仅仅是寻找存储数据的方法，而不是跟prompt直接绑定，一定要去看见[yeoman storage 文档](http://yeoman.io/authoring/storage.html)

**参数**
参数直接通过命令行传递。
```
yo webapp my-project
```
这个例子中，`my-object`将作为第一个参数。

我们使用`generator.argument()`方法告知系统我们希望得到一个参数。这个方法接受一个`name`（string）和一个可选的对象集合。

这个`name`用于在generator上创建一个getter：`generator['name']`。

可选的对象集合接受多个键值对：
- `desc`参数的描述
- `required`布尔值 是否必须
- `optional` 布尔值 是否可选
- `type` String, Number, Array, or Object
- `defaults` 这个参数的默认值

这个方法必须在`constructor`构造器内调用。因此yeoman当用户调用你的generator的help选项时不会输出可用的帮助信息 e.g.`yo webapp --help`

这是例子：
```
var _ = require('lodash');

module.exports = generators.Base.extend({
  // note: arguments and options should be defined in the constructor.
  constructor: function () {
    generators.Base.apply(this, arguments);

    // This makes `appname` a required argument.
    this.argument('appname', { type: String, required: true });
    // And you can then access it later on this way; e.g. CamelCased
    this.appname = _.camelCase(this.appname);
  }
});
```
**可选项**
可选项跟参数长得很类似，但他们都书写为命令行标记。
```
yo webapp --coffee
```
我们使用`generator.option()`方法告知系统我们期待一个选项。这个方法接受一个`name`（string）和一个可选的对象集合。

`name` 用来取回匹配`generator.option[name]`的一个参数。

可选的集合（第二个参数）接受多个键值对：
- `desc` 可选项的描述
- `alias`可选项的缩略名
- `defaults`默认值
- `hide`布尔值是都隐藏help

以下是例子：
```
module.exports = generators.Base.extend({
  // note: arguments and options should be defined in the constructor.
  constructor: function () {
    generators.Base.apply(this, arguments);

    // This method adds support for a `--coffee` flag
    this.option('coffee');
    // And you can then access it later on this way; e.g.
    this.scriptSuffix = (this.options.coffee ? ".coffee": ".js");
  }
});
```

#### 输出的信息
输出信息通过`generator.log`模块执行。

我们主要使用到`generator.log`（e.g. `generator.log('Hey! Welcome to my awesome generator')`）他使用了一段字符串然后将此传送给用户。在命令行中这个方法跟`console.log（）`是类似的。你可以如此使用：
```
module.exports = generators.Base.extend({
  myAction: function () {
    this.log('Something has gone wrong!');
  }
});
```


### 第四节-composability可组合性
> 可组合性是一种将小片段合并成大家伙的一种方法。

yeoman为generators提供了多种方法依赖相同的功能。重复写相同的功能是没有意义的，所以一个API被提供可以在generators中使用其他generators。

在yeoman中，可组合性可用两个方式实现：
- 一个generator可以使用其他generator自我构建（e.g. `generator-backbone 使用 generator-mocha`）
- 终端用户也可以初始化一个组合（e.g. Simon 想要生成一个带sass和rails功能的backbone项目）NOTE：终端用户初始化一个组合是一个计划中的功能目前还没实现。

note：用户可组合性功能在版本0.17.0中开始实现，这是一个进行中的工作但是对于使用来说已经足够稳定了。未来的文档再进行细微的改良后尽快与大家见面。

#### generator.composeWIthin()
`composeWith`方法允许generator与其他generator一同运行。通过这种方式一个generator可以使用其他generator中的功能而不需要自己在重现实现一遍。

#### API
`composeWidth`三个参数
1. `namespace` - 一段字符串声明了generator构造时使用的命名空间。默认匹配已经安装在用户喜系统里的generators，使用[peerDependencies](https://nodejs.org/en/blog/npm/peer-dependencies/)去安装需要的generator
2. `options` - 一个对象包含`options`对象 或者/和 一个`args`数组。被调用的generator在执行时会接收这个参数。
3. `settings` - 一个对象声明组合的一些设置。generator在决定如何运行其他generator时使用这些参数。
  - `settings.local` - 字符串 定义被请求的generator的路径，允许sub-generators的使用，也允许特殊版本的generator的使用。要这样做只需在`package.json`中的dependencies中声明即可。然后引用路径，通常是`node_modules/generator-name`。
  - `settings.link` - 字符串 `weak（默认）或者strong`
`weak`字段 当组合性是被用户初始化时不执行，`strong`总是运行。

A weak link is for features unrelated to the core of the generator like backend frameworks or CSS preprocessors. A strong link is for features requiring an action to occur. An example is scaffolding a module by composing a route generator and a model generator.

当构造`peerDependencies`generator时：
```
this.composeWith('backbone:route', { options: {
  rjs: true
}})
```

当构造`dependencies`generator时：
```
this.composeWith('backbone:route', {}, {
  local: require.resolve('generator-bootstrap')
});
```
`require.resolve()`返回nodejs加载提供的模块时的路径。

#### dependencies or peerDependencies
npm允许三种类型的依赖：
- `dependencies`  安装本地的generator。这是最好的选择去控制使用的依赖的版本。
- `peerDependencies` 安装并列的generator，作为同辈。如果`generator-backbone` 声明 `generator-gruntfile` 是一个peer 依赖，文件树将像下面这个样子。
```
├───generator-backbone/
└───generator-gruntfile
```
- `DevDependencies` 用于测试和开发目的，这在这没有使用。

When using peerDependencies, be aware other modules may also need the requested module. Take care not to create version conflicts by requesting a specific version (or a narrow range of versions). Yeoman's recommendation with peerDependencies is to always request higher or equal to (>=) or any (*) available versions. For example:
```
{
  "peerDependencies": {
    "generator-gruntfile": "*",
    "generator-bootstrap": ">=1.0.0"
  }
}
```

### 第五节-管理依赖

一旦你执行了你的generator，你会经常执行npm和bower去安装一些你的generator需要的额外的依赖。

因为这些任务非常频繁，yeoman早就将他们抽离了出来。我们也会覆盖到通过其他工具是如何安装的。

值得注意的是yeoman提供的安装helper会在`install`队列中自动安排安装工作运行，如果你需要在他们执行过之后再执行，可以使用`end`队列。

#### npm 
你只需要调用`generator.npmInstall()`去执行一个`npm`的安装。yeoman会确保`npm install`命令在被多个generator多次调用的情况下只执行一次。

举个例子你想要安装lodash作为开发依赖：
```
generators.Base.extend({
  installingLodash: function() {
    this.npmInstall(['lodash'], { 'saveDev': true });
  }
});
```
这跟一下的代码是一样的：
```
npm install lodash --save-dev
```

#### Bower
你只需要调用`generator.bowerInstall()`去执行一个安装。yeoman会确保`bower install`命令在被多个generator多次调用的情况下只执行一次。

#### Both?
调用`generator.installDependencies()`去执行npm和bower。

#### 使用其他工具
yeoman提供了一个抽离允许用户`spawn`任何CLI命令。这个抽离会规范化命令，可以在linux、mac和windows系统间无缝运行。

举个例子，如果你是个php爱好者并且想要运行`composer`，你会这样写：
```
generators.Base.extend({
  install: function () {
    this.spawnCommand('composer', ['install']);
  }
});
```

确保在`install`队列里调用spawCommand方法，你的用户不会想等待安装命令的完成。

### 第六节-与文件系统交互
#### 位置环境和路径
yeoman 文件功能取决于硬盘上的两个位置环境。这些位置环境就是一些你的generator经常读取存放的文件夹。

##### 目的地上下文环境
第一个上下文环境就是目的地上下文环境。这个目的地就是yeoman将会用脚手架生成新应用的文件夹。这是你的用户项目文件夹，也是你书写大部分脚手架的地方。

目的地环境被定义为当前工作的目录或者是包含`.yo-rc.json`文件的最近父元素。`.yo-rc.json`文件定义了yeoman项目的根目录。这个文件允许你的用户在子目录中运行命令并且能工作良好。这确保了用户行为的连贯性。

你可以使用`generator.destinationRoot()`获取目的地路径或者使用`generator.destinationPath('sub/path')`获取基于目的地路径的路径。

```
// Given destination root is ~/projects
generators.Base.extend({
  paths: function () {
    this.destinationRoot();
    // returns '~/projects'

    this.destinationPath('index.js');
    // returns '~/projects/index.js'
  }
});
```
你也可以人为地设置目的地路径`generator.destinationRoot('new/path')`。但是为了连贯性，你不应该改变默认的目的地路径。

##### 模板上下文环境
模板上下文是你存储模板文件的文件夹，这通常是你将要读取、复制的文件夹。

模板上下文默认被定义为`./templates/`。你可以使用`generator.sourceRoot('new/template/path')`覆盖默认设定。

你可以使用`generator.sourceRoot()` 获取路径值，也可以使用`generator.templatePath('app/index.js')`在默认路径上添加路径。

```
generators.Base.extend({
  paths: function () {
    this.sourceRoot();
    // returns './templates'

    this.templatePath('index.js');
    // returns './templates/index.js'
  }
});
```

#### 一个在“内存中的”文件系统
yeoman在涉及覆盖用户文件的时候是非常小心的，基本上对每次预先存在的文件进行写操作都会经历冲突解决的过程，这个过程要求用户对每次覆盖内容的写操作都做验证。

这个行为阻止了突发情况降低了发生错误的风险。另一方面，这也意味着每次文件的写操作都是异步的。

因为异步的API很难使用，yeoman提供同步的文件系统API。每个文件都会被写入[内存中的文件系统](https://github.com/sboudrias/mem-fs)，当yeoman执行完后立马写入硬盘。

这个内存文件系统在所有构成的generator中都是共用的。

#### 文件功能
generator使用`this.fs`显示所有的文件方法。这是一个[mem-fs-editor](https://github.com/sboudrias/mem-fs-editor)模块。确保你查看过这个[模块的文档](https://github.com/sboudrias/mem-fs-editor)有可用的方法。

值得注意的是虽然`this.fs`显示了`commit`，你也不应该在你的generator中调用这个方法，yeoman会在循环运行的冲突中在内部调用这个方法。

##### 例子：复制模板文件
这是一个我们想复制并且处理模板文件的例子。
`./templates/index.html`给定的内容是
```
<html>
  <head>
    <title><%= title %></title>
  </head>
</html>
```
然后我们使用[copyTpl](https://github.com/sboudrias/mem-fs-editor#copytplfrom-to-context-settings)方法去复制文件并把内容作为模板。`copyTpl`使用[ejs 模板语法](http://ejs.co/)

```
generators.Base.extend({
  writing: function () {
    this.fs.copyTpl(
      this.templatePath('index.html'),
      this.destinationPath('public/index.html'),
      { title: 'Templating with Yeoman' }
    );
  }
});
```

当generator结束运行，`public/index.html`会包含：
```
<html>
  <head>
    <title>Templating with Yeoman</title>
  </head>
</html>
```

#### 通过流机制改变输出文件
generator系统允许你对每次文件写操作应用定制的过滤器。自动的美化文件内容，正常化空格等等都是可能的。

每当yeoman工作时，我们会将每个改动的文件都写进硬盘。这个过程通过一个叫[vinyl](https://github.com/wearefractal/vinyl)的流对象传递（类似[gulp](http://gulpjs.com/)）任何一个generator作者都能注册一个`transformStream`去改变文件的路径或者内容。

通过`registerTransformStream()`方法可以注册一个新的修改器。这是例子：
```
var beautify = require('gulp-beautify');
this.registerTransformStream(beautify({indentSize: 2 }));
```

需要注意的是任何类型的文件都会通过这个流管道。记得任何修饰流都会通过并不支持的文件。类似[gulp-if](https://github.com/robrich/gulp-if)或者[gulp-filter](https://github.com/sindresorhus/gulp-filter)可以帮助筛选不合适的文件类型并且通过他们。

基本上通过yeoman修饰流你可以使用任何gulp插件去处理正在工作的文件。

##### 遗留的文件功能
yeoman也显示了许多较旧的文件功能，你可以参考[API文档](http://yeoman.io/generator/actions_actions.html)了解更多。

古老的文件功能后来被移植到`in memory`文件系统，因此他们是安全且可使用的。保险起见，这些方法做了很多假设因此会产生一些边缘案例。可能的情况下尽量使用新的`fs`API

古老文件系统假设你想从template环境中读取文件并写到目的地环境，所以他不会要求你写完整的路径，他们自动决定了这一切。

同时，古老文件系统的方法：`template`和`copy`会自动的将一些模板作为数据对象传递给generator（e.g. this）

##### 提醒：更新已存在的文件内容
更新以存在的文件并不总是一个简单的任务。最可靠的方法是解析文件AST（[abstract syntax tree](http://en.wikipedia.org/wiki/Abstract_syntax_tree)）然后编辑他。最主要的问题就是编辑一个AST是非常麻烦和不可控制的。

一些流行的AST解析器：
- Cheerio for parsing HTML.
- Esprima for parsing JavaScript - you might be interested in AST-Query which provide a lower level API to edit Esprima syntax tree.
- For JSON files, you can use the native JSON object methods.

通过正则表达式去解析代码是冒险的，这样做之前你应该读读[this CS anthropological answers](http://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags#answer-1732454) 然后抓住正则表达式解析的缺陷。如果你选择使用正则表达式而不是ASTtree去编辑文件，请小心并且提供完整的单元测试。

### 第八节-存储用户配置
存储用户的配置信息并且在sub-generator之间共享是常见的。例如分享语言是常见的（用户是否使用cooffeeScript）书写规则的选项（空格缩进还是制表符缩进）

这些配置信息被存储在`.yo-rc.json`文件中，使用的是[yeoman storage api](http://yeoman.io/generator/Storage.html)。这些API在`generator.config`对象中能得到。

接下来就是你写普遍你可以使用的方法。

#### 方法
`generator.config.save()`
这个方法会吧配置信息写进`.yo-rc.json`文件中，如果文件没有存在，save方法会创建一个。

`.yo-rc.json`文件同样决定了项目的根目录。因为这个原因，即使你没有储存任何配置信息，你也最好在你的app文件中调用`save`方法。

同样save方法会在每次设置配置选项时自动调用，所以你无需显示地调用。

`generator.config.set()`
set方法可以传入一个键值对，或者含多个键值对的对象。值得注意的是值一定要是序列化的JSON

`generator.config.get()`
将key作为参数返回一个匹配的值

`generator.config.getAll()`
返回一个包含所有配置项的一个对象。
返回的对象是通过值传递的，而不是通过引用传递，这意味着你必须使用set方法去更新配置项。

`generator.config.delete()`
删除一个键

`generator.config.defaults()`
接收一个对象作为默认值，如果键值对已经存在了，则保持不变，若某个键是不存在的，则加上。

##### .yo-rc.json 结构
这个文件是一个JSON数据格式的文件，包含了多个被存储的generator的配置对象。每个generator配置对象都被命名以防出现冲突。

这也意味着每个generator配置对象都是沙箱，只能由各自对象的generator分享，你不可以使用storage API 给不同的generator分享配置。使用可选项和参数在不同的generator中分享数据。

这是这个文件的内部展示形式：
```

{
  "generator-backbone": {
    "requirejs": true,
    "coffee": true
  },
  "generator-gruntfile": {
    "compass": false
  }
}
```

这个文件对于用户来说是广泛的，这意味着你可能想在配置文件中存储高级的配置项，并在prompts失效时允许用户去编辑配置项。