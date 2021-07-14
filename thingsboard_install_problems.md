本篇主要介绍在GitHub拉取代码后，通过`mvn install -Dmaven.test.skip=true`指令对**Thingsboard**源码进行编译安装过程中所遇到的常见问题。

**注：笔者所编译的版本为github 2021/7/7拉取的版本，ThingsBoard版本>=3.3.0，操作系统 Mac OS X**

1. ### could not find artifact org.thingsboard.dao-3.3.0-SNAPSHOT-tests.jar

   这是由于编译安装的时候，指定了`-Dmaven.test.skip=true`参数造成的，该参数会跳过所有module的测试环节，同时也会忽略到测试环境打的包，要解决此问题，只需要取消`-Dmaven.test.skip=true`参数或在`dao`目录下先执行不跳过测试的安装`mvn install`即可

2. ### `js-executor`安装失败

   常见的报错如下：

   ```
   AssertionError [ERR_ASSERTION]: Placeholder for not found
   at injectPlaceholder (C:\Users\XXX\AppData\Roaming\npm\node_modules\pkg\lib-es5\producer.js:46:37)
   at injectPlaceholders (C:\Users\XXX\AppData\Roaming\npm\node_modules\pkg\lib-es5\producer.js:66:3)
   at C:\Users\XXX\AppData\Roaming\npm\node_modules\pkg\lib-es5\producer.js:224:9
   at FSReqCallback.oncomplete (fs.js:155:23) {
   generatedMessage: false,
   code: 'ERR_ASSERTION',
   actual: false,
   expected: true,
   operator: '=='
   }
   ```

   具体的报错信息可能因为ThingsBoard所使用nodejs版本不同导致错误信息略有不同。

   该报错信息没有什么帮助性信息，仔细观察maven日志，发现是执行`yarn install`失败。于是手动切换到`msa/js-executor/`目录手动执行`yarn install`，输出结果与上面一致，解决过程暂时停滞。一番搜索后，没有发现对应3.3.0版本的解决方案，故决定继续自己寻找原因。

   回到`msa/js-executor`目录，打开package.json打包配置，发现打包时执行的并非常规的npm指令，而是使用`pkg`指令进行替代

   ```
   {
     "name": "thingsboard-js-executor",
     "private": true,
     "version": "3.3.0",
     "description": "ThingsBoard JavaScript Executor Microservice",
     "main": "server.js",
     "bin": "server.js",
     "scripts": {
       "install": "pkg -t node12-linux-x64,node12-win-x64 --out-path ./target . && node install.js",
       "test": "echo \"Error: no test specified\" && exit 1",
       "start": "nodemon server.js",
       "start-prod": "NODE_ENV=production nodemon server.js"
     }
   }
   ```

   手动复制install脚本对应的指令`pkg -t node12-linux-x64,node12-win-x64 --out-path ./target . && node install.js`到命令行执行，ok，错误信息终于不同了，提示找不到pkg指令。

   ```
   xml-zl:js-executor xml_zl$ pkg -t node12-linux-x64,node12-win-x64 --out-path ./target . && node install.js
   bash: pkg: command not found
   ```

   可以理解，因为我们并未安装该库，鉴于此nodejs服务使用yarn来管理依赖，所以我们手动安装`pkg`

   ```
   yarn add pkg
   ```

   如果没有网络问题，在pkg库安装后，会自动执行`pkg -t node12-linux-x64,node12-win-x64 --out-path ./target . && node install.js`输出如下所示：

   ```
   warning "pkg" is already in "devDependencies". Please remove existing entry first before adding it to "dependencies".
   success Saved 1 new dependency.
   info Direct dependencies
   └─ pkg@5.3.0
   info All dependencies
   └─ pkg@5.3.0
   $ pkg -t node12-linux-x64,node12-win-x64 --out-path ./target . && node install.js
   > pkg@5.3.0
   > Fetching base Node.js binaries to PKG_CACHE_PATH
     fetched-v12.22.1-linux-x64          [=                   ] 6%
   ```

   重点在错误信息的最后一行，由于众所周知的网络问题，绝大多数人在没有代理的情况下都无法下载下来这个包，这样的包总共有三个，分别名为`fetched-v12.22.1-linux-x64`/`fetched-v12.22.1-macos-x64`/`fetched-v12.22.1-linux-win64`，对应不同平台的打包程序，所以我们需要手动下载这三个包然后放入对应的`PKG_CACHE_PATH/v3.1`目录，在我的电脑上是`/Users/$你的用户名/.pkg-cache/v3.1`。这三个包在[pgk的GitHub主页](https://github.com/vercel/pkg-fetch/releases)即可下载。

   手动将三个包放入后，再打包，发现问题解决。

3. ### IDE中无法提示无法找到org.thingsboard.server.gen.transport.***Msg

   这是由于找不到的这些类都是执行maven编译后，由protobuf提供的编译插件自动生成在对应模块的target目录下。

   所以需要手动指定生成的代码目录为源代码目录

   以IDEA为例，需要手动在project stucture中指定`common/queue`的`target/generated-sources/protobuf/java`目录为source目录，如下图所示

   <img src="https://github.com/scybZhangliang/thingsboard_guide/blob/master/img/spec_source.png"/>

   如果上述操作后仍无法解决，我们打开`TransportProtos.java`，会发现IDEA会有如下提示

   `File size exceeds configured limit (2560000), code insight features not available`

   这是因为IDEA默认的文件大小为2.5M，但生成的文件超过了这个大小，我们重新配置限制的值即可。

   在IDEA顶部工具栏中依次点击 `Help->Edit Custom Properties`，会打开`idea.properties`文件，如果没有配置过，IDEA会提示自动生成，在打开的文件中加入如下内容：

   ```
   # custom IntelliJ IDEA properties
   
   idea.max.intellisense.filesize=2500000
   ```

   至此，问题解决，IDE中不再提示报错，

   参考链接：[File size exceeds configured limit (2560000), code insight features not available](https://stackoverflow.com/questions/23057988/file-size-exceeds-configured-limit-2560000-code-insight-features-not-availabl)