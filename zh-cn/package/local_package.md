
### 默认打包格式

2.5.5 版本之后，我们提供了一种新的本地包打包方案，将会更加无缝的对接 `add_requires` 和 `add_packages`。

我们执行 `xmake package` 命令就能够生成默认的新版打包格式。

```console
$ xmake package
package(foo): build/packages/f/foo generated
```

它将会产生 `build/packages/f/foo/xmake.lua` 文件，内容如下：

```lua
package("foo")
    set_description("The foo package")
    set_license("Apache-2.0")
    add_deps("add", "sub")

    on_load(function (package)
        package:set("installdir", path.join(os.scriptdir(), package:plat(), package:arch(), package:mode()))
    end)

    on_fetch(function (package)
        local result = {}
        result.links = "foo"
        result.linkdirs = package:installdir("lib")
        result.includedirs = package:installdir("include")
        return result
    end)
```

其实就是采用 `package()` 来定义描述本地包，就跟远程包一样。

而生成的目录结构如下：

```console
$ tree build/packages/f/foo/
build/packages/f/foo/
├── macosx
│   └── x86_64
│       └── release
│           ├── include
│           │   └── foo.h
│           └── lib
│               └── libfoo.a
└── xmake.lua
```

我们也能够使用 `add_requires`/`add_repositories` 接口来无缝集成这个包。

```lua
add_rules("mode.debug", "mode.release")

add_repositories("local-repo build")
add_requires("foo")

target("bar")
    set_kind("binary")
    add_files("src/*.cpp")
    add_packages("foo")
```

其中，add_repositories 配置指定本地包的仓库根目录，然后就可以通过 `add_requires` 来引用这个包了。

另外，生成的本地包，还有一个特性，就是支持 `target/add_deps`，会自动关联多个包的依赖关系，集成时候，也会自动对接所有依赖链接。

这里有完整的[测试例子](https://github.com/xmake-io/xmake/blob/dev/tests/actions/package/localpkg/test.lua)。

```console
"/usr/bin/xcrun -sdk macosx clang++" -o build/macosx/x86_64/release/bar build/.objs/bar/macosx/x86_64/release/src/main.cpp.o -arch x86_64 -mmacosx-version-min=10.15 -isysroot
/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX11.0.sdk -stdlib=libc++
 -L/Users/ruki/projects/personal/xmake/tests/actions/package/localpkg/bar/build/packages/f/foo/macosx/x86_64/release/lib
 -L/Users/ruki/projects/personal/xmake/tests/actions/package/localpkg/bar/build/packages/s/sub/macosx/x86_64/release/lib
 -L/Users/ruki/projects/personal/xmake/tests/actions/package/localpkg/bar/build/packages/a/add/macosx/x86_64/release/lib
 -Wl,-x -lfoo -lsub -ladd -lz
```

### 生成远程包

出了本地包格式，`xmake package` 现在也支持生成远程包，便于用户将他们快速提交到远程仓库。

我们只需要在打包时候，修改包格式。

```console
$ xmake package -f remote
```

他也会产生 packages/f/foo/xmake.lua 文件。

```lua
package("foo")
    set_description("The foo package")
    set_license("Apache-2.0")
    add_deps("add", "sub")

    add_urls("https://github.com/myrepo/foo.git")
    add_versions("1.0", "<shasum256 or gitcommit>")

    on_install(function (package)
        local configs = {}
        if package:config("shared") then
            configs.kind = "shared"
        end
        import("package.tools.xmake").install(package, configs)
    end)

    on_test(function (package)
        -- TODO check includes and interfaces
        -- assert(package:has_cfuncs("foo", {includes = "foo.h"})
    end)
```

包定义配置相比本地包，多了实际的安装逻辑，以及 urls 和 versions 的设置，

我们也能够通过附加参数，去修改 urls，versions 等配置值，例如：

```console
$ xmake package -f remote --url=https://xxxx/xxx.tar.gz --shasum=xxxxx --homepage=xxxxx`
```

xmake 也会从 target 的 `set_license` 和 `set_version` 等配置中读取相关配置信息。

### 从 CMake 中查找包

现在 cmake 已经是事实上的标准，所以 CMake 提供的 find_package 已经可以查找大量的库和模块，我们完全复用 cmake 的这部分生态来扩充 xmake 对包的集成。

我们可以通过 `find_package("cmake::xxx")` 去借助 cmake 来找一些包，xmake 会自动生成一个 cmake 脚本来调用 cmake 的 find_package 去查找一些包，获取里面包信息。

例如：

```console
$ xmake l find_package cmake::ZLIB
{
  links = {
    "z"
  },
  includedirs = {
    "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.
15.sdk/usr/include"
  },
  linkdirs = {
    "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.
15.sdk/usr/lib"
  }
}
$ xmake l find_package cmake::LibXml2
{
  links = {
    "xml2"
  },
  includedirs = {
    "/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk/usr/include/libxml2"
  },
  linkdirs = {
    "/usr/lib"
  }
}
```

#### 指定版本

```lua
find_package("cmake::OpenCV", {required_version = "4.1.1"})
```

#### 指定组件

```lua
find_package("cmake::Boost", {components = {"regex", "system"}})
```

#### 预设开关

```lua
find_package("cmake::Boost", {components = {"regex", "system"}, presets = {Boost_USE_STATIC_LIB = true}})
set(Boost_USE_STATIC_LIB ON) -- will be used in FindBoost.cmake
find_package(Boost REQUIRED COMPONENTS regex system)
```

#### 设置环境变量

```lua
find_package("cmake::OpenCV", {envs = {CMAKE_PREFIX_PATH = "xxx"}})
```

#### 指定自定义 FindFoo.cmake 模块脚本目录

mydir/cmake_modules/FindFoo.cmake

```lua
find_package("cmake::Foo", {moduledirs = "mydir/cmake_modules"})
```

#### 包依赖集成

```lua
package("xxx")
    on_fetch(function (package, opt)
         return package:find_package("cmake::xxx", opt)
    end)
package_end()

add_requires("xxx")
```

#### 包依赖集成（可选组件）

```lua
package("boost")
    add_configs("regex",   { description = "Enable regex.", default = false, type = "boolean"})
    on_fetch(function (package, opt)
         opt.components = {}
         if package:config("regex") then
             table.insert(opt.components, "regex")
         end
         return package:find_package("cmake::Boost", opt)
    end)
package_end()

add_requires("boost", {configs = {regex = true}})
```

相关 issues: [#1632](https://github.com/xmake-io/xmake/issues/1632)
