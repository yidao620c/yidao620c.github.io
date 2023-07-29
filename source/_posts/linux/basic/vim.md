---
title: vim简易教程
date: 2016-06-02 20:07:22 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

vim 是 Linux 系统上的最著名的文本/代码编辑器，也是早年的 Vi 编辑器的加强版，
而 gvim 则是其 Windows 版。它的最大特色是完全使用键盘命令进行编辑，脱离了鼠标操作虽然使得入门变得困难，
但上手之后键盘流的各种巧妙组合操作却能带来极为大幅的效率提升。

因此 vim 和现代的编辑器（如 Sublime Text）有着非常巨大的差异，而且入门学习曲线陡峭，
需要记住很多按键组合和命令，如今被看作是高手、Geek们专用的编辑器。尽管 vim 已经是古董级的软件，
但还是有无数新人迎着困难去学习使用，可见其经典与受欢迎程度。另外，由于 vim 的可配置性非常强，
各种插件、语法高亮配色方案等多不胜数，无论作为代码编辑器或是文稿撰写工具都非常给力。
<!-- more -->

## 安装

将来要支持python开发，先安装

```bash
yum install python-devel
```

这一步比较简单，在centos上面一条命令:

```bash
yum install vim
vim --version
VIM - Vi IMproved 7.4
```

下载源码并进行编译安装(8.0有些问题，暂时不建议安装):

```bash
wget https://github.com/vim/vim/archive/master.zip
unzip master.zip
cd vim-master
# 下面的python版本支持根据需要删
./configure --prefix=/usr/local/vim8
make && make install

# 卸载源码安装的vim,下面替换成你已经装好的vim命令路径
make VIMRUNTIMEDIR=/usr/local/vim8/bin/vim
make uninstall
```

## 配置

Vim本身能够满足开发人员的很多需求，但是它的可扩展性也极强，并且已经有一些杀手级的扩展，
可以让Vim拥有"现代"集成开发环境的特性。所以，你所需要的第一件东西就是一个好用的扩展管理器。

Vim有多个扩展管理器，但是我们强烈推荐[Vundle](https://github.com/gmarik/Vundle.vim)，
你可以把它想象成Vim的pip，有了Vundle，安装和更新包这种事情不费吹灰之力。

我们现在来安装Vundle：

```bash
git clone https://github.com/gmarik/Vundle.vim.git ~/.vim/bundle/Vundle.vim
```

该命令将下载Vundle插件管理器，并将它放置在你的Vim编辑器bundles文件夹中。
现在，你可以通过.vimrc配置文件来管理所有扩展了。

我这里有一个比较优化的配置了，将vim打造成一个IDE， 编辑`~/.vimrc`，输入如下内容

```vim
"vundle
set nocompatible
filetype off

set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

Plugin 'VundleVim/Vundle.vim'
"git interface
Plugin 'tpope/vim-fugitive'
"filesystem
Plugin 'scrooloose/nerdtree'
Plugin 'jistr/vim-nerdtree-tabs'
Plugin 'kien/ctrlp.vim'

"html
"  isnowfy only compatible with python not python3
Plugin 'isnowfy/python-vim-instant-markdown'
Plugin 'jtratner/vim-flavored-markdown'
Plugin 'suan/vim-instant-markdown'
Plugin 'nelstrom/vim-markdown-preview'
"python sytax checker
Plugin 'nvie/vim-flake8'
Plugin 'vim-scripts/Pydiction'
Plugin 'vim-scripts/indentpython.vim'
Plugin 'scrooloose/syntastic'

"auto-completion stuff
"Plugin 'klen/python-mode'
Plugin 'Valloric/YouCompleteMe'
Plugin 'klen/rope-vim'
"Plugin 'davidhalter/jedi-vim'
Plugin 'ervandew/supertab'
""code folding
Plugin 'tmhedberg/SimpylFold'

"Colors!!!
Plugin 'altercation/vim-colors-solarized'
Plugin 'jnurmine/Zenburn'

call vundle#end()

filetype plugin indent on    " enables filetype detection
let g:SimpylFold_docstring_preview = 1

"autocomplete
let g:ycm_autoclose_preview_window_after_completion=1

"custom keys
let mapleader=" "
map <leader>g  :YcmCompleter GoToDefinitionElseDeclaration<CR>
"
call togglebg#map("<F5>")
"colorscheme zenburn
"set guifont=Monaco:h14

let NERDTreeIgnore=['\.pyc$', '\~$'] "ignore files in NERDTree

"I don't like swap files
set noswapfile

"turn on numbering
set nu

"python with virtualenv support
py << EOF
import os.path
import sys
import vim
if 'VIRTUA_ENV' in os.environ:
  project_base_dir = os.environ['VIRTUAL_ENV']
  sys.path.insert(0, project_base_dir)
  activate_this = os.path.join(project_base_dir,'bin/activate_this.py')
  execfile(activate_this, dict(__file__=activate_this))
EOF

"it would be nice to set tag files by the active virtualenv here
":set tags=~/mytags "tags for ctags and taglist
"omnicomplete
autocmd FileType python set omnifunc=pythoncomplete#Complete

"------------Start Python PEP 8 stuff----------------
" Number of spaces that a pre-existing tab is equal to.
au BufRead,BufNewFile *py,*pyw,*.c,*.h set tabstop=4

"spaces for indents
au BufRead,BufNewFile *.py,*pyw set shiftwidth=4
au BufRead,BufNewFile *.py,*.pyw set expandtab
au BufRead,BufNewFile *.py set softtabstop=4

" Use the below highlight group when displaying bad whitespace is desired.
highlight BadWhitespace ctermbg=red guibg=red

" Display tabs at the beginning of a line in Python mode as bad.
au BufRead,BufNewFile *.py,*.pyw match BadWhitespace /^\t\+/
" Make trailing whitespace be flagged as bad.
au BufRead,BufNewFile *.py,*.pyw,*.c,*.h match BadWhitespace /\s\+$/

" Wrap text after a certain number of characters
au BufRead,BufNewFile *.py,*.pyw, set textwidth=100

" Use UNIX (\n) line endings.
au BufNewFile *.py,*.pyw,*.c,*.h set fileformat=unix

" Set the default file encoding to UTF-8:
set encoding=utf-8

" For full syntax highlighting:
let python_highlight_all=1
syntax on

" Keep indentation level from previous line:
autocmd FileType python set autoindent

" make backspaces more powerfull
set backspace=indent,eol,start


"Folding based on indentation:
autocmd FileType python set foldmethod=indent
"use space to open folds
nnoremap <space> za
"----------Stop python PEP 8 stuff--------------

"js stuff"
autocmd FileType javascript setlocal shiftwidth=2 tabstop=2

syntax on               "语法高亮显示。
set encoding=utf-8
set hlsearch            "高亮度反白
set backspace=2         "可以用Backspace键删除
set ts=4                "tab键等于4个空格
set expandtab           "tab键自动变空格
set tabstop=4
set softtabstop=4
set autoindent          "自动缩进
set pastetoggle=<F9>    "插入模式粘贴按F9取消自动缩进
set ruler               "可显示最后一行的状态
set showmode            "左下角那一行的状态
set nu                  "可以在每一行的最前面显示行号啦！
set bg=dark             "显示不同的底色色调
"set cursorline          "光标所在行一横线
set laststatus=2        "显示当前编辑文件名
set showcmd
set magic
set lazyredraw
set history=100         "历史记录条数
set hlsearch            "高亮显示搜索结果
set incsearch           "增量搜索，每次输入一个字母都自动搜
"choose and replace
vnoremap ,s y:%s/<C-R>=escape(@", '\\/.*$^~[]')<CR>/

```

注：把配置写进去后，要在vim里面执行`:PluginInstall`命令让Vundle自动帮你安装完所有插件。

### 问题记录

出现过问题"ouCompleteMe unavailable: No module named ycmd"

解决办法:

```bash
cd ~/.vim/bundle/YouCompleteMe
git pull
git submodule update --init --recursive
```

但是执行过程报错: `The remote end hung up unexpectedly`，这是因为网络不稳定，
要下载的东西很多，多试几次耐心一点就可以了。

----------------------------------

The ycmd server SHUT DOWN (restart with ':YcmRestartServer')

YCM需要手动编译才行，到`.vim/bundle/YouCompleteMe`下跑

```bash
 ./install.sh --clang-completer
```

耐心等待大概1个小时可以下载并安装完...

下面是我配置完成后的vim:

![](https://xnstatic-1253397658.file.myqcloud.com/vim02.png)

## 入门

vim有三种模式：

1. 一般模式：在Linux终端中输入"vim 文件名"就进入了一般模式,但不能输入文字。
2. 编辑模式：在一般模式下按i就会进入编辑模式，此时就可以写程式，按Esc可回到一般模式。
3. 命令模式：在一般模式下按：就会进入命令模式，左下角会有一个冒号出现，此时可以敲入命令并执行。

当你安装好一个编辑器后，你一定会想在其中输入点什么东西，然后看看这个编辑器是什么样子。
但vim不是这样的，请按照下面的命令操作:

* 启动Vim后，vim在 Normal 模式下。
* 让我们进入 Insert 模式，请按下键 i 。(vim左下角有一个–insert–字样，表示你可以输入了）
* 此时，你可以输入文本了，就像你用"记事本"一样。
* 如果你想返回 Normal 模式，请按 ESC 键

现在，你知道如何在 Insert 和 Normal 模式下切换了。下面是一些命令，可以让你在 Normal 模式下幸存下来：

```
i   → Insert 模式，按 ESC 回到 Normal 模式.
x   → 删当前光标所在的一个字符。
:wq → 存盘 + 退出 (:w 存盘, :q 退出)   （陈皓注：:w 后可以跟文件名）
dd  → 删除当前行，并把删除的行存到剪贴板里
p   → 粘贴剪贴板
```

推荐:

```
hjkl (强例推荐使用其移动光标，但不必需) →你也可以使用光标键 (←↓↑→). 注: j 就像下箭头。
:help <command> → 显示相关命令的帮助。你也可以就输入 :help 而不跟命令。（陈皓注：退出帮助需要输入:q）
```

## 进阶1

现在是时候学习一些更多的命令了，下面是我的建议：

各种插入模式

```
a  → 在光标后插入
o  → 在当前行后插入一个新行
O  → 在当前行前插入一个新行
cw → 替换从光标所在位置后到一个单词结尾的字符
```

简单的移动光标

```
0         → 数字零，到行头
^         → 到本行第一个不是blank字符的位置（所谓blank字符就是空格，tab，换行，回车等）
$         → 到本行行尾
g_        → 到本行最后一个不是blank字符的位置。
/pattern  → 搜索 pattern 的字符串（陈皓注：如果搜索出多个匹配，可按n键到下一个）
```

拷贝/粘贴(p/P都可以，p是表示在当前位置之后，P表示在当前位置之前)

```
P  → 粘贴
yy → 拷贝当前行当行于 ddP
```

Undo/Redo

```
u     → undo
<C-r> → redo
```

打开/保存/退出/改变文件(Buffer)

```
:e <path/to/file>      → 打开一个文件
:w                     → 存盘
:saveas <path/to/file> → 另存为 <path/to/file>
:x， ZZ 或 :wq          → 保存并退出 (:x 表示仅在需要时保存，ZZ不需要输入冒号并回车)
:q!                    → 退出不保存 :qa! 强行退出所有的正在编辑的文件，就算别的文件有更改。
:bn 和 :bp             → 你可以同时打开很多文件，使用这两个命令来切换下一个或上一个文件。
```

## 进阶2

重复命令次数

```
N<command> → 重复某个命令N次
2dd        → 删除2行
3p         → 粘贴文本3次
```

你要让你的光标移动更有效率，你一定要了解下面的这些命令，千万别跳过

```
NG → 到第 N 行 （陈皓注：注意命令中的G是大写的，另我一般使用 : N 到第N行，如 :137 到第137行）
gg → 到第一行。（陈皓注：相当于1G，或 :1）
G  → 到最后一行
```

按单词移动

```
w      → 到下一个单词的开头。
e      → 到下一个单词的结尾。
```

下面这三个命令对程序员来说是相当强大的

```
%      → 匹配括号移动，包括 (, {, [. （你需要把光标先移到括号上）
* 和 #  →  匹配光标当前所在的单词，移动光标到下一个（或上一个）匹配单词（*是下一个，#是上一个）
```

很多命令都是如下方式工作：

```
<start position><command><end position>
```

例如`0y$`命令意味着：

```
0 → 先到行头
y → 从这里开始拷贝
$ → 拷贝到本行最后一个字符
```

还有很多时间并不一定你就一定要按y才会拷贝，下面的命令也会被拷贝

```
d  (删除 )
v  (可视化的选择)
gU (变大写)
gu (变小写)
```

## 进阶3

你只需要掌握前面的命令，你就可以很舒服的使用VIM了。但是，现在，我们向你介绍的是VIM杀手级的功能

在当前行上移动光标:

```
0      → 到行头
^      → 到本行的第一个非blank字符
$      → 到行尾
g_     → 到本行最后一个不是blank字符的位置。
fa     → 到下一个为a的字符处，你也可以fs到下一个为s的字符。
t,     → 到逗号前的第一个字符。逗号可以变成其它字符。
3fa    → 在当前行查找第三个出现的a。
F 和 T  → 和 f 和 t 一样，只不过是相反方向。
```

## 区块操作

典型列编辑模式：

```
<Ctrl-v>  → 开始块操作
<hjkl>    → 移动光标
<Shift-i> → 插入字符
<ESC>     → 生效
```

可视化选择： v,V,<C-v>

## 多tab编辑

在不同的tab中打开多个文档，这个特性我一直很喜欢，就跟浏览器或各个IDE中是一样的

```
:tabnew filename在一个新的tab中打开文件
gt在不同的tab中切换，use 5gt to switch to tab 5，从1开始
:tabclose或者:q关闭当前tab
:tabmove将tab移动到指定的位置

:tabe {file}   edit specified file in a new tab
:tabc          close current tab
:tabc {i}      close i-th tab
:tabo          close all other tabs
:tabs          list all tabs
:tabm 0        move current tab to first
:tabm          move current tab to last
:tabm {i}      move current tab to position i+1
:tabn          go to next tab
:tabp          go to previous tab
:tabfirst      go to first tab
:tablast       go to last tab
```

## 分割布局

使用`sv <filename>`命令打开一个文件，你可以纵向分割布局（新文件会在当前文件下方界面打开），
使用相反的命令`vs <filename>``你可以得到横向分割布局（新文件会在当前文件右侧界面打开）。

![](https://xnstatic-1253397658.file.myqcloud.com/vim01.png)

你还可以嵌套分割布局，所以你可以在分割布局内容再进行分割，纵向或横向都可以，
直到你满意为止。众所周知，我们开发时经常需要同时查看多个文件。

专业贴士：记得在输入完:sv后，利用tab补全功能，快速查找文件

窗口跳转指令:

```
<Ctrl-w><Ctrl-j> 切换到下方的分割窗口
<Ctrl-w><Ctrl-k> 切换到上方的分割窗口
<Ctrl-w><Ctrl-l> 切换到右侧的分割窗口
<Ctrl-w><Ctrl-h> 切换到左侧的分割窗口
<Ctrl-ww>        切换到下一个分割窗口
:q     关闭当前窗口
:qall  关闭所有窗口并退出
```

## 缓冲区(Buffers)

虽然Vim支持tab操作，仍有很多人更喜欢缓冲区和分割布局，你可以把缓冲区想象成最近打开的一个文件。
Vim提供了方便访问近期缓冲区的方式，只需要输入`:b <buffer name or number>`
就可以切换到一个已经开启的缓冲区（此处也可使用自动补全功能），你还可以通过`:ls`命令查看所有的缓冲区。

## FAQ

不小心按到ctrl+s，结果就不动了

原来是Linux的一个快捷键呀，干什么用的？
原来Ctrl+S在Linux里，是锁定屏幕的快捷键。如果要解锁，按下Ctrl+Q就可以了

## NERDTree插件

NERDTree的作用就是列出当前路径的目录树，一般IDE都是有的。
可以方便的浏览项目的总体的目录结构和创建删除重命名文件或文件名。
至于它的配置我做了如下修改:

```vim
" NERDTree config
map <F2> :NERDTreeToggle<CR>
```

使用F2键快速调出和隐藏它；

如果想打开vim时自动打开NERDTree，可以如下设定:

```vim
autocmd vimenter * NERDTree
```

下面总结一些命令,非常的多，只要记住一些你常用的就行了，后名添加！！！的是我常用的
(原文地址：<http://yang3wei.github.io/blog/2013/01/29/nerdtree-kuai-jie-jian-ji-lu/>）

最常用窗口跳转:

```
ctrl + w + h    光标 focus 左侧树形目录
ctrl + w + l    光标 focus 右侧文件显示窗口
ctrl + w + w    光标自动在左右侧窗口切换 #！！！
ctrl + w + r    移动当前窗口的布局位置
```

主要命令集合:

```
o       在已有窗口中打开文件、目录或书签，并跳到该窗口
go      在已有窗口中打开文件、目录或书签，但不跳到该窗口
t       在新 Tab 中打开选中文件/书签，并跳到新 Tab
T       在新 Tab 中打开选中文件/书签，但不跳到新 Tab
i       split 一个新窗口打开选中文件，并跳到该窗口
gi      split 一个新窗口打开选中文件，但不跳到该窗口
s       vsplit 一个新窗口打开选中文件，并跳到该窗口
gs      vsplit 一个新 窗口打开选中文件，但不跳到该窗口
!       执行当前文件
O       递归打开选中 结点下的所有目录
x       合拢选中结点的父目录
X       递归 合拢选中结点下的所有目录
e       Edit the current dif
D       删除当前书签
P       跳到根结点
p       跳到父结点
K       跳到当前目录下同级的第一个结点
J       跳到当前目录下同级的最后一个结点
k       跳到当前目录下同级的前一个结点
j       跳到当前目录下同级的后一个结点

C       将选中目录或选中文件的父目录设为根结点
u       将当前根结点的父目录设为根目录，并变成合拢原根结点
U       将当前根结点的父目录设为根目录，但保持展开原根结点
r       递归刷新选中目录
R       递归刷新根结点
m       显示文件系统菜单 #！！！然后根据提示进行文件的操作如新建，重命名等
cd      将 CWD 设为选中目录

I       切换是否显示隐藏文件
f       切换是否使用文件过滤器
F       切换是否显示文件
B       切换是否显示书签

q       关闭 NerdTree 窗口
?       切换是否显示 Quick Help
```

tab页操作:

```
:tabnew file 建立对指定文件新的tab
:tabc        关闭当前的 tab
:tabo        关闭所有其他的 tab
:tabs        查看所有打开的 tab
:tabp        前一个 tab
:tabn        后一个 tab
```
