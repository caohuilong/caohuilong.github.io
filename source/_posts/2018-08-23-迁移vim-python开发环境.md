---
title: 迁移vim-python开发环境
date: 2018-08-23 16:06:23
categories:
  - "VIM"
tags:
  - "vim"
  - "python"
  - "YouCompleteMe"
---

# 概述

之前配置好了自己的 vim python 开发环境，为了方便在其他主机上面迁移之前的开发环境，所以将所有的插件包都打包压缩了一份，和 vim 配置文件 .vimrc 一起上传到了百度云盘（下载链接：[vim插件包](https://pan.baidu.com/s/1QJtx7CPloS4sKmwD3xjj8g) 密码：neio）。通过下载这个文件夹就可以在新的主机上部署 vim 的 python 开发环境了。

<!--more-->

# 步骤

1. 下载插件包与配置文件

   先使用百度云盘下载这两个文件，然后再通过 ftp 上传到你的 Linux 主机上。我也试过直接在 Linux 里面通过 wget 来下载百度云盘的文件，但是这个要使用浏览器来生成下载链接，挺麻烦的，所以我就不在这里说明了。

2. 将 .vimrc 放到用户主目录下，即 `~/` 目录

   .vimrc 文件里面主要记录需要安装哪些插件，我的 vim python 开发环境安装的插件有：

   - `VundleVim/Vundle.vim`
   - `Valloric/YouCompleteMe`
   - `Lokaltog/vim-powerline`
   - `scrooloose/nerdtree`
   - `Yggdroot/indentLine`
   - `jiangmiao/auto-pairs`
   - `tell-k/vim-autopep8`
   - `scrooloose/nerdcommenter`
   - `altercation/vim-colors-solarized`
   - `w0rp/ale`
   - `scrooloose/syntastic`
   - `nvie/vim-flake8`

   以及一些常用的配置信息，具体如下所示

   ```
   $ vi ~/.vimrc
   "去掉vi的一致性"
   set nocompatible
   
   filetype off
   set rtp+=~/.vim/bundle/Vundle.vim
   call vundle#begin()
   Plugin 'VundleVim/Vundle.vim'
   Plugin 'Valloric/YouCompleteMe'
   Plugin 'Lokaltog/vim-powerline'
   Plugin 'scrooloose/nerdtree'
   Plugin 'Yggdroot/indentLine'
   Plugin 'jiangmiao/auto-pairs'
   Plugin 'tell-k/vim-autopep8'
   Plugin 'scrooloose/nerdcommenter'
   Plugin 'altercation/vim-colors-solarized'
   "Plugin 'w0rp/ale'
   Plugin 'scrooloose/syntastic'
   Plugin 'nvie/vim-flake8'
   call vundle#end()
   filetype plugin indent on
   
   
   "显示行号"
   set number
   " 隐藏滚动条"    
   "set guioptions-=r 
   "set guioptions-=L
   "set guioptions-=b
   "隐藏顶部标签栏"
   "set showtabline=0
   "设置字体"
   set guifont=Monaco:h13         
   set nowrap  "设置不折行"
   "set fileformat=unix "设置以unix的格式保存文件"
   "set cindent     "设置C样式的缩进格式"
   set tabstop=4   "设置table长度"
   set shiftwidth=4        "同上"
   set showmatch   "显示匹配的括号"
   set scrolloff=5     "距离顶部和底部5行"
   set laststatus=2    "命令行为两行"
   set fenc=utf-8      "文件编码"
   set backspace=2
   set mouse=v     "启用鼠标"
   set selection=exclusive
   set selectmode=mouse,key
   set matchtime=5
   set ignorecase      "忽略大小写"
   set incsearch
   set hlsearch        "高亮搜索项"
   set noexpandtab     "不允许扩展table"
   set whichwrap+=<,>,h,l
   set autoread
   set cursorline      "突出显示当前行"
   "set cursorcolumn        "突出显示当前列"
   syntax on   "开启语法高亮"
   "set background=dark     "设置背景色"
   "colorscheme solarized
   "let g:solarized_termcolors=256  "solarized主题设置在终端下的设置"
   
   "syntastic
   let python_highlight_all=1
   "设置error和warning的标志
   let g:syntastic_enable_signs=1
   let g:syntastic_error_symbol='✗'
   let g:syntastic_warning_symbol='►'
   "总是打开Location
   "List（相当于QuickFix）窗口，如果你发现syntastic因为与其他插件冲突而经常崩溃，将下面选项置0
   let g:syntastic_always_populate_loc_list = 0
   "自动打开LocatonList，默认值为2，表示发现错误时不自动打开，当修正以后没有再发现错误时自动关闭，置1表示自动打开自动关闭，0表示关闭自动打开和自动关闭，3表示自动打开，但不自动关闭
   let g:syntastic_auto_loc_list = 2
   "修改Locaton List窗口高度
   let g:syntastic_loc_list_height = 3
   "打开文件时自动进行检查
   let g:syntastic_check_on_open = 1
   let g:syntastic_check_on_wq = 1
   "自动跳转到发现的第一个错误或警告处
   let g:syntastic_auto_jump = 1
   "高亮错误
   let g:syntastic_enable_highlighting=0
   "设置pyflakes为默认的python语法检查工具
   let g:syntastic_python_checkers = ['pyflakes']
   
   "按F5运行python"
   map <F5> :call RunPython()<CR>
   function RunPython()
   	exec "W"
   	if &filetype == 'python'
   		exec "!time python %"
   	endif
   endfunction
   
   "默认配置文件路径"
   let g:ycm_global_ycm_extra_conf = '~/.ycm_extra_conf.py'
   "打开vim时不再询问是否加载ycm_extra_conf.py配置"
   let g:ycm_confirm_extra_conf=0
   set completeopt=longest,menu
   "python解释器路径"
   let g:ycm_path_to_python_interpreter='/usr/bin/python'
   "是否开启语义补全"
   let g:ycm_seed_identifiers_with_syntax=1 
   "是否在注释中也开启补全" 
   let g:ycm_complete_in_comments=1 
   let g:ycm_collect_identifiers_from_comments_and_strings = 0
   "开始补全的字符数"
   let g:ycm_min_num_of_chars_for_completion=2
   "补全后自动关机预览窗口"
   let g:ycm_autoclose_preview_window_after_completion=1
   " 禁止缓存匹配项,每次都重新生成匹配项"
   let g:ycm_cache_omnifunc=0
   "字符串中也开启补全"
   let g:ycm_complete_in_strings = 1
   "离开插入模式后自动关闭预览窗口"
   autocmd InsertLeave * if pumvisible() == 0|pclose|endif
   
   "回车即选中当前项"
   "inoremap <expr> <CR>       pumvisible() ? '<C-y>' : '\<CR>'
   "上下左右键行为"
   inoremap <expr> <Down>     pumvisible() ? '\<C-n>' : '\<Down>'
   inoremap <expr> <Up>       pumvisible() ? '\<C-p>' : '\<Up>'
   inoremap <expr> <PageDown> pumvisible() ? '\<PageDown>\<C-p>\<C-n>' : '\<PageDown>'
   inoremap <expr> <PageUp>   pumvisible() ? '\<PageUp>\<C-p>\<C-n>' : '\<PageUp>'
   
   "F2开启和关闭树"
   map <F2> :NERDTreeToggle<CR>
   let NERDTreeChDirMode=1
   "显示书签"
   let NERDTreeShowBookmarks=1
   "设置忽略文件类型"
   let NERDTreeIgnore=['\~$', '\.pyc$', '\.swp$']
   "窗口大小"
   let NERDTreeWinSize=25
   
   "split navigations
   nnoremap <C-J> <C-W><C-J>
   nnoremap <C-K> <C-W><C-K>
   nnoremap <C-L> <C-W><C-L>
   nnoremap <C-H> <C-W><C-H>
   
   "缩进指示线"
   let g:indentLine_char='┆'
   let g:indentLine_enabled = 1
   
   "autopep8设置"
   let g:autopep8_disable_show_diff=1
   
   let mapleader=','
   
   map <F4> <leader>ci <CR>
   ```

3. 在用户主目录下新建一个 .vim 文件夹，并将插件包解压缩至该文件夹

   ```
   $ mkdir ~/.vim
   $ tar -jxv -f bundle.tar.bz2 -C ~/.vim
   ```

   到这里，除了自动补全的插件 YouCompleteMe ，其实大部分的插件都已经起作用了，我们的插件包有几百兆主要就是因为 YouCompleteMe 这个插件比较大，这也是因为这个插件的功能太强大了，这个插件在下载完成后还需要编译安装，接下来就来完成这个步骤。

4. 安装 python 和 python 库

   ```
   $ sudo apt install python python-dev
   ```

   这一步没有完成在安装的时候可能会碰到下面的问题：

   ```
   WARNING: this script is deprecated. Use the install.py script instead.
   Searching Python 2.7 libraries...
   ERROR: unable to find an appropriate Python library.
   ```

5. 安装编译环境

   ```
   $ sudo apt install cmake gcc build-essential
   ```

   未完成这步可能会遇到的问题：

   ```
   WARNING: this script is deprecated. Use the install.py script instead.
   ERROR: Unable to find executable 'cmake'. CMake is required to build ycmd
   ```

   ```
   No CMAKE_CXX_COMPILER could be found.
   ```

6. 执行 YouCompleteMe 安装脚本

   ```
   $ cd ~/.vim/bundle/YouCompleteMe
   $ ./install.sh
   ```

   完成上面的过程就实现了 YouCompleteMe 的安装，接下来就可以体验 vim 强大的功能啦！