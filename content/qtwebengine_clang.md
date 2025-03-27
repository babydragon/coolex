+++
title = "将qtwebengine换成使用clang编译"
date = "2022-05-07"
[taxonomies]
tags = ["gentoo"]
categories = ["Linux"]
+++

Gentoo系统每次升级最麻烦的应该就是`dev-qt/qtwebengine`这个包了，里面包含了整个Chromium的内核，导致每次编译的时候都要很长时间。
最近一次升级（5.15.3_p20220406）还直接被OOM Kill了。之前跟踪了[Gentoo的这个bug](https://bugs.gentoo.org/669082),
但是从维护者的回复来看，目前是没办法提供一个bin包直接安装，还是只能从源码进行编译。

在刚提高的那个bug中，有人提到了开启`jumbo-build`和使用clang能够大大增加编译速度，因此也来尝试下。

第一步，是创建一个将编译环境改成clang的env配置（位于`/etc/portage/env`目录下）：

```
# file: /etc/portage/env/clang
LDFLAGS="${LDFLAGS} -fuse-ld=lld -rtlib=compiler-rt -unwindlib=libunwind -Wl,--as-needed"

# Hardening
_HARDENING_FLAGS="-fPIC -fstack-protector-strong -D_FORTIFY_SOURCE=2"
CFLAGS="${CFLAGS} ${_HARDENING_FLAGS}"
CXXFLAGS="${CXXFLAGS} ${_HARDENING_FLAGS}"
LDFLAGS="${LDFLAGS} -Wl,-z,relro,-z,now -pie"

CC="clang"
CXX="clang++"
```

第二步，强制制定qtwebengine使用这个配置，既在`/etc/portage/package.env`里面增加：
```
dev-qt/qtwebengine clang
```

这样在emerge qtwebengine的时候，就会使用clang了。

然后，编译的过程中还遇到了这两个问题：

1： clang-14: error: invalid linker name in argument '-fuse-ld=lld'
这是因为之前都用的gcc,所以没有安装clang使用的lld,需要手工安装下：`emerge sys-devel/lld`

2: unknown argument: '-mpreferred-stack-boundary=5'
还有好几个类似的错误，都是参数里面的5变成了其他数字。参考Gentoo的bugzilla,需要在编译的时候增加环境变量：
`EXTRA_GN="use_lld=true is_clang=true clang_use_chrome_plugins=false"`

经过这两步修正，就可以正常编译qtwebengine了，不管时间是否缩短了，起码没有再被OOM kill了。。。
