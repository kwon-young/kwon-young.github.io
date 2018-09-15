---
layout: post
section-type: post
title: Vim Debugging
category: Vim
tags: [ 'Vim', 'debugging' ]
---

# Abstract

This blog post is the occasion for me to gather my thoughts about my debugging experience while coding which mainly takes place in Vim.
The goal is to define the problems of the current workflow and design possible solutions to these problems.

# Current Debugging Workflow

I mainly code in python, c and c++.
Therefore, I use pdb, gdb or lldb to debug the program that I write.
My current workflow is very simple and consists of opening a vertical split with a Vim or Neovim terminal buffer and run the debugged program in that buffer.
The other vertical split is used to browse the code while debugging.

![Current debugging workflow](/img/2018-09-15-vim-debugging/current_workflow.png)

## Pain Points

This workflow is globally acceptable with exactly zero cost of learning or maintaining a plugin.
However, the main pain point is the ability to jump from a stack trace in the terminal buffer to the correct location in the source code.
For example, if you take the previous image, I would like to be able to automatically jump the file `music_loc_classif_v7_test.py` at line 266.

The good Vimmer in me would say:

> Well, just use `gf` ...

... Until you can't.
First of all, this doesn't take into account the correct line number.
Secondly, wrapped lines in an Neovim terminal window are *not* soft wrapped but hard wrapped.
Meaning that if the file name is too long, you will have a broken file name.

I've often wondered if I could just make a plugin that would watch the terminal window and try to parse it's content in order to spot filenames and line numbers.
Note that this idea is inspired by the `compiler` feature in Vim, that allows use the run `make` and parse the output of the compiler into the quickfix window.
But again, the hard wrapped line in the terminal buffer is problematic and would require the plugin to detect which lines to *un* wrap, which could prove difficult.
Another way would be to run the debugger directly in Vim using the `channel` or `job` feature of Vim or Neovim.
You would then have to manually connect the stdin and stdout of the debugger to a Vim buffer.
It would also require to write a sort of `errorformat` for debugger to spot file names and line numbers, which would be very difficult because `errorformat` is designed for compiler output, while debugger output can be much more unconstrained (with user defined output using `print` or `printf`).

Finally, I miss the ability to do remote debugging.
While I don't often need it, it can be a life saver once in a while.

# Existing Vim Debugging Plugin

Well, debugging inside of Vim is not exactly a new idea.
I remember that the first time I searched for a debugging plugin, I found pyclewn, which I never was able to make it work.

## lldb.nvim

The only success story I had is [lldb.nvim](https://github.com/dbgx/lldb.nvim) which was a full-featured debugging plugin using lldb as a backend.
One thing that I really liked about this plugin was that the UI of the plugin was very solid.
If anything would go wrong, it would never leave Vim in a broken state and always tear down gracefully.
The UI organization was also full featured with a local variable, memory, expression, breakpoint window, all of that neatly organized around the code source and in different tabs.

Unfortunately, this plugin is discontinued because the python interface they used to communicate with lldb is not supported anymore.
I wish someone would port this plugin to use the new gdb-mi interface to make it work with both gdb and lldb.

## Vim termdebug plugin

Actually, Bram himself wrote a plugin using this very gdb-mi interface in a plugin called `termdebug` that is now shipped natively with Vim.
While the feature provided is still very basic, it works well for debugging c and c++ program with gdb.
It is also robust, meaning that the plugin will never let your Vim in a broken state (well, it's the least we could except for a plugin written by Bram himself).
One of the interesting feature of the plugin is that it remap `K` to actually evaluate the value of the variable under the cursor.
I'm actually in the process to port this plugin to Neovim, using its own `job` and `terminal` feature.

## nvim-gdb

Before Bram started to write his own debugging plugin, the original author of the fork Neovim, Thiago de Arruda wrote a little script in order to debug in Neovim.
This script turned out into a full plugin called [nvim-gdb](https://github.com/sakhnik/nvim-gdb).
This plugin actually implement most of the idea I describe in the [beginning of this post](#pain-points).
It is mainly a thin-wrapper around pdb, gdb and lldb using Neovim `job` feature and controls debugger using their standard input and output.

I think that this plugin is currently the best plugin we have to debug in Vim.
However, the fact that when I tried the plugin, I manage to break my Vim instance after only a few debugging sessions tells a lot about the stability of this approach.

## vimspector

[vimspector](https://github.com/puremourning/vimspector) is an experimental debugger plugin made by one of the developer of YCM.
It uses as a backend debugger plugins from Visual Studio Code using the Debug Adapter Protocol.
It is still very experimental and absolutely not ready for use.
