---
title: webpack2编译过程最佳实践
date: 2017-08-03 11:56:13
tags:
---

背景
构建一个项目的时候往往需要第三方库与插件分离，因为第三方库并不需要经常更新，被浏览器缓存可以提高项目的加载速度

CommonsChunkPlugin

webpack2 支持第三方第三方库与项目源码分别打包，通过CommonsChunkPlugin 可以把共享的模块单独打包出来，通过将公共模块与软件包分开，生成的块文件可以初始加载并存储在缓存中供以后使用。这样浏览器可以从缓存快速提供共享代码，而不是在访问新页面时强制加载更大的包。
commonsChunkPlugin配置第三方模块的代码示例如下：
new webpack.optimize.CommonsChunkPlugin({
  name: "commons",
  filename: "commons.js",
})

但是在开发中并不需要时时打包commons模块，每次修改代码时热起项目都会把commons重新打包，这样没有意义并且会拖慢打包速度，因此最好的结果是commons的打包过程与源码分离，并单独打包

DllPlugin
DllPlugin将commons模块单独打包，并且输出一个dll文件，dll文件作为依赖不会单独存在，他只包含第三发模块，合并的commons通过output.libery选项把function暴露给全局作用域，并生成manifest映射文件

DllReferencePlugin
引用一个打包完成的dll文件，通过manifest把名称映射到dll函数可访问的ID模块

打包过程
开发环境
开发环境下，先由dll打包公共模块，再由dev打包源码，并把dll映射载入，这样每次修改源码，webpack只打包修改的部分，大大节省了编译时间
构建环境
由CommonsChunkPlugin打包公共模块

对比
dll编译过程
{% asset_img PastedGraphic1.jpg PastedGraphic1%}
打包源码过程
{% asset_img PastedGraphic2.jpg PastedGraphic2%}
通过CommonsChunkPlugin打包过程
{% asset_img PastedGraphic3.jpg PastedGraphic3%}
由上可知，在两边的vendor文件同样大小的情况下，DllPlugin每次编译时间是CommonsChunkPlugin 1/3左右，刨除dll单独打包时间影响，再只改动源码的条件下，DllPlugin编译时间约为CommonsChunkPlugin 1/10