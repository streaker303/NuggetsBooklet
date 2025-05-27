上节实现了 git int、git add，这节我们来实现 git commit 命令。

git add 是把文件内容保存到暂存区：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b065421d4bd7450296b331b74212619a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=2016&h=846&s=550561&e=png&b=fdfcfc)

而 git commit 则是把这些内容保存到版本库。

上节我们实现了暂存区的功能：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7215c9a9b07f4302976c58c3472e1120~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=502&h=714&s=58043&e=png&b=181818)

这节继续来实现 commit 命令：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7eb2723219f4a24955f2a508f2a1138~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=782&h=664&s=106340&e=png&b=1f1f1f)

```javascript
program.command('commit')
  .description('提交改动到版本库')
  .action((str, options) => {
    commit();
  });
```

创建 src/commit.mjs

```javascript
import fs from 'node:fs';

export function commit() {
    console.log('commit');
}

```

跑一下:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c783053d56f94da8a3c4ab368c94c8e3~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=664&h=110&s=18086&e=png&b=181818)

git add 是把文件写入了 objects，创建 blob 对象，并且创建了 index 暂存区来保存文件的索引。

而 git commit 是创建一个 tree 对象来保存 index 中的索引内容，并且创建一个 commit 对象来引用这个 tree 对象。

这些原理我们在前面三大 object 那节讲过：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c733d73d44c4615b40bc6d090b3c825~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1072&h=1028&s=233270&e=png&b=fdfdfd)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c15513aa1f14e5c8cfb65f44a10b284~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1072&h=912&s=180354&e=png&b=fcfcfc)

所以，我们实现 git commit 就是把暂存区的内容创建 tree 对象，然后创建一个 commit 对象，和之前的 commit 连接起来。

在 git 的实现里，tree 对象用来保存目录信息：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a11d5c82b0f34a0983fd05a684ba0b59~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=2354&h=992&s=417414&e=png&b=1f1f1f)

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9517cef18734296996560917a569b43~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1860&h=1208&s=563598&e=png&b=fcfbfa)

在 tree 对象里存储每个子目录和文件的名字和 hash。

在 blob 对象里存储文件内容。

tree 对象里通过 hash 指向了对应的 blob 对象。

但我们实现做了简化，上节实现 git add 的时候，我们只创建了 blob 对象，没有创建 tree 对象。

这里我们也同样做下简化：

实际上的实现应该是每一层 tree 对象只包含这一层的目录。

然后下一层 tree 对象继续这样：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9517cef18734296996560917a569b43~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1860&h=1208&s=563598&e=png&b=fcfbfa)

就和文件目录是一样的。

这里我们就简化下，tree 对象里保存所有层级的文件。

我们首先 parse 下 index 文件的内容，拿到暂存区里保存的文件信息：

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';

function parseIndex() {
    try {
        const content = fs.readFileSync(process.cwd() + '/.my-git/index');
        
        const res = zlib.gunzipSync(content);
        return JSON.parse(res);
    } catch(e) {
        return null;
    } 
}

export function commit() {
    const indexContent = parseIndex();
    
    if(!indexContent) {
        console.log('use "my-git add" to track');
    } else {
        console.log(indexContent)
    }
}
```

解析 index 的内容，如果没有的话，就提示先执行 add 命令。

跑一下：

```bash
node src/index.mjs commit
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49e6d5ebe1c348e9925aa552ece92859~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1336&h=306&s=87172&e=png&b=181818)

这就是暂存区里的内容。

然后我们把它生成 tree 对象。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c127355de4c4ac49940e3ea70daead5~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1556&h=1124&s=232910&e=png&b=1f1f1f)

我们创建了 tree 对象，然后拿到它的 SHA1 值作为目录和文件名，对内容做压缩之后写入文件。

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';
import crypto from 'node:crypto';

function parseIndex() {
    try {
        const content = fs.readFileSync(process.cwd() + '/.my-git/index');
        
        const res = zlib.gunzipSync(content);
        return JSON.parse(res);
    } catch(e) {
        return null;
    } 
}

function getSHA1(content) {
    return crypto.createHash('sha1').update(content, 'utf8').digest('hex');
}

export function commit() {
    const indexContent = parseIndex();
    
    if(!indexContent) {
        console.log('use "my-git add" to track');
    } else {        
        const tree = {
            type: 'tree',
            metadata: indexContent
        }

        const contentStr = JSON.stringify(tree);
        const treeSha = getSHA1(contentStr);

        const dir = process.cwd() + '/.my-git/objects/' + treeSha.substring(0, 2);
        const filename = dir + '/' + treeSha.substring(2);
        const content = zlib.gzipSync(contentStr);
    
        try{
            fs.mkdirSync(dir, {
                recursive: true
            });
        } catch(e) {}

        fs.writeFileSync(filename, content);

        console.log('tree object write success');
    }
}
```

跑一下：

```bash
node src/index.mjs commit
```

![2025-03-03 17.12.33.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7dba69f99d51478d880611f9e6c080a8~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.gif#?w=1840&h=1418&s=518044&e=gif&f=36&b=191919)

可以看到 objects 目录下多了一个文件，这里就保存着 tree 对象。

然后我们继续创建 commit 对象：

commit 对象就是保存着描述，以及 tree 的 hash：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71f0a856e8ef4374a610c83cf8981388~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1010&h=730&s=477611&e=png&b=fdfcfc)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d1b59dcc7904017893309509e203d19~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1374&h=272&s=56518&e=png&b=1e1e1e)

首先，我们添加一个 commit 命令的选项：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0d57078a9e841e79a3a80c0ca86d78c~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=964&h=466&s=88842&e=png&b=1f1f1f)

```javascript
program.command('commit')
  .description('提交改动到版本库')
  .option('-m, --message <char>', '描述信息')
  .action((options) => {
    commit(options.message);
  });
```

然后创建 commit 对象的时候添加 desc 信息。

commit 对象里需要保存 parent commit 的 hash。

如何找到 parent 的 commit，也就是当前 HEAD 的 commit 呢？

从这里：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c758739811c347efa299620430982ba1~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1168&h=748&s=72114&e=png&b=1d1d1d)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dedce434c8444e14ab093de067f310b8~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1150&h=698&s=75477&e=png&b=1d1d1d)

首先读取 .git/HEAD 的内容，拿到当前 HEAD 指向哪个分支，然后读取 refs/HEAD/xxx 的内容，拿到这个分支的 commit 的 hash。

我们 init 的时候也创建了这个文件：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c180357909b41319c4ccea98e139add~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=562&h=128&s=13560&e=png&b=212121)

来实现下：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/236e502ceba54505885ebeee0cdf8f14~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1564&h=1326&s=243605&e=png&b=1f1f1f)

创建 commit 对象，然后按照同样的方式压缩内容，存入 objects 目录。

commit 对象里保存着 desc 以及 tree 的 hash。

还有 parent 的 commit 的信息。

getHEAD 就是读取 HEAD 文件的内容，解析出分支名和 heads 文件的路径：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62ed7e2611fc40469c69be39691cdfb5~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1530&h=692&s=142157&e=png&b=1f1f1f)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c180357909b41319c4ccea98e139add~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=562&h=128&s=13560&e=png&b=212121)

最后我们还要把最新的 commit 的内容保存到 refs/heads/xxx 下。

也就是让 HEAD 指向这个最新的 commit。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f47ff93dabdc441db3e5ea45a6e216f8~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1158&h=458&s=72777&e=png&b=1f1f1f)

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';
import crypto from 'node:crypto';

function parseIndex() {
    try {
        const content = fs.readFileSync(process.cwd() + '/.my-git/index');
        
        const res = zlib.gunzipSync(content);
        return JSON.parse(res);
    } catch(e) {
        return null;
    } 
}

function getSHA1(content) {
    return crypto.createHash('sha1').update(content, 'utf8').digest('hex');
}

export function commit(desc) {
    const indexContent = parseIndex();
    
    if(!indexContent) {
        console.log('use "my-git add" to track');
    } else {        
        const tree = {
            type: 'tree',
            metadata: indexContent
        }

        const contentStr = JSON.stringify(tree);
        const treeSha = getSHA1(contentStr);

        const dir = process.cwd() + '/.my-git/objects/' + treeSha.substring(0, 2);
        const filename = dir + '/' + treeSha.substring(2);
        const content = zlib.gzipSync(contentStr);
    
        try{
            fs.mkdirSync(dir, {
                recursive: true
            });
        } catch(e) {}

        fs.writeFileSync(filename, content);

        console.log('tree object write success');

        let parentHash = null;
        try{
            parentHash = fs.readFileSync(getHead().fullPath);
        } catch(e) {}

        const commitObj = {
            type: 'commit',
            tree: treeSha,
            time: Date.now(),
            desc: desc,
            parent: parentHash
        }

        const commitStr = JSON.stringify(commitObj);
        const commitSha = getSHA1(commitStr);

        const commitDir = process.cwd() + '/.my-git/objects/' + commitSha.substring(0, 2);
        const commitObjFilename = dir + '/' + commitSha.substring(2);
        const zipContent = zlib.gzipSync(commitStr);
    
        try{
            fs.mkdirSync(commitDir, {
                recursive: true
            });
        } catch(e) {}

        fs.writeFileSync(commitObjFilename, zipContent);

        console.log('commit object write success');

        fs.writeFileSync(getHead().fullPath, commitSha);

    }
}

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
```

跑一下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e95e984c6a7f493daae90f874aff9048~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1432&h=852&s=183016&e=png&b=1d1d1d)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4daed9ee8c14f3b8d800c2a64380c8e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1548&h=972&s=161194&e=png&b=1c1c1c)

commit 对象创建成功，并且 HEAD 也执行了这个 commit。

文件内容保存到了 .my-git 目录下，目录信息和 commit 信息也保存到了 .my-git 目录下。

这不就是保存成功了么？

如果想查看某个 commit 的内容，只要从中解析出 tree 对象的信息，然后依次读取文件内容就好了。

这样我们就实现了 git commit。

## 总结

这节我们实现了 git commit。

它需要首先把 git add 添加到暂存区的内容创建为一个 tree 对象，也就是索引所有 blob 的对象。

然后创建一个 commit 对象，来保存着这个 tree 对象的 hash。

这个 commit 对象还要保存着当前 HEAD 指向的 commit，从 HEAD 和 refs/heads/xxx 里读取。

这样 commit 之后，当前分支就指向了这个 commit，然后从中读取 tree 的内容，之后分别读取 blob 信息，就可以拿到这个 commit 的所有文件了。