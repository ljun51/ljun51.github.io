---
layout: post
title: VIM配置
description: VIM作为开发人员最常用的文本编辑工具，特别是作为中、高级开发、管理、运维人员是必须要熟悉，因为很多服务器都是没有桌面环境的；不管是查看服务器上的资源、配置，还是查看系统日志，都需要一个顺手的工具。
date: 2017-01-06
---

VIM作为开发人员最常用的文本编辑工具，特别是作为中、高级开发、管理、运维人员是必须要熟悉的，因为很多服务器都是没有桌面环境的，不管是查看服务器上的资源、配置，还是查看系统日志，都需要一个顺手的工具。下面是我的VIM配置和插件。

### 配置

#### 基本配置
```
"pathogen
execute pathogen#infect()
call pathogen#helptags()
set backspace=indent,eol,start " backspace over everything in insert mode
filetype on
filetype plugin on
filetype plugin indent on
syntax enable
syntax on
set autoindent
set cindent
set cursorline
" set cursorcolumn
set incsearch
set nocp
set nu
set ruler
set shiftwidth=4
set showmatch
set showmode
set smartindent
set smarttab
set tabstop=4
set title
set laststatus=2
set autowrite
"let &termencoding=&encoding
"set fileencodings=uft-8,gbk,gb2312
if has("multi_byte")
	"if &termencoding == "
		let &termencoding=&encoding
	"endif
	set encoding=UTF-8
	setglobal fileencoding=UTF-8
	"setglobal bomb
	set fileencodings=UTF-8
endif
"set foldmethod=indent
"set foldmethod=marker
"set foldmethod=syntax
set nofoldenable
```

#### 插件配置
```
" tagbar
nmap <F8> :TagbarToggle<CR>
C>
inoremap <C-h> <Left>
inoremap <C-j> <Down>
inoremap <C-k> <Up>
inoremap <C-l> <Right>
inoremap <C-a> <HOME>
inoremap <C-e> <END>
inoremap <C-f> <S-Right>
inoremap <C-b> <S-Left>
" ;q formatted, ;h cancel formatted
vmap <silent> ;h :s?^\(\s*\)+ '\([^']\+\)',*\s$*?\1\2?g<CR>
vmap <silent> ;q :s?^\(\s*\)\(.*\)\s*$? \1 + '\2'?<CR>

" vim-go
let g:go_highlight_functions = 1
au FileType go nmap <leader>r <Plug>(go-run)
au FileType go nmap <leader>b <Plug>(go-build)
au FileType go nmap <leader>t <Plug>(go-test)
au FileType go nmap <leader>c <Plug>(go-coverage)
au FileType go nmap <Leader>gd <Plug>(go-doc)
au FileType go nmap <Leader>gb <Plug>(go-doc-browser)
au FileType go nmap <Leader>s <Plug>(go-implements)
au FileType go nmap <Leader>i <Plug>(go-install)
au FileType go nmap <Leader>e <Plug>(go-rename)
au FileType go nmap <Leader>p <Plug>(go-import)

" nerdtree
autocmd vimenter * NERDTree
autocmd StdinReadPre * let s:std_in=1
autocmd VimEnter * if argc() == 0 && !exists("s:std_in") | NERDTree | endif
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree") && b:NERDTree.isTabTree()) | q | endif
" cursor focus right
autocmd VimEnter * NERDTree
wincmd w
autocmd VimEnter * wincmd w
nmap <F3> :NERDTreeToggle<CR>

" vim-javacomplete2
autocmd FileType java setlocal omnifunc=javacomplete#Complete

" ctrlp
set runtimepath^=~/.vim/bundle/ctrlp.vim
let g:ctrlp_custom_ignore = {
    \ 'dir':  '\v[\/]\.(git|hg|svn|rvm|idea)$',
	\ 'file': '\v\.(exe|so|dll|zip|tar|tar.gz|pyc)$',
	\ }
let g:ctrlp_working_path_mode='ra'
let g:ctrlp_regexp = 1
			
" neocomplete
let g:neocomplete#enable_at_startup = 1

" emmet-vim
let g:user_emmet_leader_key='<C-Z>' " default <C-Y>

" syntastic 
set statusline+=%#warningmsg#
set statusline+=%{SyntasticStatuslineFlag()}
set statusline+=%*

let g:syntastic_javascript_checkers = ['eslint']
"let g:syntastic_disabled_filetypes=['html']
let g:syntastic_always_populate_loc_list = 1
let g:syntastic_auto_loc_list = 1
let g:syntastic_check_on_open = 0
let g:syntastic_check_on_wq = 1
nnoremap <leader>sc :SyntasticCheck<CR> 
nnoremap <leader>st :SyntasticToggleMode<CR>
function! SyntasticCheckHook(errors)
if !empty(a:errors)    
let g:syntastic_loc_list_height = min([len(a:errors), 10])
endif
endfunction

" smartim
let g:smartim_default = 'com.apple.keylayout.USInternational-PC'

" fastfold
let g:vimsyn_folding='af'
let g:xml_syntax_folding = 1
let g:fastfold_savehook = 1
nmap zuz (FastFoldUpdate)}))])'"
```

### 插件

##### FastFold 代码折叠
##### ctrlp.vim 文件搜索
##### delimitMate 自动匹配括号
##### editorconfig-vim 代码风格
##### emmet-vim 
##### git-vim
##### html5.vim
##### neocomplete.vim 自动完成
##### nerdcommenter 注释
##### nerdtree 文件管理
##### nerdtree-git-plugin 文件管理集成git
##### smartim 输入法切换
##### syntastic 语法检查
##### tabular 
##### tern_for_vim javascript自动完成
##### toobar 代码浏览
##### ultisnips
##### vim-css3-syntax
##### vim-easygrep
##### vim-go
##### vim-javacomplete2
##### vim-markdown
##### vim-powerline 状态栏增强

### tmux.conf
```
#
# author  : ljun51<ljun51@outlook.com>
# modified: 2017-01-04
#

# base settings
set -g escape-time 0
set -g history-limit 65535
set -g base-index 1
set -g pane-base-index 1

# split window
unbind '"'
# vertical split (prefix -)
bind - splitw -v
unbind %
bind | splitw -h # horizontal split (prefix |)

# select pane
bind k selectp -U # above (prefix k)
bind j selectp -D # below (prefix j)
bind h selectp -L # left (prefix h)
bind l selectp -R # right (prefix l)

# resize pane
bind -r ^k resizep -U 10 # upward (prefix Ctrl+k)
bind -r ^j resizep -D 10 # downward (prefix Ctrl+j)
bind -r ^h resizep -L 10 # to the left (prefix Ctrl+h)
bind -r ^l resizep -R 10 # to the right (prefix Ctrl+l)

# swap pane
# swap with the previous pane (prefix Ctrl+u)
bind ^u swapp -U
# swap with the next pane (prefix Ctrl+d)
bind ^d swapp -D

# misc
# select the last pane (prefix e)
bind e lastp
# select the last window (prefix Ctrl+e)
bind ^e last
# kill pane (prefix q)
bind q killp
# kill window (prefix Ctrl+q)
bind ^q killw

# copy mode
# enter copy mode (prefix Ctrl+i)
bind ^i copy-mode
# paste buffer (prefix Ctrl+p)
bind ^p pasteb
# select (v)
bind -t vi-copy v begin-selection
# copy (y)
bind -t vi-copy y copy-selection

set-window-option -g mode-mouse on
setw -g mode-keys vi

# reload config (prefix r)
bind r source ~/.tmux.conf \; display "Configuration reloaded!"
```
