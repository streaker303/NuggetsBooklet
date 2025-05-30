Puppeteer 是一个网页的自动化测试工具，它支持写一些 JS 脚本来控制浏览器执行一些行为，可以用来跑测试用例，或者用来做爬虫。

它的脚本类似这样：

```javascript
const puppeteer = require('puppeteer');
const fs = require('fs/promises');

(async () => {
  const browser = await puppeteer.launch({
    headless: false
  });

  const page = await browser.newPage();

  await page.goto('https://baidu.com');

  const $input = await page.$('#kw');
  await $input.type('guangguangguang');

  const $button = await page.$('#su');
  await $button.click();

  await page.waitForSelector('#container');
  const screenshot = await page.screenshot();
  await fs.writeFile('./screenshot.png', screenshot);

  await browser.close();
})();
```

我们 launch 了一个浏览器，打开一个标签页，访问 baidu，在输入框输入一些内容，然后点击搜索按钮，等页面出现结果就截下图存到本地文件，然后关闭浏览器。

跑起来是这样的：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6cd29bc4c9bf445298b2395ca2ae847c~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=2570&h=1502&s=1650983&e=gif&f=77&b=1f1f1f)

其实跑这种脚本不需要看到界面，所以 puppeteer 默认是 headless 的，也就是无界面的。（上面是我把 headless 给关掉了）

这种脚本写起来还是很简单的，就是按照你操作的步骤一步步写对应的脚本就好了，甚至还有录制你的行为来生成 puppeteer 脚本的工具。

这节我们不讲它的应用，而是来探究下它的实现原理。我们能不能自己实现一个呢？

**puppeteer 是基于 Chrome DevTools Protocol 实现的，会以调试模式跑一个 chromium 的实例，然后通过 WebSocket 连接上它，之后通过 CDP 协议来远程控制。**

**我们写的脚本最终都会转成 CDP 协议来发送给 Chrome 浏览器，这就是它的实现原理。**

接下来我们尝试自己实现一个简易版 puppeteer 来深入理解它。

要想控制 Chromium，总得先把他下下来吧，所以这一节我们来实现 Chromium 的自动下载（**这节不涉及 CDP，大家简单看下就行，重点在下一节**）。

google 有个网站存储了所有版本、所有平台的 chromium，它的 url 是这样的：

mac 的 url：

[storage.googleapis.com/chromium-br…](https://storage.googleapis.com/chromium-browser-snapshots/Mac/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-mac.zip "https://storage.googleapis.com/chromium-browser-snapshots/Mac/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-mac.zip")

linux 的 url：

[storage.googleapis.com/chromium-br…](https://storage.googleapis.com/chromium-browser-snapshots/Linux_x64/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-linux.zip "https://storage.googleapis.com/chromium-browser-snapshots/Linux_x64/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-linux.zip")

win32 的 url：

[storage.googleapis.com/chromium-br…](https://storage.googleapis.com/chromium-browser-snapshots/Win/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-win32.zip "https://storage.googleapis.com/chromium-browser-snapshots/Win/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-win32.zip")

win64 的 url：

[storage.googleapis.com/chromium-br…](https://storage.googleapis.com/chromium-browser-snapshots/Win_x64/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-win32.zip "https://storage.googleapis.com/chromium-browser-snapshots/Win_x64/%E7%89%88%E6%9C%AC%E5%8F%B7/chrome-win32.zip")

你可以把 url 换成具体的版本号试试，比如 468266、546920

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c05ff1e9f0574a4b8e6d5443eb6e6493~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1910&h=702&s=362037&e=gif&f=51&b=fafafa)

所有的版本号可以在国内的一个镜像网站看到：

[registry.npmmirror.com/binary.html…](https://registry.npmmirror.com/binary.html?path=chromium-browser-snapshots/ "https://registry.npmmirror.com/binary.html?path=chromium-browser-snapshots/")

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15f81efbeb7942119eb9a7d38265bf29~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1530&h=792&s=513310&e=png&b=ffffff)

把下载下来的 zip 包解压，这个不就是我们要的 chromium 浏览器么？

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6a5104754f949fdb3982e0e66b84678~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=396&h=188&s=24169&e=png&b=fefefe)

流程是这么个流程，但我们肯定不能手动搞，要做成自动化的。

因为安装 puppeteer 之后是要下载这个 chromium 的，不能让开发者手动去下吧。

所以接下来我们就把这个流程给自动化了。

创建个项目：

```bash
mkdir mini-puppeteer
cd mini-puppeteer
npm init -y
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80d222e412b245eaa9e6fecb7152351c~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=854&h=680&s=85865&e=png&b=010101)

我们一步步来，首先是下载 chromium 到本地的一个目录：

创建 src/lib/Downloader.mjs

```javascript
import os from 'node:os';
import path from 'node:path';
import util from 'node:util';
import fs from 'node:fs';

const CHROMIUM_PATH = path.join(import.meta.dirname, '..', '..', '.local-chromium');

const downloadURLs = {
    linux: 'https://storage.googleapis.com/chromium-browser-snapshots/Linux_x64/%d/chrome-linux.zip',
    darwin: 'https://storage.googleapis.com/chromium-browser-snapshots/Mac/%d/chrome-mac.zip',
    win32: 'https://storage.googleapis.com/chromium-browser-snapshots/Win/%d/chrome-win32.zip',
    win64: 'https://storage.googleapis.com/chromium-browser-snapshots/Win_x64/%d/chrome-win32.zip',
};

async function downloadChromium(revision, progressCallback) {
    let url = null;

    const platform = os.platform();
    if (platform === 'darwin')
        url = downloadURLs.darwin;
    else if (platform === 'linux')
        url = downloadURLs.linux;
    else if (platform === 'win32')
        url = os.arch() === 'x64' ? downloadURLs.win64 : downloadURLs.win32;

    console.assert(url, `Unsupported platform: ${platform}`);

    url = util.format(url, revision);
    
}
```

首先，下载到的本地目录是 .local-chromium，我们根据输入的版本号，以及从 os.platform() 拿到的系统信息来确定下载的 url。

有两个 node 的 api 要解释下：

console.assert 就是第一个参数的值为 false 的时候，才输出第二个参数的信息：

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed3a9225a7b4403ebae298b653cf8b0a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=862&h=214&s=42600&e=png&b=1f1f1f)

util.format 是格式化字符串用的，有一些占位符，%d 是数字、%s 是字符串、%j 是 JSON 等：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c10c3cb5c644eaa838726be0966ba24~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1362&h=136&s=77680&e=png&b=1e1e1e)

到了这里，就拿到了下载 chromium 的 url。

那么接下来就是用 https 访问这个 url，下载到本地的目录了。

我们接下来实现下载到本地的功能：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6a4ec7d1e7414669a482f1d37c2470a5~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1022&h=694&s=190323&e=png&b=1f1f1f)

```javascript
import https from 'node:https';

function downloadFile(url, destinationPath, progressCallback) {
    let resolve , reject;
    const promise = new Promise((x, y) => { resolve = x; reject = y; });

    const request = https.get(url, response => {
        if (response.statusCode !== 200) {
            const error = new Error(`Download failed: server returned code ${response.statusCode}. URL: ${url}`);
            response.resume();
            reject(error);
            return;
        }

        const file = fs.createWriteStream(destinationPath);

        file.on('finish', () => resolve ());
        file.on('error', error => reject(error));

        response.pipe(file);
    });
    request.on('error', error => reject(error));
    return promise;
}
```

因为这个下载过程是异步的，我们希望返回一个 promise。

很多人写返回 promsie 的方法都是这么写：

```javascript
function func() {
    return new Promise((resolve, reject) => {

        // resolve();

        // reject()
    });
}
```

其实也可以这么写：

```javascript
function func() {
    let resolve , reject;
    const promise = new Promise((x, y) => { resolve = x; reject = y; });

    // resolve();
    // reject();

    return promise;
}
```

中间的部分就是用 https.get 下载 url 的数据了，不过这个回调函数的 response 参数是一个流。

为什么呢？

因为如果数据很多，需要等好久才能传完，那要等全部传完再处理么？

不用，可以每传一部分就处理一部分。这就是流的思想。

基本所有语言处理网络和文件 IO 的 api 都是基于流的，这个前面 stream 章节讲过。

我们创建了一个写入流，写入到本地的文件的，然后把响应流 pipe 到文件流，也就是直接写入到文件里了：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd34d83ca6974cb4896525b85a83e266~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=948&h=580&s=107995&e=png&b=1f1f1f)

失败的时候，流中的数据就不需要了，所以要调用 response.resume() 来消费掉。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37a9c401b71d41c3a5da85ceeb52575e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=938&h=622&s=112139&e=png&b=1f1f1f)

这样就实现了下载功能。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2f64c16fe864412853fb6388bb26034~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=620&h=176&s=22747&e=png&b=262627)

待会再跑，我们先来实现第二步，解压缩：

这个自己处理就比较麻烦了，直接用第三方的包就行，比如 extract-zip：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/229b6bea4a784b369a1bfe06e1635e13~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1050&h=340&s=100304&e=png&b=1f1f1f)

```javascript
import extract from 'extract-zip';
```

```javascript
function extractZip(zipPath, folderPath) {
    return new Promise(resolve  => extract(zipPath, {dir: folderPath}, resolve ));
}
```

它会解压 zip 文件到目标目录。

安装下：

```css
npm install --save extract-zip
```

在处理完下载的 url 之后，调用下这两步：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eff2b8f8806e49b59c5d6f54116346d2~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1190&h=870&s=188501&e=png&b=1f1f1f)

```javascript
const zipPath = path.join(CHROMIUM_PATH, `download-${revision}.zip`);
const folderPath = path.join(CHROMIUM_PATH, revision);

if (fs.existsSync(folderPath)) {
    return;
}

try {
    if (!fs.existsSync(CHROMIUM_PATH)) {
        fs.mkdirSync(CHROMIUM_PATH);
    }

    await downloadFile(url, zipPath, progressCallback);
    await extractZip(zipPath, folderPath);
} catch(e) {}
```

首先确定 zip 包的路径和解压到的目录的路径，如果目录已经存在了，那就不下载了。

否则调用刚才实现的两个方法来下载 zip 和解压缩。

chromium 下载还是比较慢的，我们给它加个进度条：

也就是给 response 流的 data 事件加个回调，把从 content-length 拿到的数据的总大小，还有当前 chunk 的数据大小传过去：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b602b6a23044bb291eaa581824f36a9~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1044&h=612&s=192040&e=png&b=1f1f1f)

```javascript
const totalBytes = parseInt(response.headers['content-length'], 10);
if (progressCallback)
    response.on('data', onData.bind(null, totalBytes));
```

```javascript
function onData(totalBytes, chunk) {
    progressCallback(totalBytes, chunk.length);
}
```

导出 downloadChromium 和拿到下载后的 chrome 的可执行文件路径的方法：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71888548921140ffbec4a674a64dc113~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1754&h=776&s=201789&e=png&b=1f1f1f)

```javascript
function executablePath(revision) {
    const platform = os.platform();
    if (platform === 'darwin')
        return path.join(CHROMIUM_PATH, revision, 'chrome-mac', 'Chromium.app', 'Contents', 'MacOS', 'Chromium');
    if (platform === 'linux')
        return path.join(CHROMIUM_PATH, revision, 'chrome-linux', 'chrome');
    if (platform === 'win32')
        return path.join(CHROMIUM_PATH, revision, 'chrome-win32', 'chrome.exe');
    throw new Error(`Unsupported platform: ${platform}`);
}

export {
    executablePath,
    downloadChromium
}
```

创建 src/install.mjs

```javascript
import { downloadChromium } from './lib/Downloader.mjs';
import fs from 'node:fs';
import ProgressBar from 'progress';
import path from 'node:path';

const pkgJsonPath = path.join(import.meta.dirname, '..', 'package.json');
const pkg = JSON.parse(fs.readFileSync(pkgJsonPath, 'utf-8'));
const revision = pkg.puppeteer.chromium_revision;

downloadChromium(revision, onProgress)
    .catch(error => {
        console.error('Download failed: ' + error.message);
    });

let progressBar = null;
function onProgress(bytesTotal, delta) {
    if (!progressBar) {
        progressBar = new ProgressBar(`Downloading Chromium - ${toMegabytes(bytesTotal)} [:bar] :percent :etas `, {
            complete: '\u2588',
            incomplete: '\u2591',
            width: 20,
            total: bytesTotal,
        });
    }
    progressBar.tick(delta);
}

function toMegabytes(bytes) {
    const mb = bytes / 1024 / 1024;
    return (Math.round(mb * 10) / 10) + ' Mb';
}
```

Downloader 就是我们刚刚实现的下载解压的逻辑，revision 是版本号，这个在 package.json 里配置。

progress 是一个第三方的控制台进度条，传入宽度、总大小和显示的字符，每次调用 tick 更新下长度就可以了。

安装下用到的包：

```scss
npm install --save progress
```

在 package.json 声明 chromium 版本：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dcc1f53ba8d74eb9b2f362cc716101cf~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=736&h=420&s=53256&e=png&b=1f1f1f)

```javascript
"puppeteer": {
    "chromium_revision": "970501"
},
```

我们来整体试一下：

```bash
node ./src/install.mjs
```

![2024-12-24 16.41.41.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/177c9df5396d4983bb240f4db0c50945~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1448&h=390&s=90401&e=gif&f=68&b=181818)

下载、解压、进度条都没问题，下载下来的 chromium 也能正常跑起来：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4d7503aad8fb4bb9a1b02689a366e2fc~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=568&h=654&s=56834&e=png&b=191919)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9b6832e7f1c45d18b7ca1864ee24597~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1982&h=606&s=177150&e=png&b=fbfafa)

至此，我们就实现了 chromium 的自动下载，只要在 package.json 里配一个版本号，就能自动下载。

当然，现在还不算完全自动，还要手动执行 node install.js

可以把它配在 postinstall 的 npm scripts 里，安装完这个 mini-puppeteer 的依赖之后触发下载：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0a1fa4db44024a92ba3cb3c6d04a8903~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1010&h=210&s=26329&e=png&b=1e1e1e)

> 代码上传了[小册仓库](https://github.com/QuarkGluonPlasma/nodejs-course-code/tree/main/mini-puppeteer "https://github.com/QuarkGluonPlasma/nodejs-course-code/tree/main/mini-puppeteer")

## 总结

puppeteer 是一个基于 CDP 实现的网页自动化测试工具，可以用来跑测试用例，也可以用来做爬虫等。

我们会从 0 实现一个 mini puppeteer。

这节我们实现了自动下载 chromium：

chromium 所有平台和版本的 zip 包都在 google 的一个网站上存着，通过 os 模块拿到系统信息，再根据传入的版本号就能确定 url。

确定了 url 之后通过 https 模块就可以下载，通过流的方式写入本地文件，并且在每次有 data 的时候更新下进度条。

最后通过第三方的 extract-zip 包实现了解压缩。

并且把这个脚本配到了 postinstall 的 npm scripts 里，只要安装完依赖就会自动下载。

下载 Chromium 只是第一步，下一节我们把 Chromium 跑起来实现远程控制。