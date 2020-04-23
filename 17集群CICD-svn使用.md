[toc]

# svn使用

参考链接：https://www.kancloud.cn/i281151/svn

## 检出--checkout

用法示例

```bash
svn checkout http://路径(目录或文件的全路径)　[本地目录全路径] --username　用户名
svn checkout svn://路径(目录或文件的全路径)　[本地目录全路径]  --username　用户名
# 使用http方式，co是checkout的简写
svn co http://路径(目录或文件的全路径)　[本地目录全路径]  --username 用户名 --password 密码
# 使用svn方式
svn co svn://路径(目录或文件的全路径)　[本地目录全路径]  --username 用户名 --password 密码
# 检出指定版本就在后面加入-r 版本号，后面细讲
```

**注意：**

- 如果不带--password 参数传输密码的话，会提示输入密码，建议不要用明文的--password 选项。

- 不指定本地目录全路径，则检出到当前目录下。
- checkout导出获得文件后，导出的文件仍处于SVN版本控制中，与版本库保持关联，你可以进行Svn Update或者Svn Commit操作。同时导出文件夹下有一个.svn的隐藏文件夹，存储着一些版本的元数据信息。

**应用场景：**当你需要修改和提交的时候，用checkout，它会在你本地建立一个工作区。

## 导出--export

用法示例，和checkout一样

```bash
# 导出最新版本
svn export http://路径(目录或文件的全路径) --username=${MY_POD_SVN_NAME}
# 导出指定版本，svn的版本号都是r开头，后接数字
svn export -r31  http://路径(目录或文件的全路径) --username=${MY_POD_SVN_NAME}
# 上面的命令会打印每一个文件和目录，
```

**注意：**export导出一个版本的数据，导出的文件脱离SVN版本控制，修改后无法进行Update和Commit操作。导出文件夹下没有.svn目录。

**应用场景：**当你要发布或编译的时候，最好采用export，它不会引入svn的附加文件。

##  取消Add/Delete

取消文件

```bash
svn revert 文件名
```

取消目录

```bash
svn revert --depth=infinity 目录名
```

## 回退版本

### 方法1：用svn merge

1. 先 `svn up`，保证更新到最新的版本，如20；
2. 然后用 `svn log `，查看历史修改，找出要恢复的版本，如10 。如果想要更详细的了解情况，可以使用`svn diff -r 10:20 [文件或目录]`;
3. 回滚到版本号10：`svn merge -r 20:10 [文件或目录]`，注意版本号之间的顺序，这个叫反向合并；
4. 查看当前工作版本中的文件，如test.cpp和版本号10中文件的差别：`svn diff -r 10 test.cpp`， 有差别则手动改之；
5. 若无差别，则提交：`svn ci -m“back to r 10，xxxxx” [文件或目录]`。这时svn库中会生成新的版本，如21。

### 方法2:  用svn up

**前2步如方法1，然后直接 svn up -r 10。当前的工作版本就是版本10了。**

**但是注意，这时svn库中并不会生成新的版本，下次svn up之后，还是会回到当前的版本。**

改动已经被提交（commit）。

用svn merge命令来进行回滚。

### 回滚的操作过程如下

1、保证我们拿到的是最新代码：

```bash
svn update
```

假设最新版本号是28。

2、然后找出要回滚的确切版本号：

```bash
svn log
```

假设根据svn log日志查出要回滚的版本号是25，此处的something可以是文件、目录或整个项目

**如果想要更详细的了解情况，可以使用`svn diff -r 28:25 ""`**

3、回滚到版本号25：

```bash
svn merge -r 28:25 ""
```

为了保险起见，再次确认回滚的结果：

```bash
svn diff ""
```

发现正确无误，提交。

4、提交回滚：

```bash
svn commit -m "Revert revision from r28 to r25,because of ..."
```

提交后版本变成了29。

将以上操作总结为三条如下：

1. svn update，svn log，找到最新版本（latest revision）

2. 找到自己想要回滚的版本号（rollbak revision）

3. 用svn merge来回滚： svn merge -r : something

参数`-r`有效选项:

```
-r [--revision] ARG : ARG (一些命令也接受ARG1:ARG2范围)

版本参数可以是如下之一:

NUMBER 版本号

'{' DATE '}' 在指定时间以后的版本

'HEAD' 版本库中的最新版本

'BASE' 工作副本的基线版本

'COMMITTED' 最后提交或基线之前

'PREV' COMMITTED的前一版本
```

## 常用命令简要说明

- `svn add`

  **用法示例**

  ```bash
  svn add PATH...
  ```

  **描述**

  添加文件、目录或符号链到你的工作拷贝并且预定添加到版本库。它们会在下次提交上传并添加到版本库，如果你在提交之前改变了主意，你可以使用**svn revert**取消预定。

  **别名**：无

  **变化：**工作拷贝

  **是否访问版本库**：否

  **注意：**通常情况下，命令**svn add ***会忽略所有已经在版本控制之下的目录，有时候，你会希望添加所有工作拷贝的未版本化文件，包括那些隐藏在深处的文件，可以使用**svn add**的`--force`递归到版本化的目录下：

  **可用选项**

  ```
  --targets FILENAME
  --non-recursive (-N)
  --quiet (-q)
  --config-dir DIR
  --auto-props
  --no-auto-props
  --force
  ```

- `svn blame`

  **用法示例**

  ```bash
  svn blame TARGET...
  ```

  **描述**

  显示特定文件和URL内嵌的作者和修订版本信息。每一行文本在开头都放了最后修改的作者（用户名）和修订版本号。

  **别名**：praise、annotate、ann

  **是否访问版本库**：是

  **可用选项**

  ```
  --revision (-r) REV
  --username USER
  --password PASS
  --no-auth-cache
  --non-interactive
  --config-dir DIR
  --verbose
  ```

- `svn cat`

  **用法示例**

  ```bash
  svn cat TARGET[@REV]...
  ```

  **描述**

  输出特定文件或URL的内容。列出目录的内容可以使用**svn list**。

  **别名**：无

  **变化：**无

  **是否访问版本库**：是

  **可用选项**

  ```
  --revision (-r) REV
  --username USER
  --password PASS
  --no-auth-cache
  --non-interactive
  --config-dir DIR
  ```

- `svn checkout`

  **用法示例**

  ```bash
  svn checkout URL[@REV]... [PATH]
  ```

  **描述**

  从版本库取出一个工作拷贝，如果省略*`PATH`*，URL的基名称会作为目标，如果给定多个URL，每一个都会检出到PATH的子目录，使用URL基名称的子目录名称。

  **别名**：co

  **变化：**创建一个工作拷贝。

  **是否访问版本库**：是

  **可用选项**

  ```
  --revision (-r) REV
  --quiet (-q)
  --non-recursive (-N)
  --username USER
  --password PASS
  --no-auth-cache
  --non-interactive
  --config-dir DIR
  ```

- `svn cleanup`

  **用法示例**

  ```bash
  svn cleanup [PATH...]
  svn cleanup
  svn cleanup /path/to/working-copy
  ```

  **描述**

  递归清理工作拷贝，删除未完成的操作锁定。如果你得到一个“工作拷贝已锁定”的错误，运行这个命令可以删除无效的锁定，让你的工作拷贝再次回到可用的状态。见[附录B, *故障解决*]。

  如果，因为一些原因，运行外置的区别程序（例如，用户输入或是网络错误）有时候会导致一个**svn update**失败，使用`--diff3-cmd`选项可以完全清除你的外置区别程序所作的合并，你也可以使用`--config-dir`指定任何配置目录，但是你应该不会经常使用这些选项。

  **别名**：无

  **变化：**工作拷贝

  **是否访问版本库**：否

  **可用选项**

  ```
  --diff3-cmd CMD
  --config-dir DIR
  ```

  **注意：svn cleanup**没有输出，如果你没有传递路径，会使用`.`

- `svn commit`

  **用法示例**

  ```bash
  svn commit [PATH...]
  svn commit -m "added howto section."
  ```

  **描述**

  将修改从工作拷贝发送到版本库。如果你没有使用`--file`或`--message`提供一个提交日志信息，**svn**会启动你的编辑器来编写一个提交信息，见[“config”一节]的`editor-cmd`小节。

  如果你开始一个提交并且Subversion启动了你的编辑器来编辑提交信息，你仍可以退出而不会提交你的修改，如果你希望取消你的提交，只需要退出编辑器而不保存你的提交信息，Subversion会提示你是选择取消提交、空信息继续还是重新编辑信息。

  **别名**：ci（“check in”的缩写；不是“checkout”的缩写“co”。）

  **变化：**工作拷贝，版本库

  **是否访问版本库**：是

  **可用选项**

  ```
  --message (-m) TEXT
  --file (-F) FILE
  --quiet (-q)
  --non-recursive (-N)
  --targets FILENAME
  --force-log
  --username USER
  --password PASS
  --no-auth-cache
  --non-interactive
  --encoding ENC
  --config-dir DIR
  ```

- `svn copy`

  **用法示例**

  ```bash
  svn copy SRC DST
  ```

  **描述**

  拷贝工作拷贝的一个文件或目录到版本库。*`SRC`*和*`DST`*既可以是工作拷贝（WC）路径也可以是URL：

  WC -> WC
  拷贝并且预定一个添加的项目（包含历史）。

  WC -> URL
  将WC或URL的拷贝立即提交。

  URL -> WC
  检出URL到WC，并且加入到添加计划。

  URL -> URL
  完全的服务器端拷贝，通常用在分支和标签。

  **注意：**

  你只可以在单个版本库中拷贝文件，Subversion还不支持跨版本库的拷贝。

  **别名**：cp

  **变化：**如果目标是URL则包括版本库。如果目标是WC路径，则是工作拷贝。

  **是否访问版本库**：如果目标是版本库，或者需要查看修订版本号，则会访问版本库。

- `svn delete`

- `svn diff`

- `svn export`

- `svn import`

- `svn list`

- `svn log`

- `svn merge`

- `svn mkdir`

- `svn move`

- `svn revert`

- `svn status`

- `svn switch`

- `svn update`

- `svn resolved`

- `svnadmin create`

- `svnadmin dump`

- `svnadmin hotcopy`

- `svnadmin list-dblogs`

- `svnadmin list-unused-dblogs`

- `svnadmin load`

- `svnadmin lstxns`

- `svnadmin recover`

- `svnadmin rmtxns`

- `svnadmin setlog`

- `svnadmin verify`

- 

## svn相关工具包

- **svnkit**：pure Java Subversion client

  这个说纯java客户端，其实比subversion还大，不建议安装

- **subversion-tools**：Assorted tools related to Apache Subversion(与Apache Subversion相关的各种工具)

- **subversion**：Advanced version control system(高级版本控制系统)

