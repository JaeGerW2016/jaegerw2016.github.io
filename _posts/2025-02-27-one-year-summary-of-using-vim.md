---
layout: mypost
title: One year summary of using Vim
categories: [Linux, Vim]
---

VIM 使用一年总结
=============================

**简介**

本文是我使用 VIM 一年后的个人总结。我将分享我与 VIM 结缘的故事、VIM 的入门技巧、我常用的 VIM 快捷键，以及我的 ideavim 配置。希望这篇文章能够帮助读者更好地入门和使用 VIM。

**我的 VIM 之旅**

我第一次接触 VIM 是在学习 Linux 的时候。当时，我按照教程的命令，使用 VIM 编辑一个文件，最初觉得这个编辑器非常不友好。各种奇怪的模式和命令让我摸不着头脑。

大约在去年这个时候，VIM 的作者 Bram Moolenaar 去世了。在读了一篇分享 Bram Moolenaar 和 VIM 故事的文章后，我决定学习这款由如此伟大的程序员创造的编辑器。

今天，一年过去了，VIM 已经成为我日常开发中不可或缺的一部分：在浏览器中、在笔记中、在我的 IDE 中，我都能看到 VIM 的身影。

**如何入门**

很多人都说 VIM 有一个非常陡峭的学习曲线，但是我在开始使用 VIM 时并没有遇到太多难以克服的困难。相反，当你逐渐开始使用这个编辑器，并遇到让你觉得“别扭”的地方时，你可以学习更多的快捷键和配置，让 VIM 适应你的个人习惯，而不是强迫自己使用不舒服的方法。

入门 VIM，我最初使用的方法是 VIM Tutor。这是一个内置于 VIM 的经典教程。

如果你使用 VIM，你可以使用以下命令打开它：

    vimtutor
    

如果你使用 Neovim，你可以使用以下命令在 Neovim 的 Normal 模式下打开它：

    :Tutor
    

VIM Tutor 提供了一个大约需要 30 分钟的交互式教程，涵盖了 VIM 中最常用的快捷键和模式。

一旦你熟悉并记住了 VIM Tutor 中的大部分内容，我相信你将能够在 VIM 中“生存”下来。接下来，你可以阅读一些介绍 VIM 使用方法的博客。这些博客通常包含一些 VIM Tutor 和基本教程中没有涵盖的非常有用的快捷键，可以帮助你构建自己的快捷键库。

以下是一些在我刚开始使用 VIM 时对我帮助很大的优秀博客，你可以参考一下：

*   [Learn Vim Progressively](https://yannesposito.com/Scratch/en/blog/Learn-Vim-Progressively/)
*   [Vim Best Practices For IDE Users](https://medium.com/better-programming/50-vim-mode-tips-for-ide-users-f7b525a794b3)

如果你觉得这些博客不够正式，想要以更正式的方式学习 VIM，我建议阅读 VIM User Manual。它由 Bram Moolenaar 和社区贡献者编写，内容非常全面和详细。你可以使用以下命令在 VIM 或 Neovim 的 Normal 模式下打开它：

    :help user-manual
    

**常用快捷键**

本节主要分享一些我常用的 VIM 快捷键。我希望它们对你有所帮助。

*   `ci"`：删除引号内的所有内容并进入插入模式。

当您需要对各种类型的引号或括号内的内容进行操作时，这些快捷方式非常方便。您还可以使用其他运算符来构建类似的快捷方式，例如：

*   `yi(`：复制括号内的所有内容。
*   `va'`: 选择单引号和其中的所有内容，进入可视化模式。
*   `di<`: 删除尖括号内的所有内容。

为了更容易理解和记忆，`i` 代表“内部 (inner)”，表示对引号或括号内的内容进行操作，而 `a` 代表“周围 (around)”，表示对引号或括号周围的内容进行操作。

* * *

*   `daw`: 删除光标下的单词。

当您需要对单词、句子或段落进行操作时，这种类型的快捷方式非常有用。由于像 `cw` 和 `dw` 这样的命令需要在单词的开头使用才能对整个单词进行操作，因此您可能需要使用像 `b` 和 `w` 这样的快捷方式先移动到开头。`daw` 命令允许您在光标位于单词内时对整个单词进行操作，这使得它非常方便。类似的快捷方式包括：

*   `caw`: 删除一个单词和它后面的空格，进入插入模式。
*   `das`: 删除一个句子。
*   `cap`: 删除一个段落并进入插入模式。

* * *

*   `zz`: 将当前行居中显示在屏幕中间。
*   `zt`: 将当前行移动到屏幕顶部。
*   `zb`: 将当前行移动到屏幕底部。

* * *

*   `H`: 将光标移动到屏幕的顶行。
*   `M`: 将光标移动到屏幕的中间行。
*   `L`: 将光标移动到屏幕的底行。

* * *

*   `<C-y>`: 向上滚动当前窗口的内容一行，同时保持光标位置不变。
*   `<C-e>`: 向下滚动当前窗口的内容一行，同时保持光标位置不变。

* * *

*   `<C-w>v`: 垂直分割当前窗口。
*   `<C-w>s`: 水平分割当前窗口。
*   `<C-w>w`: 在窗口之间切换。
*   `<C-w>c`: 关闭当前窗口。
*   `<C-w>h`: 将焦点移动到左侧的窗口。
*   `<C-w>j`: 将焦点移动到下方的窗口。
*   `<C-w>k`: 将焦点移动到上方的窗口。
*   `<C-w>l`: 将焦点移动到右侧的窗口。

**IDEAVIM 配置**

作为一名 JetBrains 用户，在学习 VIM 之后，我配置了 ideavim 插件，以便在我的 IDE 中同时享受 JetBrains IDE 和 VIM 的便利。以下是一组根据我的个人习惯量身定制的 ideavim 配置。我希望它可以为您提供一些参考：

您也可以在这里找到它：[https://github.com/justlorain/euphonium](https://github.com/justlorain/euphonium)

    " Vim mode toggle
    nmap <leader>vim <Action>(VimPluginToggle)
    
    " --- Basic Configuration ---
    
    " leader key
    let mapleader = " "
    
    " Move to the previous/next line when pressing h/l at the beginning/end of a line
    set whichwrap=b,s,<,>,h,l,[,]
    
    "" visual shifting (builtin-repeat)
    vnoremap < <gv
    vnoremap > >gv
    
    " Vertical scroll offset
    set scrolloff=5
    
    " Search
    set incsearch
    set nohls
    set ic
    set smartcase
    nnoremap <leader>ss :set invhlsearch<CR>
    
    " Clipboard mapping
    set clipboard+=unnamed
    
    " Show line numbers
    set number
    " Set relative line numbers
    set relativenumber
    
    " Don't use Ex mode, use Q for formatting.
    map Q gq
    
    " --- Plugin Configuration ---
    
    " Highlight copied text
    Plug 'machakann/vim-highlightedyank'
    " Commentary plugin
    Plug 'tpope/vim-commentary'
    " vim-surround
    set surround
    " easymotion
    set easymotion
    " nerdtree
    set NERDTree
    nnoremap <leader>nf :NERDTreeFind<CR>
    " quickscope
    set quickscope
    let g:qs_highlight_on_keys = ['f', 'F', 't', 'T']
    " which-key
    set which-key
    set notimeout
    
    " --- Coding Configuration ---
    
    " Show file structure
    let g:WhichKeyDesc_FileStructure = "<leader>fs FileStructure"
    nmap <leader>fs <action>(FileStructurePopup)
    let g:WhichKeyDesc_FindFile = "<leader>ff FindFile"
    nmap <leader>ff <action>(GotoFile)
    " Close tab
    let g:WhichKeyDesc_CloseCurrentTab = "<leader>xx CloseCurrentTab"
    nmap <leader>xx <action>(CloseContent)
    let g:WhichKeyDesc_CloseOtherTabs = "<leader>xo CloseOtherTabs"
    nmap <leader>xo <action>(CloseAllEditorsButActive)
    let g:WhichKeyDesc_CloseAllTabsOnTheLeft = "<leader>x[ CloseAllTabsOnTheLeft"
    nmap <leader>x[ <action>(CloseAllToTheLeft)
    let g:WhichKeyDesc_CloseAllTabsOnTheRight = "<leader>x] CloseAllTabsOnTheRight"
    nmap <leader>x] <action>(CloseAllToTheRight)
    " Scroll page
    let g:WhichKeyDesc_EditorScrollUp = "<C-k> EditorScrollUp"
    nmap <C-k> <action>(EditorScrollUp)
    let g:WhichKeyDesc_EditorScrollDown = "<C-j> EditorScrollDown"
    nmap <C-j> <action>(EditorScrollDown)
    " Go to definition or reference
    let g:WhichKeyDesc_GotoDeclaration = "gd GotoDeclaration"
    nmap gd <action>(GotoDeclaration)
    " Go to usage
    let g:WhichKeyDesc_FindUsages = "<leader>gr FindUsages"
    nmap <leader>gr <action>(FindUsages)
    " Go to superclass
    let g:WhichKeyDesc_GotoSuperMethod = "<leader>gs GotoSuperMethod"
    nmap <leader>gs <action>(GotoSuperMethod)
    " Go to implementation
    let g:WhichKeyDesc_GotoImplementation = "<leader>gi GotoImplementation"
    nmap <leader>gi <action>(GotoImplementation)
    " Jump to method
    let g:WhichKeyDesc_MethodUp = "<M-k> MethodUp"
    nmap <M-k> <Action>(MethodUp)
    let g:WhichKeyDesc_MethodDown = "<M-j> MethodDown"
    nmap <M-j> <Action>(MethodDown)
    let g:WhichKeyDesc_ExtractMethod = "<leader>em ExtractMethod"
    vmap <leader>em <Action>(ExtractMethod)
    " Jump tab
    let g:WhichKeyDesc_PreviousTab = "<M-h> PreviousTab"
    nmap <M-h> <action>(PreviousTab)
    let g:WhichKeyDesc_NextTab = "<M-l> NextTab"
    nmap <M-l> <action>(NextTab)
    " Translation
    let g:WhichKeyDesc_EditorTranslate = "<leader>t EditorTranslate"
    vmap <leader>t <action>($EditorTranslateAction)
    " Cursor back
    let g:WhichKeyDesc_Back = "<C-i> Back"
    nmap <C-i> <action>(Back)
    " Cursor forward
    let g:WhichKeyDesc_Forward = "<C-o> Forward"
    nmap <C-o> <action>(Forward)
    " Open recent project
    let g:WhichKeyDesc_OpenRecentProject = "<leader>p OpenRecentProject"
    nmap <leader>p <action>($LRU)
    " Replace
    let g:WhichKeyDesc_ReplaceInFile = "<leader>rif ReplaceInFile"
    nmap <leader>rif <action>(Replace)
    vmap <leader>rif <action>(Replace)
    let g:WhichKeyDesc_ReplaceInProject = "<leader>rip ReplaceInProject"
    nmap <leader>rip <action>(ReplaceInPath)
    vmap <leader>rip <action>(ReplaceInPath)
    " Find
    let g:WhichKeyDesc_FindInFile = "<leader>fif FindInFile"
    nmap <leader>fif <action>(Find)
    vmap <leader>fif <action>(Find)
    let g:WhichKeyDesc_FindInProject = "<leader>fip FindInProject"
    nmap <leader>fip <action>(FindInPath)
    vmap <leader>fip <action>(FindInPath)
    " New line
    let g:WhichKeyDesc_NewLine = "<M-o> NewLine"
    nnoremap <M-o> :normal o<CR>
    " Toggle breakpoint
    let g:WhichKeyDesc_ToggleLineBreakpoint = "<leader>bb ToggleLineBreakpoint"
    nmap <leader>bb <action>(ToggleLineBreakpoint)
    " Show expression type
    let g:WhichKeyDesc_ExpressionTypeInfo = "<leader>et ExpressionTypeInfo"
    nmap <leader>et <action>(ExpressionTypeInfo)
    " Show method parameters
    let g:WhichKeyDesc_ParameterInfo = "<leader>et ParameterInfo"
    nmap <leader>pp <action>(ParameterInfo)
    " Recent files
    let g:WhichKeyDesc_RecentFiles = "<leader>ee RecentFiles"
    nmap <leader>ee <action>(RecentFiles)
    
    sethandler <C-j> a:vim i:ide
    sethandler <C-k> a:vim i:ide

总结
-------------------

这就是本文的全部内容。我希望这篇个人总结能帮助您更有效地使用 VIM。

再次感谢 Bram Moolenaar 为我们带来了如此出色的软件。R.I.P.

如果有什么错误或问题，请随时发表评论或私信我。谢谢。

参考
-----------------------

*   [https://yannesposito.com/Scratch/en/blog/Learn-Vim-Progressively/](https://yannesposito.com/Scratch/en/blog/Learn-Vim-Progressively/)
*   [https://medium.com/better-programming/50-vim-mode-tips-for-ide-users-f7b525a794b3](https://medium.com/better-programming/50-vim-mode-tips-for-ide-users-f7b525a794b3)
*   [https://github.com/justlorain/euphonium](https://github.com/justlorain/euphonium)
