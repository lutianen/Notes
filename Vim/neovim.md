# Neovim (Nvim)

## Install neovim

```bash
sudo pacman -S neovim xsel
```

## 版本信息

```bash
~►nvim --version
NVIM v0.9.2
Build type: Release
LuaJIT 2.1.1694285958

   system vimrc file: "$VIM/sysinit.vim"
  fall-back for $VIM: "/usr/share/nvim"

Run :checkhealth for more info
```

## 常用命令

### 段落移动

在程序开发时，通常一段功能相关的代码会写在一起，即之间没有空行，这时可以用段落移动命令来快速移动光标。

```bash
{   # 移动到上一段的开头
}   # 移动到下一段的开头
```

### 括号切换

`%`: 括号匹配以及切换

### 单词快速匹配

`#`: 匹配光标所在单词，向前查找（快速查看这个单词在其他什么位置使用过）

### 文件操作

`:e`：edit，会打开内置的文件浏览器，浏览当下目录的文件
`:n filename`：new，新建文件
`:w filename`：write，保存文件

### 分屏

`:sp[filename]`：split，水平分屏
`:vsp[filename]`：vertical split，垂直分屏

## Nvim 配置

`Nvim` 的配置目录在 `~/.config/nvim` 下。

### 配置路径以文件

在 Linux/Mac 系统上，`Nvim` 会默认读取 `~/.config/nvim/init.lua` 文件，**理论上**来说可以将所有配置的东西都放在这个文件里面，但这样不是一个好的做法

```bash
mkdir ~/.config/nvim
mkdir ~/.config/nvim/lua
touch ~/.config/nvim/init.lua

# 目录结构
/home/tianen/.config/nvim/
├── init.lua
├── lua
│   ├── config
│   │   ├── bufferline.lua
│   │   ├── gitsigns.lua
│   │   ├── lualine.lua
│   │   ├── mason-null-ls.lua
│   │   ├── nvim-autopairs.lua
│   │   ├── nvim-cmp.lua
│   │   ├── nvim-telescope.lua
│   │   ├── nvim-tree.lua
│   │   ├── nvim-treesitter.lua
│   │   └── toggleterm.lua
│   ├── core
│   │   ├── colorscheme.lua
│   │   ├── keymaps.lua
│   │   └── options.lua
│   ├── lsp.lua
│   └── plugins.lua
└── plugin
    └── packer_compiled.lua
```

- `init.lua` 为 `Nvim` 配置的 Entry point，我们主要**用来导入其他 `*.lua` 文件**

  - `colorscheme.lua` 配置主题
  - `keymaps.lua` 配置按键映射
  - `lsp.lua` 配置 LSP
  - `options.lua` 配置选项
  - `plugins.lua` 配置插件

- `config` 用于**存放各种插件自身的配置**，文件名为插件的名字，这样比较好找

- `lua` 目录。当我们在 Lua 里面调用 `require` 加载模块（文件）的时候，它会自动在 `lua` 文件夹里面进行搜索

  - **将路径分隔符从 `/` 替换为 `.`，然后去掉 `.lua` 后缀就得到了 `require` 的参数格式**

    *比如要导入上面的 `nvim-cmp.lua` 文件，可以用 `require('config.nvim-cmp')`*

### 选项配置、按键配置

#### `init.lua`

```lua
require("core.options")
require("core.keymaps")
require("core.colorscheme")

require("plugins")
require("lsp")
```

#### `lsp.lua`

```lua
-- Note: The order matters: mason -> mason-lspconfig -> lspconfig
require('mason').setup({
    ui = {
        icons = {
            package_installed = "✓",
            package_pending = "➜",
            package_uninstalled = "✗"
        }
    }
})

require('mason-lspconfig').setup({
    -- A list of servers to automatically install if they're not already installed
    ensure_installed = { 'pylsp', 'gopls', 'lua_ls', 'bashls', 'rust_analyzer' },
})


-- Set different settings for different languages' LSP
-- LSP list: https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md
-- How to use setup({}): https://github.com/neovim/nvim-lspconfig/wiki/Understanding-setup-%7B%7D
--     - the settings table is sent to the LSP
--     - on_attach: a lua callback function to run after LSP attaches to a given buffer
local lspconfig = require('lspconfig')

-- Customized on_attach function
-- See `:help vim.diagnostic.*` for documentation on any of the below functions
local opts = { noremap = true, silent = true }
vim.keymap.set('n', '<space>e', vim.diagnostic.open_float, opts)
vim.keymap.set('n', '[d', vim.diagnostic.goto_prev, opts)
vim.keymap.set('n', ']d', vim.diagnostic.goto_next, opts)
vim.keymap.set('n', '<space>q', vim.diagnostic.setloclist, opts)

-- Use an on_attach function to only map the following keys
-- after the language server attaches to the current buffer
local on_attach = function(client, bufnr)
    -- Enable completion triggered by <c-x><c-o>
    vim.api.nvim_buf_set_option(bufnr, 'omnifunc', 'v:lua.vim.lsp.omnifunc')

    -- See `:help vim.lsp.*` for documentation on any of the below functions
    local bufopts = { noremap = true, silent = true, buffer = bufnr }
    vim.keymap.set('n', 'gD', vim.lsp.buf.declaration, bufopts)
    vim.keymap.set('n', 'gd', vim.lsp.buf.definition, bufopts)
    vim.keymap.set('n', 'K', vim.lsp.buf.hover, bufopts)
    vim.keymap.set('n', 'gi', vim.lsp.buf.implementation, bufopts)
    vim.keymap.set('n', '<C-k>', vim.lsp.buf.signature_help, bufopts)
    vim.keymap.set('n', '<space>wa', vim.lsp.buf.add_workspace_folder, bufopts)
    vim.keymap.set('n', '<space>wr', vim.lsp.buf.remove_workspace_folder, bufopts)
    vim.keymap.set('n', '<space>wl', function()
        print(vim.inspect(vim.lsp.buf.list_workspace_folders()))
    end, bufopts)
    vim.keymap.set('n', '<space>D', vim.lsp.buf.type_definition, bufopts)
    vim.keymap.set('n', '<space>rn', vim.lsp.buf.rename, bufopts)
    vim.keymap.set('n', '<space>ca', vim.lsp.buf.code_action, bufopts)
    vim.keymap.set('n', 'gr', vim.lsp.buf.references, bufopts)
    vim.keymap.set('n', '<space>f', function()
        vim.lsp.buf.format({
            async = true,
            filter = function (client)
                return client.name == "null-ls"
            end
        })
    end, bufopts)
end

-- Configure each language
-- How to add LSP for a specific language?
-- 1. use `:Mason` to install corresponding LSP
-- 2. add configuration below
lspconfig.pylsp.setup({
    on_attach = on_attach,
})

lspconfig.gopls.setup({
    on_attach = on_attach,
})

lspconfig.lua_ls.setup {
    on_attach = on_attach,
    settings = {
        Lua = {
            runtime = {
                -- Tell the language server which version of Lua you're using (most likely LuaJIT in the case of Neovim)
                version = 'LuaJIT',
            },
            diagnostics = {
                -- Get the language server to recognize the `vim` global
                globals = { 'vim' },
            },
            workspace = {
                -- Make the server aware of Neovim runtime files
                library = vim.api.nvim_get_runtime_file("", true),
            },
            -- Do not send telemetry data containing a randomized but unique identifier
            telemetry = {
                enable = false,
            },
        },
    },
}

lspconfig.bashls.setup({})

-- source: https://rust-analyzer.github.io/manual.html#nvim-lsp
lspconfig.rust_analyzer.setup({
    on_attach = on_attach,
    settings = {
        ['rust-analyzer'] = {
            inlayHints = {
                closingBraceHints = true, -- Whether to show inlay hints after a closing } to indicate what item it belongs to.
            }
        }
    }
})

lspconfig.clangd.setup({
    on_attach = on_attach,
})
```

#### `plugins.lua`

```lua
-----------------
-- Packer.nvim --
-----------------
-- Install Packer automatically if it's not installed(Bootstraping)
-- Hint: string concatenation is done by `..`
local ensure_packer = function()
	local fn = vim.fn
	local install_path = fn.stdpath("data") .. "/site/pack/packer/start/packer.nvim"
	if fn.empty(fn.glob(install_path)) > 0 then
		fn.system({ "git", "clone", "--depth", "1", "https://github.com/wbthomason/packer.nvim", install_path })
		vim.cmd([[packadd packer.nvim]])
		return true
	end
	return false
end
local packer_bootstrap = ensure_packer()

-- Reload configurations if we modify plugins.lua
-- Hint
--     <afile> - replaced with the filename of the buffer being manipulated
vim.cmd([[
  augroup packer_user_config
    autocmd!
    autocmd BufWritePost plugins.lua source <afile> | PackerSync
  augroup end
]])

-- Install plugins here - `use ...`
-- Packer.nvim hints
--     after = string or list,           -- Specifies plugins to load before this plugin. See "sequencing" below
--     config = string or function,      -- Specifies code to run after this plugin is loaded
--     requires = string or list,        -- Specifies plugin dependencies. See "dependencies".
--     ft = string or list,              -- Specifies filetypes which load this plugin.
--     run = string, function, or table, -- Specify operations to be run after successful installs/updates of a plugin
return require("packer").startup(function(use)
	-- Packer can manage itself
	use("wbthomason/packer.nvim")

	---------------------------------------
	-- NOTE: PUT YOUR THIRD PLUGIN HERE --
	---------------------------------------

	-- Colorschem
	use("tanvirtin/monokai.nvim")

	-- Status line
	use({
		"nvim-lualine/lualine.nvim",
		requires = { 'nvim-tree/nvim-web-devicons', opt = true },
		-- requires = { "kyazdani42/nvim-web-devicons", opt = true },
		config = [[require('config.lualine')]],
	})

	-- File explorer
	use({
		"nvim-tree/nvim-tree.lua",
		requires = {
			"nvim-tree/nvim-web-devicons", -- optional
		},
		config = [[require('config.nvim-tree')]],
	})
	use("christoomey/vim-tmux-navigator") -- Ctrl - hjkl 定位窗口

	-- Treesitter
	use({
		"nvim-treesitter/nvim-treesitter",
		run = function()
			local ts_update = require("nvim-treesitter.install").update({ with_sync = true })
			ts_update()
		end,
		config = [[require('config.nvim-treesitter')]],
	})

	-- rainbow
	use("p00f/nvim-ts-rainbow")

	-- LSP manager
	use({ "williamboman/mason.nvim" })
	use({ "williamboman/mason-lspconfig.nvim" })
	use({ "neovim/nvim-lspconfig" })

	-- Add hooks to LSP to support Linter && Formatter
	use({ "nvim-lua/plenary.nvim" })
	use({
		"jay-babu/mason-null-ls.nvim",
		after = "plenary.nvim",
		requires = { "jose-elias-alvarez/null-ls.nvim" },
		config = [[require('config.mason-null-ls')]],
	})

	-- Vscode-like pictograms
	use({ "onsails/lspkind.nvim", event = "VimEnter" })

	-- Auto-completion engine
	-- Note:
	--     the default search path for `require` is ~/.config/nvim/lua
	--     use a `.` as a path seperator
	--     the suffix `.lua` is not needed
	use({ "hrsh7th/nvim-cmp", after = "lspkind.nvim", config = [[require('config.nvim-cmp')]] })
	use({ "hrsh7th/cmp-nvim-lsp", after = "nvim-cmp" })
	use({ "hrsh7th/cmp-buffer", after = "nvim-cmp" }) -- buffer auto-completion
	use({ "hrsh7th/cmp-path", after = "nvim-cmp" }) -- path auto-completion
	use({ "hrsh7th/cmp-cmdline", after = "nvim-cmp" }) -- cmdline auto-completion

	-- Code snippet engine
	use("L3MON4D3/LuaSnip")
	use({ "saadparwaiz1/cmp_luasnip", after = { "nvim-cmp", "LuaSnip" } })

	-- Autopairs: [], (), "", '', etc
	-- it relies on nvim-cmp
	use({
		"windwp/nvim-autopairs",
		after = "nvim-cmp",
		config = [[require('config.nvim-autopairs')]],
	})

	-- Code comment helper
	--     1. `gcc` to comment a line
	--     2. select lines in visual mode and run `gc` to comment/uncomment lines
	use("tpope/vim-commentary")

	-- Buffer
	use({
		"akinsho/bufferline.nvim",
		config = [[require('config.bufferline')]],
	})

	-- Git integration
	use("tpope/vim-fugitive")
	use({ "lewis6991/gitsigns.nvim", config = [[require('config.gitsigns')]] })

	-- Markdown support
	use({ "preservim/vim-markdown", ft = { "markdown" } })

	-- Markdown previewer
	-- It require nodejs and yarn. Use homebrew to install first
	use({
		"iamcco/markdown-preview.nvim",
		run = "cd app && npm install",
		setup = function()
			vim.g.mkdp_filetypes = { "markdown" }
		end,
		ft = { "markdown" },
	})

	-- Smart motion
	use({
		"phaazon/hop.nvim",
		branch = "v2", -- optional but strongly recommended
		config = function()
			-- you can configure Hop the way you like here; see :h hop-config
			require("hop").setup({ keys = "etovxqpdygfblzhckisuran" })
		end,
	})

	-- Better terminal integration
	-- tag = string,
	-- Specifies a git tag to use. Supports '*' for "latest tag"
	use({
		"akinsho/toggleterm.nvim",
		tag = "*",
		config = [[require('config.toggleterm')]],
	})

	-- telescope
	use({
		"nvim-telescope/telescope.nvim",
		tag = "0.1.3",
		requires = { { "nvim-lua/plenary.nvim" } },
		config = [[require('config.nvim-telescope')]],
	})

	-- Automatically set up your configuration after cloning packer.nvim
	-- Put this at the end after all plugins
	if packer_bootstrap then
		require("packer").sync()
	end
end)
```

#### `options.lua`

```bash
-- Hint: use `:h <option>` to figure out the meaning if needed
vim.opt.clipboard = 'unnamedplus' -- use system clipboard
vim.opt.completeopt = { 'menu', 'menuone', 'noselect' }
vim.opt.mouse = 'a' -- allow the mouse to be used in Nvim

-- Tab
vim.opt.tabstop = 2 -- number of visual spaces per TAB
vim.opt.softtabstop = 2 -- number of spacesin tab when editing
vim.opt.shiftwidth = 2 -- insert 4 spaces on a tab
vim.opt.expandtab = true -- tabs are spaces, mainly because of python

-- UI config
vim.opt.number = true -- show absolute number
vim.opt.relativenumber = true -- add numbers to each line on the left side
vim.opt.cursorline = true -- highlight cursor line underneath the cursor horizontally
vim.opt.splitbelow = true -- open new vertical split bottom
vim.opt.splitright = true -- open new horizontal splits right
vim.opt.termguicolors = true        -- enabl 24-bit RGB color in the TUI
vim.opt.signcolumn = "yes"
vim.opt.showmode = false -- we are experienced, wo don't need the "-- INSERT --" mode hint

vim.cmd[[colorscheme monokai_pro]]

-- Searching
vim.opt.incsearch = true -- search as characters are entered
vim.opt.hlsearch = false -- do not highlight matches
vim.opt.ignorecase = true -- ignore case in searches by default
vim.opt.smartcase = true -- but make it case sensitive if an uppercase is entered

-- word wrap
vim.opt.wrap = true
```

- 默认采用系统剪贴板，同时支持鼠标操控 `Nvim`
- Tab 和空格的换算
- UI 界面
- “智能”搜索

#### `keymaps.lua`

```lua
vim.g.mapleader = " "

local keymap = vim.keymap
local opts = {
	noremap = true, -- non-recursive
	silent = true, -- do not show message
}

----------------------------------
-- Insert mode --
----------------------------------
keymap.set("i", "jk", "<ESC>")

----------------------------------
-- Normal mode --
----------------------------------
-- New windows
keymap.set("n", "<leader>sv", "<C-w>v") -- 水平新增窗口
keymap.set("n", "<leader>sh", "<C-w>s") -- 垂直新增窗口

--
-- Hint: see `:h vim.map.set()`
-- Better window navigation
--keymap.set('n', '<C-h>', '<C-w>h', opts)
--keymap.set('n', '<C-j>', '<C-w>j', opts)
--keymap.set('n', '<C-k>', '<C-w>k', opts)
--keymap.set('n', '<C-l>', '<C-w>l', opts)

-- Resize with arrows
-- delta: 2 lines
keymap.set("n", "<C-Up>", ":resize -2<CR>", opts)
keymap.set("n", "<C-Down>", ":resize +2<CR>", opts)
keymap.set("n", "<C-Left>", ":vertical resize -2<CR>", opts)
keymap.set("n", "<C-Right>", ":vertical resize +2<CR>", opts)

-- 文件
keymap.set("n", "<leader>w", ":w!<CR>") -- Save file
keymap.set("n", "zz", ":q<CR>") -- Quit file

-- 取消高亮
keymap.set("n", "<leader>nh", ":nohl<CR>")

-- 光标快速移动
keymap.set("n", "H", "^") -- 移动光标至行首
keymap.set("n", "L", "$") -- 移动光标至行尾

-- nvim-tree
keymap.set("n", "<leader>ef", ":NvimTreeToggle<CR>")

-- buffer line
keymap.set("n", "<leader>l", ":bnext<CR>")
keymap.set("n", "<leader>h", ":bprevious<CR>")

----------------------------------
-- Visual mode --
----------------------------------
-- Hint: start visual mode with the same area as the previous area and the same mode
keymap.set("v", "<", "<gv", opts)
keymap.set("v", ">", ">gv", opts)

-- 单行或多行移动
keymap.set("v", "J", ":m '>+1<CR>gv=gv")
keymap.set("v", "K", ":m '<-2<CR>gv=gv")
```

- 用 `<C-h/j/k/l>` 快速在多窗口之间移动光标
- 用 `Ctrl` + 方向键进行窗口大小的调整
- 选择模式下可以一直用 `Tab` 或者 `Shift-Tab` 改变缩进

#### `colorscheme.lua`

```lua
-- define your colorscheme here
local colorscheme = 'monokai_pro'

local is_ok, _ = pcall(vim.cmd, "colorscheme " .. colorscheme)
if not is_ok then
    vim.notify('colorscheme ' .. colorscheme .. ' not found!')
    return
end
```

#### `bufferline.lua`

```lua
vim.opt.termguicolors = true

require("bufferline").setup{
    options = {
      diagnostics = "nvim_lsp",

      offsets = {
        {
            filetype = "NvimTree",
            text = "File Explorer",
            highlight = "Directory",
            text_align = "left"
        },
      },
  }
}

vim.cmd [[
  aug buffer_accessed_time
    au!
    au BufEnter,BufWinEnter * let b:accessedtime = localtime()
  aug END

  function! BufferLineSortByMRU()
    lua require'bufferline'.sort_buffers_by(function(a, b) return (vim.b[a.id].accessedtime or 0) > (vim.b[b.id].accessedtime or 0) end)
  endfunction

  command -nargs=0 BufferLineSortByMRU call BufferLineSortByMRU()
]]
```

#### `gitsigns.lua`

```lua
local is_ok, gitsigns = pcall(require, 'gitsigns')
if not is_ok then
    return
end

gitsigns.setup {
    signs                        = {
        add          = { text = '│' },
        change       = { text = '│' },
        delete       = { text = '_' },
        topdelete    = { text = '‾' },
        changedelete = { text = '~' },
        untracked    = { text = '┆' },
    },
    signcolumn                   = true, -- Toggle with `:Gitsigns toggle_signs`
    numhl                        = false, -- Toggle with `:Gitsigns toggle_numhl`
    linehl                       = false, -- Toggle with `:Gitsigns toggle_linehl`
    word_diff                    = false, -- Toggle with `:Gitsigns toggle_word_diff`
    watch_gitdir                 = {
        interval = 1000,
        follow_files = true
    },
    attach_to_untracked          = true,
    current_line_blame           = false, -- Toggle with `:Gitsigns toggle_current_line_blame`
    current_line_blame_opts      = {
        virt_text = true,
        virt_text_pos = 'eol', -- 'eol' | 'overlay' | 'right_align'
        delay = 1000,
        ignore_whitespace = false,
    },
    current_line_blame_formatter = '<author>, <author_time:%Y-%m-%d> - <summary>',
    sign_priority                = 6,
    update_debounce              = 100,
    status_formatter             = nil, -- Use default
    max_file_length              = 40000, -- Disable if file is longer than this (in lines)
    preview_config               = {
        -- Options passed to nvim_open_win
        border = 'single',
        style = 'minimal',
        relative = 'cursor',
        row = 0,
        col = 1
    },
    yadm                         = {
        enable = false
    },
}
```

#### `lualine.lua`

```lua
local is_ok, lualine = pcall(require, 'lualine')
if not is_ok then
    return
end

lualine.setup {
    options = {
        icons_enabled = true,
        theme = 'material',
        component_separators = { left = '', right = '' },
        section_separators = { left = '', right = '' },
        disabled_filetypes = {
            statusline = {},
            winbar = {},
        },
        ignore_focus = {},
        always_divide_middle = true,
        globalstatus = false,
        refresh = {
            statusline = 1000,
            tabline = 1000,
            winbar = 1000,
        }
    },
    -- +-------------------------------------------------+
    -- | A | B | C                             X | Y | Z |
    -- +-------------------------------------------------+
    sections = {
        lualine_a = { 'mode' },
        lualine_b = { 'branch', 'diff', 'diagnostics' },
        lualine_c = { 'filename' },
        lualine_x = { 'encoding', 'fileformat', 'filetype' },
        lualine_y = { 'progress' },
        lualine_z = { 'location' }
    },
    inactive_sections = {
        lualine_a = {},
        lualine_b = {},
        lualine_c = { 'filename' },
        lualine_x = { 'location' },
        lualine_y = {},
        lualine_z = {}
    },
    tabline = {},
    winbar = {},
    inactive_winbar = {},
    extensions = {}
}
```

#### `maso-null-ls.lua`

```lua
local mason_ok, mason = pcall(require, "mason")
if not mason_ok then
	return
end

mason.setup()

local null_ls_ok, null_ls = pcall(require, "null-ls")
if not null_ls_ok then
	return
end

local sources = {
	null_ls.builtins.formatting.black.with({ extra_args = { "--target-version", "py310" } }),
	null_ls.builtins.formatting.stylua,
}

null_ls.setup({
	debug = false,
	log_level = "warn",
	update_in_insert = false,
	sources = sources,
})

local mason_null_ls_ok, mason_null_ls = pcall(require, "mason-null-ls")
if not mason_null_ls_ok then
	return
end

mason_null_ls.setup({
	-- A list of sources to install if they're not already installed.
	-- This setting has no relation with the `automatic_installation` setting.
	ensure_installed = {
		"black",
		"stylua",
	},
	automatic_installation = false,
	-- Sources found installed in mason will automatically be setup for null-ls.
	automatic_setup = true,
	handlers = {},
})
```

#### `nvim-autopairs.lua`

```lua
local is_ok, npairs = pcall(require, 'nvim-autopairs')
if not is_ok then
    return
end

npairs.setup({
    check_ts = true, -- check if treesitter is installed
    disable_filetype = { "TelescopePrompt", "vim" },
    ts_config = {
        lua = { 'string' }, -- it will not add a pair on that treesitter node
        javascript = { 'template_string' },
        java = false, -- don't check treesitter on java
    }
})

-- If you want insert `(` after select function or method item
local cmp_autopairs = require('nvim-autopairs.completion.cmp')
local cmp = require('cmp')
cmp.event:on(
    'confirm_done',
    cmp_autopairs.on_confirm_done()
)
```

#### `nvim-cmp.lua`

```lua
local luasnip_ok, luasnip = pcall(require, "luasnip")
local cmp_ok, cmp = pcall(require, "cmp")
local lspkind_ok, lspkind = pcall(require, "lspkind")

if not luasnip_ok or not cmp_ok or not lspkind_ok then
	return
end

local has_words_before = function()
	unpack = unpack or table.unpack
	local line, col = unpack(vim.api.nvim_win_get_cursor(0))
	return col ~= 0 and vim.api.nvim_buf_get_lines(0, line - 1, line, true)[1]:sub(col, col):match("%s") == nil
end

cmp.setup({
	snippet = {
		-- REQUIRED - you must specify a snippet engine
		expand = function(args)
			require("luasnip").lsp_expand(args.body) -- For `luasnip` users.
		end,
	},
	mapping = cmp.mapping.preset.insert({
		-- Use <C-b/f> to scroll the docs
		["<C-b>"] = cmp.mapping.scroll_docs(-4),
		["<C-f>"] = cmp.mapping.scroll_docs(4),
		-- Use <C-k/j> to switch in items
		["<C-k>"] = cmp.mapping.select_prev_item(),
		["<C-j>"] = cmp.mapping.select_next_item(),
		-- Use <CR>(Enter) to confirm selection
		-- Accept currently selected item. Set `select` to `false` to only confirm explicitly selected items.
		["<CR>"] = cmp.mapping.confirm({ select = true }),

		-- A super tab
		-- sourc: https://github.com/hrsh7th/nvim-cmp/wiki/Example-mappings#luasnip
		["<Tab>"] = cmp.mapping(function(fallback)
			-- Hint: if the completion menu is visible select next one
			if cmp.visible() then
				cmp.select_next_item()
			elseif luasnip.expand_or_locally_jumpable() then
				-- You could replace the expand_or_jumpable() calls with expand_or_locally_jumpable()
				-- they way you will only jump inside the snippet region
				luasnip.expand_or_jump()
			elseif has_words_before() then
				cmp.complete()
			else
				fallback()
			end
		end, { "i", "s" }), -- i - insert mode; s - select mode
		["<S-Tab>"] = cmp.mapping(function(fallback)
			if cmp.visible() then
				cmp.select_prev_item()
			elseif luasnip.jumpable(-1) then
				luasnip.jump(-1)
			else
				fallback()
			end
		end, { "i", "s" }),
	}),
	-- Let's configure the item's appearance
	-- source: https://github.com/hrsh7th/nvim-cmp/wiki/Menu-Appearance
	formatting = {
		-- customize the appearance of the completion menu
		format = lspkind.cmp_format({
			-- show only symbol annotations
			mode = "symbol_text",
			-- prevent the popup from showing more than provided characters (e.g 50 will not show more than 50 characters)
			maxwidth = 100,
			-- when popup menu exceed maxwidth, the truncated part would show ellipsis_char instead (must define maxwidth first)
			ellipsis_char = "...",

			-- The function below will be called before any actual modifications from lspkind
			-- so that you can provide more controls on popup customization. (See [#30](https://github.com/onsails/lspkind-nvim/pull/30))
			before = function(entry, vim_item)
				vim_item.menu = ({
					nvim_lsp = "[Lsp]",
					luasnip = "[Luasnip]",
					buffer = "[File]",
					path = "[Path]",
				})[entry.source.name]
				return vim_item
			end,
		}),
	},
	-- Set source precedence
	sources = cmp.config.sources({
		{ name = "nvim_lsp" }, -- For nvim-lsp
		{ name = "luasnip" }, -- For luasnip user
		{ name = "buffer" }, -- For buffer word completion
		{ name = "path" }, -- For path completion
	}),
})

-- Set configuration for specific filetype.
cmp.setup.filetype("gitcommit", {
	sources = cmp.config.sources({
		{ name = "cmp_git" }, -- You can specify the `cmp_git` source if you were installed it.
	}, {
		{ name = "buffer" },
	}),
})

-- Use buffer source for `/` and `?` (if you enabled `native_menu`, this won't work anymore).
cmp.setup.cmdline({ "/", "?" }, {
	mapping = cmp.mapping.preset.cmdline(),
	sources = {
		{ name = "buffer" },
	},
})

-- Use cmdline & path source for ':' (if you enabled `native_menu`, this won't work anymore).
cmp.setup.cmdline(":", {
	mapping = cmp.mapping.preset.cmdline(),
	sources = cmp.config.sources({
		{ name = "path" },
	}, {
		{ name = "cmdline" },
	}),
})
```

#### `nvim-telescope.lua`

```lua
local builtin = require('telescope.builtin')

vim.keymap.set('n', '<leader>ff', builtin.find_files, {})
vim.keymap.set('n', '<leader>fg', builtin.live_grep, {})
vim.keymap.set('n', '<leader>fb', builtin.buffers, {})
vim.keymap.set('n', '<leader>fh', builtin.help_tags, {})
```

#### `nvim-tree.lua`

```lua
local is_ok, nvim_tree = pcall(require, "nvim-tree")
if not is_ok then
	return
end

--
-- This function has been generated from your
--   view.mappings.list
--   view.mappings.custom_only
--   remove_keymaps
--
-- You should add this function to your configuration and set on_attach = on_attach in the nvim-tree setup call.
--
-- Although care was taken to ensure correctness and completeness, your review is required.
--
-- Please check for the following issues in auto generated content:
--   "Mappings removed" is as you expect
--   "Mappings migrated" are correct
--
-- Please see https://github.com/nvim-tree/nvim-tree.lua/wiki/Migrating-To-on_attach for assistance in migrating.
--

local function on_attach(bufnr)
	local api = require("nvim-tree.api")

	local function opts(desc)
		return { desc = "nvim-tree: " .. desc, buffer = bufnr, noremap = true, silent = true, nowait = true }
	end

	-- Default mappings. Feel free to modify or remove as you wish.
	--
	-- BEGIN_DEFAULT_ON_ATTACH
	vim.keymap.set("n", "<C-]>", api.tree.change_root_to_node, opts("CD"))
	vim.keymap.set("n", "<C-e>", api.node.open.replace_tree_buffer, opts("Open: In Place"))
	vim.keymap.set("n", "<C-k>", api.node.show_info_popup, opts("Info"))
	vim.keymap.set("n", "<C-r>", api.fs.rename_sub, opts("Rename: Omit Filename"))
	vim.keymap.set("n", "<C-t>", api.node.open.tab, opts("Open: New Tab"))
	vim.keymap.set("n", "<C-v>", api.node.open.vertical, opts("Open: Vertical Split"))
	vim.keymap.set("n", "<C-x>", api.node.open.horizontal, opts("Open: Horizontal Split"))
	vim.keymap.set("n", "<BS>", api.node.navigate.parent_close, opts("Close Directory"))
	vim.keymap.set("n", "<CR>", api.node.open.edit, opts("Open"))
	vim.keymap.set("n", "<Tab>", api.node.open.preview, opts("Open Preview"))
	vim.keymap.set("n", ">", api.node.navigate.sibling.next, opts("Next Sibling"))
	vim.keymap.set("n", "<", api.node.navigate.sibling.prev, opts("Previous Sibling"))
	vim.keymap.set("n", ".", api.node.run.cmd, opts("Run Command"))
	vim.keymap.set("n", "-", api.tree.change_root_to_parent, opts("Up"))
	vim.keymap.set("n", "a", api.fs.create, opts("Create"))
	vim.keymap.set("n", "bmv", api.marks.bulk.move, opts("Move Bookmarked"))
	vim.keymap.set("n", "B", api.tree.toggle_no_buffer_filter, opts("Toggle No Buffer"))
	vim.keymap.set("n", "c", api.fs.copy.node, opts("Copy"))
	vim.keymap.set("n", "C", api.tree.toggle_git_clean_filter, opts("Toggle Git Clean"))
	vim.keymap.set("n", "[c", api.node.navigate.git.prev, opts("Prev Git"))
	vim.keymap.set("n", "]c", api.node.navigate.git.next, opts("Next Git"))
	vim.keymap.set("n", "d", api.fs.remove, opts("Delete"))
	vim.keymap.set("n", "D", api.fs.trash, opts("Trash"))
	vim.keymap.set("n", "E", api.tree.expand_all, opts("Expand All"))
	vim.keymap.set("n", "e", api.fs.rename_basename, opts("Rename: Basename"))
	vim.keymap.set("n", "]e", api.node.navigate.diagnostics.next, opts("Next Diagnostic"))
	vim.keymap.set("n", "[e", api.node.navigate.diagnostics.prev, opts("Prev Diagnostic"))
	vim.keymap.set("n", "F", api.live_filter.clear, opts("Clean Filter"))
	vim.keymap.set("n", "f", api.live_filter.start, opts("Filter"))
	vim.keymap.set("n", "g?", api.tree.toggle_help, opts("Help"))
	vim.keymap.set("n", "gy", api.fs.copy.absolute_path, opts("Copy Absolute Path"))
	vim.keymap.set("n", "H", api.tree.toggle_hidden_filter, opts("Toggle Dotfiles"))
	vim.keymap.set("n", "I", api.tree.toggle_gitignore_filter, opts("Toggle Git Ignore"))
	vim.keymap.set("n", "J", api.node.navigate.sibling.last, opts("Last Sibling"))
	vim.keymap.set("n", "K", api.node.navigate.sibling.first, opts("First Sibling"))
	vim.keymap.set("n", "m", api.marks.toggle, opts("Toggle Bookmark"))
	vim.keymap.set("n", "o", api.node.open.edit, opts("Open"))
	vim.keymap.set("n", "O", api.node.open.no_window_picker, opts("Open: No Window Picker"))
	vim.keymap.set("n", "p", api.fs.paste, opts("Paste"))
	vim.keymap.set("n", "P", api.node.navigate.parent, opts("Parent Directory"))
	vim.keymap.set("n", "q", api.tree.close, opts("Close"))
	vim.keymap.set("n", "r", api.fs.rename, opts("Rename"))
	vim.keymap.set("n", "R", api.tree.reload, opts("Refresh"))
	vim.keymap.set("n", "s", api.node.run.system, opts("Run System"))
	vim.keymap.set("n", "S", api.tree.search_node, opts("Search"))
	vim.keymap.set("n", "U", api.tree.toggle_custom_filter, opts("Toggle Hidden"))
	vim.keymap.set("n", "W", api.tree.collapse_all, opts("Collapse"))
	vim.keymap.set("n", "x", api.fs.cut, opts("Cut"))
	vim.keymap.set("n", "y", api.fs.copy.filename, opts("Copy Name"))
	vim.keymap.set("n", "Y", api.fs.copy.relative_path, opts("Copy Relative Path"))
	vim.keymap.set("n", "<2-LeftMouse>", api.node.open.edit, opts("Open"))
	vim.keymap.set("n", "<2-RightMouse>", api.tree.change_root_to_node, opts("CD"))
	-- END_DEFAULT_ON_ATTACH

	-- Mappings migrated from view.mappings.list
	--
	-- You will need to insert "your code goes here" for any mappings with a custom action_cb
	vim.keymap.set("n", "u", api.tree.change_root_to_parent, opts("Up"))
	vim.keymap.set("n", "o", api.node.open.edit, opts("Open"))
	vim.keymap.set("n", "h", api.node.navigate.parent_close, opts("Close Directory"))
	vim.keymap.set("n", "v", api.node.open.vertical, opts("Open: Vertical Split"))
end

-- Hint: :help nvim-tree-default-mappings
-- setup with some options
nvim_tree.setup({
	sort_by = "case_sensitive",
	on_attach = on_attach,
	renderer = {
		group_empty = true,
	},
	filters = {
		dotfiles = true,
	},
	diagnostics = {
		enable = true,
	},
})
```

#### `nvim-treesitter.lua`

```lua
local is_ok, configs = pcall(require, 'nvim-treesitter.configs')
if not is_ok then
    return
end

require'nvim-treesitter.configs'.setup({
    -- A list of parser names, or "all" (the four listed parsers should always be installed)
    ensure_installed = { 'c', 'cpp', 'python', "cmake", 'lua', 'vim', 'markdown', 'yaml', 'make', 'json', 'dockerfile', 'gomod', 'go','comment' },

    -- Install parsers synchronously (only applied to `ensure_installed`)
    sync_install = false,
    -- Automatically install missing parsers when entering buffer
    -- Recommendation: set to false if you don't have `tree-sitter` CLI installed locally
    auto_install = true,
    -- List of parsers to ignore installing (for "all")
    ignore_install = { "javascript" },
    ---- If you need to change the installation directory of the parsers (see -> Advanced Setup)
    -- parser_install_dir = "/some/path/to/store/parsers", -- Remember to run vim.opt.runtimepath:append("/some/path/to/store/parsers")!

    highlight = {
        -- When `enable` is `true` this will enable the module for all supported languages,
        -- `false` will disable the whole extension
        enable = true,

        -- NOTE: these are the names of the parsers and not the filetype. (for example if you want to
        -- disable highlighting for the `tex` filetype, you need to include `latex` in this list as this is
        -- the name of the parser)
        -- if you want to disable the module for some languages you can pass a list to the `disable` option.
        disable = { "c", "rust" },
        -- Or use a function for more flexibility, e.g. to disable slow treesitter highlight for large files
        -- disable = function(lang, buf)
        --     local max_filesize = 100 * 1024 -- 100 KB
        --     local ok, stats = pcall(vim.loop.fs_stat, vim.api.nvim_buf_get_name(buf))
        --     if ok and stats and stats.size > max_filesize then
        --         return true
        --     end
        -- end,

        -- Setting this to true will run `:h syntax` and tree-sitter at the same time.
        -- Set this to `true` if you depend on 'syntax' being enabled (like for indentation).
        -- Using this option may slow down your editor, and you may see some duplicate highlights.
        -- Instead of true it can also be a list of languages
        additional_vim_regex_highlighting = false,
    },
    -- Indentation based on treesitter for the = operator. NOTE: This is an experimental feature.
    -- indent = {
    --     enable = true
    -- },
    incremental_selection = {
        enable = true,
        -- init_selection: in normal mode, start incremental selection.
        -- node_incremental: in visual mode, increment to the upper named parent.
        -- scope_incremental: in visual mode, increment to the upper scope
        -- node_decremental: in visual mode, decrement to the previous named node.
        keymaps = {
            init_selection = "gnn",
            node_incremental = "grn",
            scope_incremental = "grc",
            node_decremental = "grm",
        },
    },
})

-- Hints:
--   A uppercase letter followed `z` means recursive
--   zo: open one fold under the cursor
--   zc: close one fold under the cursor
--   za: toggle the folding
--   zv: open just enough folds to make the line in which the cursor is located not folded
--   zM: Close all foldings
--   zR: Open all foldings
-- source: https://github.com/nvim-treesitter/nvim-treesitter/wiki/Installation
vim.api.nvim_create_autocmd({ 'BufEnter', 'BufAdd', 'BufNew', 'BufNewFile', 'BufWinEnter' }, {
    group = vim.api.nvim_create_augroup('TS_FOLD_WORKAROUND', {}),
    callback = function()
        vim.opt.foldmethod = 'expr'
        vim.opt.foldexpr   = 'nvim_treesitter#foldexpr()'
    end
})
```

#### `toggleterm.lua`

```lua
local is_ok, toggleterm = pcall(require, 'toggleterm')
if not is_ok then
    return
end

vim.o.shell = "/bin/fish"

toggleterm.setup({
    size = 20,
    open_mapping = [[<C-\>]], --  How to open a new terminal
    hide_numbers = true, -- hide the number column in toggleterm buffers
    direction = 'float',
    close_on_exit = true, -- close the terminal window when the process exits
    shell = vim.o.shell, -- change the default shell
    shade_filetypes = {},
    float_opts = {
        -- The border key is *almost* the same as 'nvim_open_win'
        -- see :h nvim_open_win for details on borders however
        -- the 'curved' border is a custom border type
        -- not natively supported but implemented in this plugin.
        border = 'single',
        winblend = 0,
    },
})


-- define key mappints
--  t: terminal mode
function _G.set_terminal_keymaps()
    local opts = { noremap = true, buffer = 0 }
    -- Use <C-\> to toggle terminals when direction='float'
    vim.keymap.set('t', '<esc>', [[<C-\><C-n>]], opts)
    vim.keymap.set('t', 'jk', [[<C-\><C-n>]], opts)
    -- We can use <C-h/j/k/l> to move cursor among windows(including terminal window)
    -- If we set direction='float', these key mappings wont' helpful
    vim.keymap.set('t', '<C-h>', [[<Cmd>wincmd h<CR>]], opts)
    vim.keymap.set('t', '<C-j>', [[<Cmd>wincmd j<CR>]], opts)
    vim.keymap.set('t', '<C-k>', [[<Cmd>wincmd k<CR>]], opts)
    vim.keymap.set('t', '<C-l>', [[<Cmd>wincmd l<CR>]], opts)
end

-- if you only want these mappings for toggle term use term://*toggleterm#* instead
vim.cmd('autocmd! TermOpen term://* lua set_terminal_keymaps()')


-- Toggleterm also exposes the `Terminal` class so that this can be used to create
-- custom terminals for showing terminal UIs like `lazygit`, `htop` etc.
local Terminal = require('toggleterm.terminal').Terminal

-- cmd = string -- command to execute when creating the terminal e.g. 'top'
local htop = Terminal:new({ cmd = 'htop', hidden = true })

function _HTOP_TOGGLE()
    htop:toggle()
end
```

## Problems

1. Problem: `clipboard: No provider. Try “:checkhealth” or “:h clipboard”`

   Solution: `sudo pacman -S xsel`
