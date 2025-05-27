前面实现了 git init、add、commit 命令，这节我们继续来实现 cat-file、log 命令。

cat-file 命令并不常用，它是 git 的底层命令。

在学习 git 实现原理的时候我们用过：

cat-file -t 可以查看 object 的类型：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f3524bcb02c48c0b75c59852a9b963c~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1152&h=102&s=19992&e=png&b=1e1e1e)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d95b9f58dd434e6b91353d06d309e83a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1282&h=124&s=22218&e=png&b=1e1e1e)

也就是 blob、tree、commit 这三种对象。

然后 cat-file -p 可以查看对象的内容：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cda5f738236344358663e7a01e273c2f~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1982&h=884&s=107152&e=png&b=1e1e1e)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c336b877d34c489c913dea4f8e356b60~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1406&h=150&s=92435&e=png&b=1f1f1f)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a972d64012a64b459e899839f7263362~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1374&h=272&s=56518&e=png&b=1e1e1e)

我们来实现下这个命令。

首先在 index.mjs 注册这个命令：

```javascript
program.command('cat-file')
  .description('查看对象的内容或类型')
  .argument('<hash>', '对象的 sha1 值')
  .option('-t', '查看对象类型')
  .option('-p', '查看对象内容')
  .action((hash, options) => {
    console.log(hash, options);
  });
```

跑一下：

```javascript
node src/index.mjs cat-file -h
node src/index.mjs cat-file -t 111
node src/index.mjs cat-file -p 111
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5279455e22314fe6b2566f14d221fc7f~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=812&h=426&s=57760&e=png&b=191919)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/377f97864dac4e7fa6d5e1145c2c6e3a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=762&h=210&s=42438&e=png&b=191919)

然后来实现这个命令

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fee2cd0ced8e4957b041c5fc0cfd776f~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=818&h=484&s=106595&e=png&b=1f1f1f)

创建 src/cat-file.mjs

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';

export function catFile(hash, options) {
    if(options.t) {
        const dir = hash.slice(0, 2);
        const filename = hash.slice(2);

        const filePath = `${process.cwd()}/.my-git/objects/${dir}/${filename}`;

        const compressedContent = fs.readFileSync(filePath);
        const content = zlib.gunzipSync(compressedContent);

        const obj = JSON.parse(content);
        console.log(obj);
    } else {

    } 
}
```

按照传入的 hash 来读取对应的 object 文件，解压缩后 parse 拿到对象内容。

跑一下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/831dc31891174699bf655b61732bdb5c~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1378&h=564&s=104163&e=png&b=181818)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/738baa3ede104ed5842b956d8811e35b~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1370&h=294&s=50917&e=png&b=181818)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6cf93882a9ae476a90b1166b52ae6dc8~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1436&h=382&s=103433&e=png&b=181818)

然后 -t 就是展示 type，-p 就是展示 content。

实现下：

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';

export function catFile(hash, options) {
    const dir = hash.slice(0, 2);
    const filename = hash.slice(2);

    const filePath = `${process.cwd()}/.my-git/objects/${dir}/${filename}`;

    const compressedContent = fs.readFileSync(filePath);
    const content = zlib.gunzipSync(compressedContent);

    const obj = JSON.parse(content);

    if(options.t) {
        console.log(obj.type);
    } else {
        switch(obj.type) {
            case 'blob':
                console.log(Buffer.from(obj.content.data).toString('utf-8'))
                break;
            case 'tree':
                console.log(obj.metadata);
                break;
            case 'commit':
                console.log(obj.tree);
                break;
        }
    } 
}
```

分别根据 type 来做不同的处理。

blob 存储的内容是 buffer 的数组，需要用 Buffer.from 创建 buffer 对象，然后再 toString

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77cbfe82aa2f4c3c93eb9f6088eb2ad2~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1378&h=308&s=80552&e=png&b=181818)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6dea8bbabcaa4c0480cc9994c000d02d~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1440&h=540&s=92480&e=png&b=181818)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/860274b149734143a5ec126eb577df68~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1376&h=116&s=34129&e=png&b=191919)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a86164f7b3cd450f83f26494fa5575fd~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1396&h=278&s=93519&e=png&b=181818)

这样，git cat-file 就实现了。

接下来实现 git log 命令。

![2025-03-03 20.19.46.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a153634cde364123bc268a58ef048c7e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1698&h=810&s=2256914&e=gif&f=53&b=171717)

git log 会展示当前分支从 HEAD 开始的串联起来的 commit 链表：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be009f464af642d2aec83b330824ffac~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1790&h=786&s=246316&e=png&b=fefefe)

这个我们在实现 git commit 的时候记录了 parent 的信息，所以 parent -> parent -> parent 这样查找就可以了。

直到 parent 为 null，就找到了第一个 commit。

我们在 index.mjs 添加这个命令：

```javascript
program.command('log')
  .description('查看当前分支的 commit 历史')
  .action(() => {
    log()
  });
```

然后创建 src/log.mjs

```javascript
import fs from 'node:fs';

function getHead() {
    const content = fs.readFileSync(process.cwd() + '/.my-git/HEAD', 'utf-8');

    let refPath = content.split(':')[1].trim();

    let refPathArr = refPath.split('/');
    const curBranch = refPathArr[refPathArr.length - 1];
    return {
        curBranch: curBranch,
        fullPath: `${process.cwd()}/.my-git/${refPath}`
    }
}

export function log() {
    const head = getHead();
    const hash = fs.readFileSync(head.fullPath, 'utf-8');

    let list = [];
    let curHash = hash;

    while(curHash !== null) {
        const dir = curHash.slice(0, 2);
        const filename = curHash.slice(2);

        const filePath = `${process.cwd()}/.my-git/objects/${dir}/${filename}`;

        const compressedContent = fs.readFileSync(filePath);
        const content = zlib.gunzipSync(compressedContent);

        const obj = JSON.parse(content);

        list.push(obj);

        curHash = obj.parent
    }

    console.log(list);
}
```

我们读取 HEAD 文件里的分支，然后去 refs/heads/xxx 下读取 commit 的信息：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c758739811c347efa299620430982ba1~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1168&h=748&s=72114&e=png&b=1d1d1d)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dedce434c8444e14ab093de067f310b8~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1150&h=698&s=75477&e=png&b=1d1d1d)

之后从读取那个 commit 对象，根据它的 parent 一直往上找，直到 parent 为 null。

这就找到了当前分支的所有 commit。

我们在本地改动下内容，然后 add、commit 几次

```bash
node src/index.mjs add
node src/index.mjs commit -m 'bbb'
node src/index.mjs add
node src/index.mjs commit -m 'ccc'
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c4891b126ea44f093dae29c921b0342~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=930&h=634&s=125871&e=png&b=191919)

之后执行 git log

```bash
node src/index.mjs log
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f355821ceb549a7b9391ac39f5e5a55~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1068&h=864&s=128658&e=png&b=181818)

可以看到，三个 commit 的内容都展示出来了。

我们可以继续完善交互逻辑

做成这种：

![2025-03-03 20.19.46.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a153634cde364123bc268a58ef048c7e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1698&h=810&s=2256914&e=gif&f=53&b=171717)

这需要键盘控制。

键盘控制要用到 readline 模块，这些我们前面学过，这里就不实现了。

这样，我们的 git log 命令就完成了。

> 代码上传了[小册仓库](https://github.com/QuarkGluonPlasma/nodejs-course-code/tree/main/my-git "https://github.com/QuarkGluonPlasma/nodejs-course-code/tree/main/my-git")

## 总结

这节我们实现了 git cat-file、git log 命令。

git cat-file 是根据 hash 值去查看对象的类型、内容的命令。

git log 是查看当前分支的所有 commit 的命令。

cat-file 只要读取对应的 objects 文件，parse 下，做下相应处理就可以了。

log 则是要读取 HEAD、refs/heads/xxx 来拿到 commit 信息，然后依次读取 parent 的 commit 的 hash，一直到 null。

当然，我们没有做很多键盘、光标的交互控制，这些可以结合前面学过的内容来完善交互。