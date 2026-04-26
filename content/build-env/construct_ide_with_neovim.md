+++
date = '2026-04-26T08:39:02+08:00'
draft = false
title = 'Neovim构建IDE'
+++


## 下载Neovim

***我们一般下载最新stable版本的neovim***

- 首先我们通过https://github.com/neovim/neovim地址进入到Neovim的github页面，找到右侧的Releases，如下图所示
![download_neovim_1](/home/phoon/Downloads/download_neovim_1.png)
- 点击releases，进入到Releases页面，按Ctrl+f，搜索"stable"关键字，找到最近的stable版本，如下图所示
![download_neovim_2](/home/phoon/Downloads/download_neovim_2.png)
- 下拉到该版本对应的Assets，单击Assets展开，找到nvim-linux-x86_64.appimage，点击下载，如下图所示
![download_neovim_3](/home/phoon/Downloads/download_neovim_3.png)

## 下载javascript运行时环境包

javascript运行时环境包的下载地址https://nodejs.org/en/，下载后得到node-v24.15.0-linux-x64.tar.xz（***这个是我下载的版本***）

## 预安装包

```bash
# ccls是一种c/c++语言服务，如果你主要开发c语言工程建议用ccls作为语言服务，如果主要开发c++工程则建议安装clangd作为语言服务（因为ccls对最新的c++语言标准支持不大好，而clangd属于llvm项目的一部分，能够更好的支持最新的c++标准）
sudo apt install make git ripgrep xclip ccls
```

## 配置环境变量
```bash
# 创建neovim目录
mkdir ~/neovim && cd neovim
# 将node-v24.15.0-linux-x64.tar.xz文件拷贝到该目录
cp /path/to/node-v24.15.0-linux-x64.tar.xz .
# 将nvim-linux-x86_64.appimage文件拷贝到该目录
cp /path/to/nvim-linux-x86_64.appimage .
# 解压node-v24.15.0-linux-x64.tar.xz文件
xz -d node-v24.15.0-linux-x64.tar.xz
tar -xvf node-v24.15.0-linux-x64.tar
# 给nvim-linux-x86_64.appimage文件添加可执行权限
chmod +x nvim-linux-x86_64.appimage
# 修改可执行文件名字为nvim
mv nvim-linux-x86_64.appimage nvim
# 将当前目录路径和node-v24.15.0-linux-x64/bin目录路径添加到可执行文件默认搜索路径
cd ~
vi .bashrc ---> 在最末尾添加export PATH=~/neovim:~/neovim/node-v24.15.0-linux-x64/bin:$PATH
```

## 构建neovim的IDE环境

- 下载neovim的插件管理器
```bash
git clone https://github.com/folke/lazy.nvim.git ~/.local/share/nvim/lazy/lazy.nvim
```
- 创建neovim配置文件，目前我们创建了三个配置文件，分别是init.lua、cmp.lua和lsp.lua，分别用于配置neovim的基本设置、nvim-cmp插件和LSP插件。

    - ***创建init.lua配置文件***
    ```lua
    -- 设置 runtimepath，确保 Lazy.nvim 被正确加载
    vim.opt.rtp:prepend("~/.local/share/nvim/lazy/lazy.nvim")
    
    -- 加载 Lazy.nvim
    require("lazy").setup({
        -- 这里是你的插件列表
        "neovim/nvim-lspconfig",            -- LSP配置
        "hrsh7th/nvim-cmp",                 -- 自动补全
        "hrsh7th/cmp-nvim-lsp",             -- LSP补全源
        "hrsh7th/cmp-buffer",               -- 缓冲区补全
        "hrsh7th/cmp-path",                 -- 文件路径补全
        "hrsh7th/cmp-cmdline",              -- 命令行补全
        "ray-x/lsp_signature.nvim",         -- 函数签名补全
        {
          "windwp/nvim-autopairs",
          event = "InsertEnter",
          config = function()
            require("nvim-autopairs").setup({})
          end,
        },                                  -- 括号自动补全插件
        {
          "kevinhwang91/nvim-bqf",
          ft = "qf",  -- 仅在 quickfix 文件类型下加载
          config = function()
            require("bqf").setup({
              auto_preview = true,  -- 启用自动预览
              func_map = {
                pscrolldown = '',
                pscrollup = '',
                open = 'o',
                openc = '<CR>',
              },
            })
          end,
        },                                  --  代码autopreview功能插件
        {
          "nvim-telescope/telescope.nvim",
          dependencies = { "nvim-lua/plenary.nvim" },
          config = function()
            require("telescope").setup({
              defaults = {
                layout_strategy = "vertical",
                layout_config = {
                  width = 0.98, -- 占据 98% 终端宽度
                  height = 0.98,   -- 占据 98% 终端高度
                  preview_cutoff = 0,  -- 让 preview 在所有窗口大小下都显示
                },
                sorting_strategy = "ascending",
                scroll_strategy = "limit",
                mappings = {
                  i = {
                    ["<C-j>"] = require('telescope.actions').move_selection_next,
                    ["<C-k>"] = require('telescope.actions').move_selection_previous,
                    -- 其他映射保持默认
                  },
                },
              }
            })
          end,
        },                                  -- 模糊查找插件
        {
          "dhananjaylatkar/cscope_maps.nvim",
          config = function()
            require("cscope_maps").setup({
              cscope = {
                db_file = "cscope.out",
                exec = "cscope",
                picker = "quickfix",        -- "也可以用telescope"
              },
            })
    
            -- 如果有cscope.out文件，则自动加载
            if vim.fn.filereadable("cscope.out") == 1 then
                vim.api.nvim_command("Cscope db add cscope.out")
            end
    
            -- 绑定快捷键
            vim.api.nvim_set_keymap("n", "<C-_>s", ":Cscope find s <C-R>=expand('<cword>')<CR><CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>g", ":Cscope find g <C-R>=expand('<cword>')<CR><CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>c", ":Cscope find c <C-R>=expand('<cword>')<CR><CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>t", ":Cscope find t <C-R>=expand('<cword>')<CR><CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>e", ":Cscope find e <C-R>=expand('<cword>')<CR><CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>f", ":Cscope find f <C-R>=expand('<cfile>')<CR><CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>i", ":Cscope find i ^<C-R>=expand('<cfile>')<CR>$<CR>", { noremap = true, silent = true } )
            vim.api.nvim_set_keymap("n", "<C-_>d", ":Cscope find d <C-R>=expand('<cword>')<CR><CR>", { noremap = true, silent = true } )
          end,
        },                                  -- 添加nvim的cscope支持
        {
          "iamcco/markdown-preview.nvim",
          cmd = { "MarkdownPreviewToggle", "MarkdownPreview", "MarkdownPreviewStop" },
          build = "cd app && npm install",
          init = function()
            vim.g.mkdp_filetypes = { "markdown" }
          end,
          ft = { "markdown" },
        },                                  -- 添加markdown preview支持
        {
          "github/copilot.vim",
          event = "InsertEnter", -- 仅在插入模式时加载，提高启动速度
          ft = { "c", "cpp", "cc", "h", "hpp", "rust" },
        },
        {
          "CopilotC-Nvim/CopilotChat.nvim",
          dependencies = {
            { "nvim-lua/plenary.nvim", branch = "master" },
          },
          build = "make tiktoken",
          opts = {
            -- See Configuration section for options
            window = {
              layout = 'horizontal',       -- 'vertical', 'horizontal', 'float'
              width = 0.5,              -- 50% of screen width
            },
          },
        },
    })
    
    vim.cmd("syntax on")      -- 启用语法高亮
    vim.cmd[[autocmd FileType rust set ts=2 sw=2 et]] -- 设置rust缩进为2个空格
    vim.opt.number = true     -- 显示行号
    vim.opt.tabstop = 2       -- 硬制表符宽度设为 2
    vim.opt.softtabstop = 2   -- 软制表符宽度设为 2
    vim.opt.shiftwidth = 2    -- 缩进宽度设为 2
    vim.opt.autoindent = true -- 启用自动缩进
    vim.opt.cindent = true    -- 启用 C 语言风格缩进
    vim.opt.expandtab = true  -- 将 Tab 替换为空格
    vim.opt.incsearch = true  -- 开启增量搜索
    vim.opt.hlsearch = true   -- 高亮搜索匹配
    vim.opt.laststatus = 0    -- 不显示状态栏
    
    -- 设置成vim主题
    vim.cmd("colorscheme vim")
    vim.api.nvim_set_hl(0, "Pmenu", { bg = "#808080", fg = "White" })
    
    -- 设置高亮颜色
    vim.cmd([[hi Visual term=reverse ctermbg=92 guibg=Purple]])
    
    -- 把ta命令映射为cs find g命令
    vim.cmd[[cnoreabbrev ta Cs find g]]
    
    -- telescope快捷键设置
    vim.api.nvim_set_keymap("n", "ff", "<cmd>Telescope find_files<CR>", { noremap = true, silent = true })
    vim.api.nvim_set_keymap("n", "fg", "<cmd>lua require('telescope.builtin').live_grep({ default_text = vim.fn.expand('<cword>') })<CR>", { noremap = true, silent = true })
    
    -- copilot快捷键设置
    -- vim.g.copilot_enabled = 0
    vim.g.copilot_no_tab_map = true
    vim.keymap.set('i', '<C-J>', 'copilot#Accept("\\<CR>")', {
      expr = true,
      replace_keycodes = false
    })
    vim.keymap.set('i', '<C-K>', '<Plug>(copilot-accept-line)')
    vim.keymap.set('i', '<C-L>', '<Plug>(copilot-accept-word)')
    
    -- 禁用回车后自动延续注释
    vim.api.nvim_create_autocmd("BufEnter", {
      pattern = "*",
      callback = function()
        vim.opt.formatoptions:remove({ "o", "r" })
      end,
    })
    
    -- quickfix窗口下按下ESC键关闭quickfix窗口
    vim.api.nvim_create_autocmd("FileType", {
      pattern = "qf",
      callback = function()
        -- 在 quickfix 窗口中，普通模式下按 ESC 执行 :cclose 命令关闭 quickfix 窗口
        vim.api.nvim_buf_set_keymap(0, "n", "<ESC>", ":cclose<CR>", { noremap = true, silent = true })
      end,
    })
    
    -- 禁用 LSP 诊断
    vim.diagnostic.enable(false)
    
    local npairs = require("nvim-autopairs")
    npairs.remove_rule("(") -- 禁用小括号
    npairs.remove_rule("[") -- 禁用中括号
    require("configs.cmp")  -- 加载 nvim-cmp 配置
    require("configs.lsp")  -- 加载 LSP 配置
    ```
    
    - ***创建cmp.lua配置文件***
    ```lua
    local cmp = require("cmp")
    
    cmp.setup({
      snippet = {
        expand = function(args)
          require("luasnip").lsp_expand(args.body) -- 使用 LuaSnip 处理代码片段
        end,
      },
      mapping = {
        -- `<Tab>` 选择下一个补全项
        ["<Tab>"] = cmp.mapping(function(fallback)
          if cmp.visible() then
            cmp.select_next_item()
          else
            fallback()
          end
        end, { "i", "s" }),
    
        -- `<S-Tab>` 选择上一个补全项
        ["<S-Tab>"] = cmp.mapping(function(fallback)
          if cmp.visible() then
            cmp.select_prev_item()
          else
            fallback()
          end
        end, { "i", "s" }),
    
        -- `<Space>` 确认补全
        ["<CR>"] = cmp.mapping.confirm({ select = true }),
    
        -- `<C-Space>` 手动触发补全
        ["<C-Space>"] = cmp.mapping.complete(),
      },
      sources = cmp.config.sources({
        { name = "nvim_lsp" },  -- LSP 提供的补全
        { name = "buffer" },    -- 从 buffer 补全
        { name = "path" },      -- 路径补全
        { name = "luasnip" },   -- 代码片段补全
      }),
    })
    
    -- `:` 命令补全
    cmp.setup.cmdline(":", {
      mapping = cmp.mapping.preset.cmdline(),
      sources = cmp.config.sources({
        { name = "path" },
        { name = "cmdline" },
      }),
    })
    
    -- `/` 和 `?` 搜索补全
    cmp.setup.cmdline({ "/", "?" }, {
      mapping = cmp.mapping.preset.cmdline(),
      sources = {
        { name = "buffer" }
      }
    })
    ```
    
    - ***创建lsp.lua配置文件***
    ```lua
    local lsp_signature = require("lsp_signature")
    local util = require("lspconfig.util")
    local capabilities = require("cmp_nvim_lsp").default_capabilities()
    
    -- 1. LspAttach 保持不变 (处理按键绑定)
    vim.api.nvim_create_autocmd("LspAttach", {
      group = vim.api.nvim_create_augroup("UserLspConfig", { clear = true }),
      callback = function(args)
        local bufnr = args.buf
        lsp_signature.on_attach({
          bind = true,
          hint_enable = false,
          floating_window = true,
          doc_lines = 10,
          handler_opts = { border = "rounded" },
        }, bufnr)
    
        local opts = { noremap = true, silent = true, buffer = bufnr }
        vim.keymap.set("n", "K", vim.lsp.buf.hover, opts)
        vim.keymap.set("n", "gd", vim.lsp.buf.definition, opts)
        vim.keymap.set("n", "gr", vim.lsp.buf.references, opts)
      end,
    })
    
    local function common_on_attach(client, bufnr)
      -- 禁用 clangd / rust-analyzer 的 semantic tokens
      if client.server_capabilities.semanticTokensProvider then
        client.server_capabilities.semanticTokensProvider = nil
      end
    end
    
    -- 2. 启动函数 (手动处理 root_dir)
    local function setup_server(server_name, config)
      -- 提取 root_dir 逻辑，不放入 vim.lsp.config (防止类型错误)
      local get_root = config.root_dir
      config.root_dir = nil 
    
      -- 注入 capabilities
      config.capabilities = vim.tbl_deep_extend("force", 
        config.capabilities or {}, 
        capabilities
      )
    
      -- 注册基础配置
      vim.lsp.config(server_name, config)
    
      -- 设置文件类型触发
      if config.filetypes then
        vim.api.nvim_create_autocmd("FileType", {
          pattern = config.filetypes,
          callback = function(args)
            local root_dir = nil
            if get_root then
              root_dir = get_root(args.file)
            else
              root_dir = vim.loop.cwd()
            end
    
            -- 如果找不到根目录，通常不启动或者以当前目录启动
            if not root_dir then return end
    
            -- 使用 vim.lsp.start 显式启动，传入计算好的 root_dir
            vim.lsp.start({
              name = server_name,
              cmd = config.cmd,
              root_dir = root_dir,
              init_options = config.init_options,
              capabilities = config.capabilities,
              offset_encoding = config.offset_encoding,
              on_attach = function(client, bufnr)
                common_on_attach(client, bufnr)
              end,
            }, { bufnr = args.buf })
          end,
        })
      end
    end
    
    -- 3. 配置ccls
    setup_server("ccls", {
      cmd = { "ccls" },
      offset_encoding = "utf-16",
      filetypes = { "c", "cc", "cpp", "c++", "objc", "objcpp" },
      root_dir = function(fname)
        return util.root_pattern(".ccls", "compile_commands.json", ".git")(fname) or vim.loop.cwd()
      end,
      init_options = {
        cache = {
          directory = "/tmp/ccls",
        },
      },
    })
    
    -- -- 3. 配置clangd，如果需要查看clangd的调试信息，加上以下两个参数
    -- -- a. --log=verbose:设置log等级为verbose
    -- -- b. --pretty:设置log输出更好看的日志格式
    -- -- 在/home/phoon/.local/state/nvim/lsp.log文件中查看clangd日志
    -- setup_server("clangd", {
    --   cmd = { 
    --     "clangd",
    --     "--background-index",
    --     "--pch-storage=memory",
    --     "--header-insertion=iwyu",
    --     "--completion-style=detailed",
    --     "--function-arg-placeholders",
    --     "--all-scopes-completion",
    --     "--completion-style=detailed",
    --     "--fallback-style=llvm",
    --     "--clang-tidy",
    -- 	},
    --   offset_encoding = "utf-16",
    --   filetypes = { "c", "cc", "cpp", "c++", "objc", "objcpp" },
    --   root_dir = function(fname)
    --     return util.root_pattern("compile_commands.json", ".git", "compile_flags.txt")(fname) or vim.loop.cwd()
    --   end,
    -- })
    
    -- 4. 配置rust_analyzer
    setup_server("rust_analyzer", {
      cmd = { "rust-analyzer" },
      offset_encoding = "utf-16",
      filetypes = { "rust" },
      root_dir = function(fname)
        return util.root_pattern("Cargo.toml", "compile_commands.json", ".git")(fname) or vim.loop.cwd()
      end,
    })
    ```
- 创建配置文件目录，并拷贝init.lua、cmp.lua和lsp.lua配置文件到相关目录下
```bash
mkdir -p ~/.config/nvim/lua/configs
cp /path/to/init.lua ~/.config/nvim/init.lua
cp /path/to/cmp.lua ~/.config/nvim/lua/configs/cmp.lua
cp /path/to/lsp.lua ~/.config/nvim/lua/configs/lsp.lua
```
- 安装插件
```bash
source ~/.bashrc
nvim ---> 等待自动安装完插件就可以用了
```
