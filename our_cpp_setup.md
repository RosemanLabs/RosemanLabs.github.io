# Our C++ environment  
Here is our setup for programming in C++. It took some time to get it right, we are quite happy with it. 
We hope it can be useful to others.

## Clang Tidy project wide config
Put a file named `.clang-tidy` in your project root with e.g. the following content:
```
Checks:    'clang-diagnostic-*,clang-analyzer-*,cppcoreguidelines-*,bugprone-*,hicpp-*'
```
## Clang Format project wide config
Put a file named `.clang-format` in your project root with some settings.
The contents are obviously up for debate, see also this [online configurator](https://zed0.co.uk/clang-format-configurator/).

```
---
BasedOnStyle: Microsoft
UseTab: Never
PointerAlignment: Left
AlwaysBreakTemplateDeclarations: Yes
BreakBeforeBraces: Stroustrup
...
```

## Editor: Vim / GVim
Our editor setup. Ctrl-k for clang format. Youcompleteme for autocompletion, clangd and clang-tidy-based linting (nowadays built into clangd).
Don't forget to symbolic-link the compilation database (`compile_commands.json`) from the build folder to the project's root. 
```
set encoding=utf-8
set nocompatible              " be iMproved, required
filetype off                  " required

" set the runtime path to include Vundle and initialize
set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()

" let Vundle manage Vundle, required
Plugin 'VundleVim/Vundle.vim'
Plugin 'ycm-core/YouCompleteMe'
Plugin 'bfrg/vim-cpp-modern'
Plugin 'niekbouman/oceandeep'

call vundle#end()            " required
filetype plugin indent on    " required

" line numbering
set nu

" syntax highlighting
syntax on
set lbr
set et
set tabstop=2
set shiftwidth=2

filetype on
filetype plugin on

colors oceandeep
set guifont=Source\ Code\ Pro\ Medium\ 14

" clang format
map <C-K> :py3f /usr/share/clang/clang-format.py<cr>
imap <C-K> <c-o>:py3f /usr/share/clang/clang-format.py<cr>

" Match pairs of C++ template brackets
set matchpairs+=<:>

let g:ycm_use_clangd = 1

" Let clangd fully control code completion
let g:ycm_clangd_uses_ycmd_caching = 0

" Use installed clangd, not YCM-bundled clangd which doesn't get updates.
let g:ycm_clangd_binary_path = exepath("clangd")
```
## Build System: Meson + Ninja 
Example top-level `meson.build` file:
```
project('myproject', 'cpp',
        default_options : ['cpp_std=c++2a'])

myproject_includes = include_directories('include')

cxx = meson.get_compiler('cpp')

if cxx.get_id() == 'clang'
  add_global_arguments('-ftime-trace', language : 'cpp')
  # for measuring compilation time
endif

subdir('apps')
```
where there is another `meson.build` file in the `apps` directory, containing, e.g.,

    executable('hello-world', 'hello-world.cpp')

## Linking with `lld` and fast recompiles with Mozilla's `sccache` compilation cache
For us, [`sccache`](https://github.com/mozilla/sccache) works much better than `ccache`. And [`lld`](https://lld.llvm.org/) is a fast linker.

We generate the project with:

Debug, clang (with sanitizers):

    CC='sccache clang' CXX='sccache clang++' CC_LD='lld' CXX_LD='lld' meson setup debug_build --buildtype=debug -Db_lundef=false -Db_sanitize=address,undefined

Release, clang:

    CC='sccache clang' CXX='sccache clang++' CC_LD=lld CXX_LD=lld meson setup clang_release_build --buildtype=release -Db_ndebug=true

Release, gcc:

    CC='sccache gcc' CXX='sccache g++' CC_LD=lld CXX_LD=lld meson setup clang_release_build --buildtype=release -Db_ndebug=true
