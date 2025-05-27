这节我们来实现 git init、add 命令。

创建项目：

```perl
mkdir my-git
cd my-git
npm init -y
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38157a0666784f7d8e78cd9cbc9bd355~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=822&h=680&s=103616&e=png&b=000000)

我们先试下 git init：

```csharp
git init
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea2f9dd0920c4c4eb4dedff65e43a24e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1736&h=794&s=45370&e=webp&b=1c1c1c)

执行后会创建 .git 目录，下面有一堆文件。

我们先引入 commander 来做命令行解析。

```css
npm install --save commander
```

创建 src/index.mjs

```javascript
#!/usr/bin/env node
import { Command } from 'commander';
import { init } from './init.mjs';

const program = new Command();

program
  .name('my-git')
  .description('自己实现的 git')
  .version('0.0.1');

program.command('init')
  .description('初始化 git 仓库')
  .action((str, options) => {
    init();
  });

program.parse();
```

我们用 commander 来解析命令行，然后创建了一个 init 命令。

创建 src/init.mjs

```javascript
export function init() {
    console.log('init');   
}
```

跑一下：

```bash
node ./src/index.mjs

node ./src/index.mjs init
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbb4d570940042778673630b6c963489~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=818&h=424&s=52670&e=png&b=181818)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e6a1cba83788435e8160cc5b5cd6c6f1~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=608&h=100&s=14559&e=png&b=181818)

没啥问题。

然后来实现 init 命令。

我们先跑一下 git init：

```csharp
git init
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c2f68f38fb74eea9675bf48c5c3dae2~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1752&h=690&s=127669&e=png&b=1c1c1c)

可以看到，git 在 .git 目录下写入了一堆文件

上节分析过，git 有 blob、tree、commit 这三种 object，然后会有 branch、HEAD 等 ref 来指向 commit。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d3bc4b49b6a44028b5d5db603536d0e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1708&h=880&s=441017&e=png&b=fdfbfb)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d62aa00b55114d52ad49ee26586701ab~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=394&h=570&s=48505&e=png&b=252526)

可以看到 .git/HEAD 就是记录了当前的分支：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fac6dcc5c3a41f188892ea8c5880a3e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1100&h=478&s=48645&e=png&b=1c1c1c)

init 就是创建保存这些内容的文件、目录。

实现下：

```javascript
import fs from 'node:fs';

export function init() {
    const projectDir = process.cwd();

    const isExist = fs.existsSync(`${projectDir}/.my-git`);

    if(isExist) {
        console.log('Your project has been Initialized.');
        return;
    }

    [
        `${projectDir}/.my-git`,
        `${projectDir}/.my-git/objects`,
        `${projectDir}/.my-git/refs`,
        `${projectDir}/.my-git/refs/heads`
    ].forEach(dir => {
        fs.mkdirSync(dir, {
            recursive: true
        });
    })
    
    fs.writeFileSync(`${projectDir}/.my-git/HEAD`, 'ref: refs/heads/main');

    console.log(`Initialized empty my-git repository in ${projectDir}`);
}
```

首先检查下 .my-git 是否存在，存在说明项目初始化过了。

不存在的话就创建 objects、refs、refs/heads 目录。

之后创建 HEAD 文件来记录当前分支为 main 分支。

跑一下：

```bash
node src/index.mjs init
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b504f1e0d4541fc9cd61d020efd1283~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1700&h=852&s=204506&e=png&b=1b1b1b)

可以看到，执行 init 命令后 .my-git 目录创建成功了。

然后我们来实现 add 命令。

git 有工作区、暂存区、本地仓库、远程仓库这几个概念：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0880dd2daefd4115a5c7765dd277b94d~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1008&h=698&s=256561&e=png&b=fefefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b065421d4bd7450296b331b74212619a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=2016&h=846&s=550561&e=png&b=fdfcfc)

执行 git add 的时候，就是把工作区的内容放到暂存区。

而 git commit 是创建一个新的 commit，存入版本库。

我们来实现下 git add

![2025-03-03 09.24.21.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cf668bbbae34fe78e3575c7c5f3d029~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1834&h=1238&s=565402&e=gif&f=67&b=191919)

跑一下 git add，可以看到创建了一堆 objects，以及多了一个 index 文件。

这些 objects 就是保存的文件内容，然后 index 是暂存区。

我们来实现下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5288a58ba8fd44f0a4a7f56ebe841a0d~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=810&h=1050&s=171103&e=png&b=1f1f1f)

```javascript
import { add } from './add.mjs';
```

```javascript
program.command('add')
  .description('保存本地改动到暂存区')
  .action((str, options) => {
    add();
  });
```

创建 src/add.mjs

```javascript
import { glob } from 'glob';

export async function add() {
    const files = await glob('**', {
        cwd: process.cwd(),
        nodir: true,
        ignore: 'node_modules/**'
    });

    console.log(files);
}
```

我们用 glob 来匹配文件。

安装下：

```css
npm install --save glob
```

跑一下：

```bash
node src/index.mjs add
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57a2e4f321c7407499179e92e5bd4a2d~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=552&h=284&s=33413&e=png&b=181818)

可以看到，匹配到了所有的文件。

当然，排除哪些目录不是固定的，git 里是通过 .gitignore 来配置的。

我们也来实现下：

```javascript
import { glob } from 'glob';
import fs from 'node:fs';

export async function add() {

    let ignore = [];
    
    try {
        const ignoreContent = fs.readFileSync(`${process.cwd()}/.mygitignore`, {encoding: 'utf-8'});

        ignore = ignoreContent.split('
');
    } catch(e) {
    }

    const files = await glob('**', {
        cwd: process.cwd(),
        nodir: true,
        ignore: ignore
    });

    console.log(files);
}
```

读取 .mygitignore 文件的内容，作为 ignore 排除的目录。

创建这个文件：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/513524bc03d04302801703aae06535c7~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1136&h=560&s=79063&e=png&b=1c1c1c)

跑一下：

```bash
node src/index.mjs add
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4ff6aee224b349c9a4a3a9095a8d75e2~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=524&h=286&s=33294&e=png&b=181818)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89bebed59f7c429e9a73e00a74b4802f~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1256&h=798&s=124255&e=png&b=1b1b1b)

排除 node\_modules 目录，排除某个文件，都没问题。

然后来实现写入 objects 的功能。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e2da6e6075047169ba3e44a36618c28~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1204&h=1126&s=169027&e=png&b=1f1f1f)

创建一个 blobs 对象来保存文件信息。

我们读取每个 file 的内容，包括 content、length、type，以及计算出的 SHA1 值。

SHA1 用 cypto 这个包的 api 计算：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7f55917169cb48f89f88df9df677de82~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1438&h=868&s=145735&e=png&b=1f1f1f)

```javascript
import { glob } from 'glob';
import fs from 'node:fs';
import crypto from 'node:crypto';

export async function add() {

    let ignore = [];
    
    try {
        const ignoreContent = fs.readFileSync(`${process.cwd()}/.mygitignore`, {encoding: 'utf-8'});

        ignore = ignoreContent.split('
');
    } catch(e) {
    }

    const files = await glob('**', {
        cwd: process.cwd(),
        nodir: true,
        ignore: ignore
    });

    const blobs = {};

    files.forEach(item => {

        const content = fs.readFileSync(item);

        let metaData = {
            type: 'blob',
            length: content.length,
            content: content
        }

        blobs[item] = {
            metaData,
            SHA1: getSHA1(JSON.stringify(metaData))
        }
    });

    console.log(blobs);
}

function getSHA1(content) {
    return crypto.createHash('sha1').update(content, 'utf8').digest('hex');
}
```

跑一下：

```bash
node src/index.mjs add
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53032a8159fb482387984949a28e64b0~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1126&h=1060&s=160456&e=png&b=181818)

可以看到，所有的 blob 信息都拿到了。

然后我们来写入 index：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b23af51c52d34a088ed397f7e0882974~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=500&h=820&s=70428&e=png&b=181818)

index 里只是保存着当前有那些文件，并不会保存具体内容。

所以我们只把文件路径，还有 SHA1 内容，写入 index。

用 JSON.stringify 格式化成字符串，然后用 zlib.gzipSync 压缩下，写入 index。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d1370ef149d4bd18246dcc4027bc30e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1382&h=900&s=158960&e=png&b=1f1f1f)

```javascript
const blobsWithoutContent = {};

for(let key in blobs) {
    blobsWithoutContent[key] = {
        SHA1: blobs[key].SHA1
    }
}

const indexContent = zlib.gzipSync(JSON.stringify(blobsWithoutContent));
fs.writeFileSync(`${process.cwd()}/.my-git/index`, indexContent);

console.log('Index write succeeded.')
```

跑一下：

```bash
node src/index.mjs add
```

![2025-03-03 10.14.52.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da3f6470647141e49ccb7f71ec8a52d4~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1834&h=1238&s=341842&e=gif&f=33&b=191919)

可以看到，index 创建成功了。

这就是保存文件信息到暂存区中了。

之后我们把这些文件写入 objects 目录。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/766221e9881441c0b56b0a224a3225fb~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=490&h=542&s=46241&e=png&b=181818)

objects 下面保存着文件内容，目录名是 SHA1 前两位，文件名是后面的数字。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65b741cd723140aaa7e4cfe8655f4693~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1584&h=778&s=158099&e=png&b=1f1f1f)

```javascript
for (let item in blobs) {
    const dir = process.cwd() + '/.my-git/objects/' + blobs[item].SHA1.substring(0, 2);
    const filename = dir + '/' + blobs[item].SHA1.substring(2);

    try {
        fs.mkdirSync(dir);
    } catch(e) {
    }

    const content = zlib.gzipSync(JSON.stringify(blobs[item].metaData));

    fs.writeFileSync(filename, content);
}

console.log('Blob write succeeded');
```

我们创建 objects 目录，文件名是 SHA1 前两位，文件名是后面的数字。

然后写入文件内容，同样用 gzip 压缩。

跑一下：

![2025-03-03 10.29.28.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a2814b6b2c3458f9b5385c38e6bc69f~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1840&h=1418&s=651424&e=gif&f=40&b=1a1a1a)

objects 写入成功，index 暂存区里也保存了文件的索引。

这样，git add 就实现了。

## 总结

这节我们实现了 git init、git add 命令。

git init 就是创建 .my-git 目录，以及 objects、refs 等目录以及 HEAD 文件。

git add 是把文件内容写入 objects 目录，然后把索引保存在 index 文件，也就是暂存区。

我们用 commander 来做命令行解析，用 glob 来做文件的匹配和过滤，用 zlib.gzipSync 来压缩文件内容，用 crypto 来计算 SHA1。

实现完 git init、add，下节我们继续来实现 commit 命令。