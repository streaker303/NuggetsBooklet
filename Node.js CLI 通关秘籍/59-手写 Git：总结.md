前面几节我们按照 git 的原理基于 node 实现了部分 git 的功能。

git 有 3 个 object 以及两个 ref。

3 个对象是：

* blob： 存储文件内容
* tree： 存储目录结构和文件名，指向 blob 和 tree
* commit：存储版本信息，指向不同版本的入口 tree

2 个 ref 是：

* branch：指向某个 commit
* tag：指向某个 commit

暂存区放在 .git/index 文件里，内容其实也是个 tree 对象的内容。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d3bc4b49b6a44028b5d5db603536d0e~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.image#?w=1708&h=880&s=441017&e=png&b=fdfbfb)

这些内容都在 .git 目录下，以文件的形式存储。

比如 .git/objects 下保存的是对象的内容：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ade45fd995ca46b39bd2eb485db1de11~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=474&h=488&s=34652&e=png&b=191919)

三种对象 blob、tree、commit 都保存在这里，内容的 hash 作为目录和文件名。

而 ref 保存在 refs 目录下：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f6b12bbb0968494e816c7126a26a37cb~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1198&h=566&s=63987&e=png&b=1e1e1e)

refs/heads 保存着分支的引用，refs/tags 保存着标签的引用。

其实就是保存某个 commit 的 hash。

然后 HEAD 指针记录着当前的分支：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/301ba8fb85634e93af7caa0ae627a436~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=992&h=750&s=67480&e=png&b=1c1c1c)

index 里则保存着暂存区的内容。

这些就是 git 的存储结构。

git pull 底层就是从远程仓库把这些文件下载下来：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0880dd2daefd4115a5c7765dd277b94d~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1008&h=698&s=256561&e=png&b=fefefe)

也可以把这些文件保存后 push 到远程仓库。

本地想操作这些 object、ref，就是通过 git add、commit 等命令：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b065421d4bd7450296b331b74212619a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=2016&h=846&s=550561&e=png&b=fdfcfc)

我们分别实现了 git init、add、commit、log、cat-file 等命令。

git init 就是创建这个 .git 目录，用来存放 object、ref

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6209f38761d6482e8a7f1b3b321a9bb1~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1700&h=852&s=185094&e=png&b=1b1b1b)

git add 会把工作区除了 gitignore 里的文件之外的文件，分别用 blob、tree 对象来保存下来：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e91e0537d0954155a125b87cfbf8c8e0~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1860&h=1208&s=404393&e=png&b=fcfbfa)

最后 commit 的时候，会把 index 暂存区也创建一个 tree 对象，这样一层层 tree 对象就可以保存整个文件树的结构。

我们实现的时候做了简化，没用用 tree 对象保存目录，只保存了文件内容的 blob，而是在 index 里保存了所有层级的文件：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd35fd4454964b9684055ea8427b4fc5~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1436&h=382&s=95601&e=png&b=181818)

这样效果也是一样的。

git add 的时候，会把文件写入 objects 目录，并且在 index 里创建索引：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1957ed10a064444aa0b79d863765205a~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=500&h=820&s=63770&e=png&b=181818)

之后 git commit 会创建 tree 对象保存 index 暂存区的内容，然后创建一个 commit 对象来指向这个 tree。

然后把这个 commit 的 hash 写入 refs/heads/xxx 也就是让分支指向这个 commit：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/809220dfa63747c588118f0600bc8d8b~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1198&h=566&s=58662&e=png&b=1e1e1e)

commit 创建的时候，是记录了它的 parent commit 的：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e5d60a42380487ca0edc1646014da19~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=918&h=800&s=186253&e=png&b=1f1f1f)

就是通过 HEAD 拿到当前分支，然后去 refs/heads/xxx 来拿到最新 commit 的 hash，让这个 commit 指向之前的 commit。

最后，我们实现了 git cat-file 和 git log 命令：

cat-file 就是读取 objects 文件，解析它的类型和内容：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/860274b149734143a5ec126eb577df68~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1376&h=116&s=34129&e=png&b=191919)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a86164f7b3cd450f83f26494fa5575fd~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1396&h=278&s=93519&e=png&b=181818)

git log 则是从 HEAD 拿到当前分支，从 refs/heads/xxx 下拿到最新 commit。

解析 commit 和 commit.parent 一直往上找，知道 parent 为 null

这样就拿到了当前分支的所有 commit 信息：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f355821ceb549a7b9391ac39f5e5a55~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=1068&h=864&s=128658&e=png&b=181818)

当然，我们没有做交互上的优化，可以结合之前的知识来做光标、颜色、键盘控制等功能。

你还可以进一步扩展功能：

比如 git branch 的 list 是不是就是 refs/heads 下的目录列出来：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f71ede963344ce1bb6b393ce9f5c26b~tplv-k3u1fbpfcp-jj-mark:1600:0:0:0:q75.jpg#?w=312&h=676&s=41725&e=png&b=191919)

git 分支切换是不是就是修改 HEAD 文件的内容，指向其他分支。

类似的 git 命令的实现原理也是围绕这些来的。

做这个实战，其实也是用到不少 node 的知识的。

我们用 commander 来做的命令行解析，发到 npm 之后，就可以安装来用 package.json 的 bin 注册的命令了。

用 node:zlib 模块做的 gzip 压缩和解压缩。

用 node:crypto 模块做的 sha1 的计算。

用 Buffer 来做的二进制数据的解析。

用 glob 做了文件匹配和过滤。

过程中还涉及到了 fs 的大量文件读写。

做完这个实战项目，一个是对 git 的实现原理就有了清晰的认识，另一个是对 node 如何开发一些命令行工具也就更熟练了。