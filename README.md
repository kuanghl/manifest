# directory structure
default.xml
dev/default.xml
projects/default.xml
template/default.xml
tools/default.xml

# repo url
## http://source.android.com/source/git-repo.html
## http://source.android.com/images/git-repo-1.png
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo'

# repo bootstrap
```
curl http://android.git.kernel.org/repo > ~/bin/repo'
chmod a+x ~/bin/repo
export PATH=$~/bin:$PATH

repo init -u $url --mirror
repo sync
```

# 撤销整个工程的本地修改
```
repo forall -c 'git reset --hard HEAD; git clean -df; git rebase --abort'
```

# 删除所有多余的文件
```
repo forall -p -c git clean -xfd
```


# [manifest-format.txt](https://gerrit.googlesource.com/git-repo/+/master/docs/manifest-format.txt)
repo的manifest文件描述了repo client的结构：哪些目录能够被下载，并且可以从哪里下载
基本的manifest的git裸仓库只包含了一个default.xml.
manifest仓库也是通过git进行版本控制，当我们执行repo sync的时候会自动更新manifest仓库。
manifest的xml有以下结构构成：
    <manifest>根标签
    <remote>标签
        在xml中可以有多个<remote>标签，每个<remote>标签代表了一个Git仓库的地址，一个或多个<project>可以使用该地址来下载代码，并且<remote>标签还可以通过review属性来指定Gerrit review server的地址
        看一下<remote>标签都可以有哪些属性
        name:代表 该Git Url的名字，在xml中必须唯一。所有使用该Git Url下载代码的<project>所对应的.git/config中都会默认使用该name作为远端仓库的名称，所以我们在该project下执行git fetch,git remote,git pull等操作都会自动的从该Git Url同步代码。
        alias:对name的别名，alias允许重复。如果我们指定了alias,那么alias会覆盖对应Project下的.git/config中的远端仓库的名称。这样就允许不同的Git Url有相同的名称，方便做统一管理。
        fetch：完整下载Url的前缀，比如git@chico.dong.git,当下载某一个Project的时候，他会和<project>标签中的name属性拼成完整的url,比如git@chico.dong.git/mysample
        pushurl：这个是可选属性，当指定该属性的时候，这个值会和<project>标签中的name属性拼成完整的push url路径，这样当我们使用git push命令的时候，就会使用该url。如果不指定该属性，则pushurl和fetch一样。
        review:这个属性是可选的，用来指定Gerrit review server的完整ulr。当指定这个属性的时候，我们使用repo upload命令上传的代码就会上传到该服务器上。如果不指定该属性的话，repo upload命令将不能使用
        revision:可选属性，用来指定每个Project默认使用的git branch。每个<project>可以有自己的revision属性，会覆盖默认的。
    <default>标签
    最多可以指定一个<default>标签。该标签用来提供一些默认值。看一下它的属性
    remote：标签来用来引用某个<remote>标签的name属性值。当<project>不指定remote属性的时候采用该值
    revieson:当一个<project>不指定revision的时候使用该值
    dest-branch:用来指定Project使用的git branch。<project>不指定dest-branch的时候使用该值。如果不指定该值，默认会使用revision指定的branch.
    upstream:该属性代表了一个git提交节点的sha1值。当使用-c模式去指定同步某一个revision的代码时，通过指定upstrem可以使其只同步到该sha1,避免同步整个git节点。如果<project>没有指定该属性，就会使用这个默认值。
    sync-c:当设置该值为true的时候，同步project的时候就只会同步<revision>指定的branch，而不是所有的branch.
    sync-s:当设置该值为true的时候，会同步该Project的sub-project
    sync-tags:设置该值为false,也是只会同步<revision>指定的分支，否则会同步所有tag。
    clone-depth:该属性代表了在同步Project的时候，需要同步最新的多少个commit,默认是都同步。如果<project>没有指定该属性，则会使用该默认值。
    ``
    <project>标签,每一个<project>代表了一个可以被clone到工作区的仓库。<project>允许嵌套，也就是说允许<project>在嵌套<project>,里面的那个<project>被成为Sub-modules.里面<project>的属性自动继承外面<project>的属性
    name:最重要的一个属性，<project>的name属性会被用来和<remote>标签的<fetch>属性拼成该project最终的url。像下面这样
    ${remote_fetch}/${project_name}.git
    .git是repo自动帮我们添加的。如果这个project有一个parent属性，则该project最终的url会被这样拼凑
    ${remote_fetch}/${project_parent}/${project_name}.git
    path:从该git下载下来的代码在本地的保存路径，相对于repo的根目录而言。如果不指定path属性，则会使用name属性。
    remote含义和上面一样。
    revision:和上面的含义一样，是manifest想要track的git branch的名称。这个属性理论上只能是git branch的名称，但是实际上可以是sha1或者是tag。
    dest-branch：git branch的名称，当我们使用repo upload命令的时候，代码会上传到这个branch.如果不指定的话，就会使用revision.
    groups:该project所属的group,一个project可以属于多个group,多个group用 空格或者逗号 分开。所有的project都属于group "all",每个project还额外属于两个group,一个是"name":name和"path":path,比如<project name="monkeys" path="barrel-of"/>,它自动属于default,name:monkeys,path:barrel-off这三个组。如果我们声明一个Project属于notdefault这个组，repo将不会自动下载这个project.如果这个project有parent属性，那么name组和path组将会把parent属性值作为前缀。
    sync-c:和上面一样，只下载revision指定的branch
        sync-s：自动下载sub-modules，就是嵌套的<project>。默认是不会自动下载sub-modules的
            upstream：和上面的含义一样。当sync-c的时候，只同步到该节点指定的代码，不用同步整个ref。
            clone-depth:和上面的含义一样，用来指定同步最新的几个commit
            force-path这个属性基本用不到，当不设置这个属性的时候，我们创建本地镜像的时候，都会按照name属性来组织，但是当设置这个属性为true的时候，会按照path属性来组织。这个属性只有在同步的时候有--mirror参数的时候才会生效。
    <extend-project>标签，这个标签没见有人用过,具体也不知道有什么效果……
    这个属性最常用在本地的manifest当中，可以对已经存在的project的属性进行修改，而不用更改整个project的定义。它有几个属性:
        path：如果被指定的话，强制project被checkout 到指定的目录。
        revision,groups和上面的含义一样
    <annotation>标签
    一个<project>标签可以有零个或者多个<annotation>标签，每个<annotation>标签声明了一个键值对，这个键值对可以在对该project执行forall命令的时候作为环境变量被使用。
    它有一个 keep属性 ，有true或者false两个值，表明这个<annotation>在执行subcommand的时候是否会被保留。
    <copyfile>标签
    可以作为<project>标签的子标签，每一个<copyfile>标签表明了在repo sync的时候从src把文件拷贝到dest。 src相对于该project来说，dest相对于根目录来说。
    <linkfile>标签
    和<copyfile>标签的作用类似，不过是不进行拷贝，而是进行一个符号链接
    <remove-project>这个属性不知道怎么用，但是从翻译来看，这个属性经常用在内部的manifest中，这样就允许同一个manifest中后来的<project>用不同的source来替换这个project.不明觉厉，完全不懂
    这个属性最长用在local manifest中就可以了，平时遇不到。
    <include>
    用来引入一个其他的manifest,有一个name属性指向被引用的manifest, 路径是相对于mamanifest库的根目录
Local Manifest
Local Manifest可以提供额外的<remote>和<project>.local manifest的存储路径是$TOP_DIR/.repo/local_manifests/*.xml
注意local_manifests文件夹中定义的所有.xml文件,在repo sync的时候都会被优先的下载。也就是说这个文件夹下的所有xml文件所定义的<project>都会被下载。
每个local manifest按照字母顺序被加载，也就是a.xml中的Project先下载，然后下载b.xml中的project.
在.repo下有一个local_manifest.xml文件，也就是.repo/local_manifest.xml,这个文件是最先被加载的，然后才是local_manifests文件夹下的xml文件.repo/local_manifests/*.xml.

