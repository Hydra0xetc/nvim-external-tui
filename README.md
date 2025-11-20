# nvim-external-tui

A Neovim plugin for seamlessly integrating external TUI (Terminal User Interface) tools that can open files at specific line numbers.

## Features

- Simple API for registering external TUI tools
- Automatic command creation with visual selection support
- Terminal window management (floating by default)
- Bidirectional communication between Neovim and external tools
- Support for pre-launch and post-callback hooks
- Automatic callback function generation

## Installation

### Using [lazy.nvim](https://github.com/folke/lazy.nvim)

```lua
{
  'gfontenot/nvim-external-tui',
  dependencies = {
    'folke/snacks.nvim', -- Required for terminal management
  },
  config = function()
    -- Your tool configurations here
  end
}
```

### Using [packer.nvim](https://github.com/wbthomason/packer.nvim)

```lua
use {
  'gfontenot/nvim-external-tui',
  requires = {
    'folke/snacks.nvim', -- Required for terminal management
  },
  config = function()
    -- Your tool configurations here
  end
}
```

## Usage

### Basic Example

```lua
local external_tui = require('external-tui')

local config = external_tui.add({
  user_cmd = 'Scooter',      -- Creates :Scooter command
  cmd = 'scooter',            -- External command to run
  text_arg = '--search-text', -- Flag for passing search text
})
```

This creates a `:Scooter` command that:
- Accepts visual selection: `:'<,'>Scooter`
- Accepts arguments: `:Scooter search_term`
- Opens without arguments: `:Scooter`

### Getting the Editor Command

The `add()` function returns a table with the editor command that needs to be configured in your external tool:

```lua
local config = external_tui.add({ ... })

print(config.editor_command)
-- Output: nvim --server $NVIM --remote-send '<cmd>lua EditLineFromScooter("%file", %line)<CR>'
print(config.callback_name)
-- Output: EditLineFromScooter
```

### Advanced Example with Hooks

```lua
external_tui.add({
  user_cmd = 'Indextui',
  cmd = 'indextui',
  text_arg = '--search-text',
  editor_flag = '--editor',  -- Tool supports passing editor command directly

  -- Called before launching the TUI
  pre_launch = function(search_text)
    print("Launching with search:", search_text)
    vim.cmd('write') -- Save current buffer
  end,

  -- Called after opening the file
  post_callback = function(file_path, line)
    vim.cmd('normal! zz') -- Center the line on screen
  end,
})
```

## API Reference

### `external_tui.add(opts)`

Register a new external TUI tool integration.

#### Options

| Option | Type | Required | Default | Description |
|--------|------|----------|---------|-------------|
| `user_cmd` | `string` | Yes | - | Neovim command name (e.g., 'Scooter' creates `:Scooter`) |
| `cmd` | `string` | Yes | - | External command to execute (e.g., 'scooter') |
| `text_arg` | `string` | No | `nil` | Flag for passing search text (e.g., '--search-text') |
| `editor_flag` | `string` | No | `nil` | Flag for passing editor command (e.g., '--editor') |
| `file_format` | `string` | No | `'%file'` | Template variable for file path in tool's config |
| `line_format` | `string` | No | `'%line'` | Template variable for line number in tool's config |
| `pre_launch` | `function` | No | `nil` | Hook called before launching TUI: `function(search_text)` |
| `post_callback` | `function` | No | `nil` | Hook called after opening file: `function(file_path, line)` |

#### Returns

Table with:
- `editor_command`: String to configure in external tool
- `callback_name`: Name of the generated callback function

## Configuring External Tools

After registering a tool in Neovim, you need to configure the external tool to call back to Neovim when a file is selected.

### Example: Scooter

In `~/.config/scooter/config.toml`:

```toml
[editor_open]
command = "nvim --server $NVIM --remote-send '<cmd>lua EditLineFromScooter(\"%file\", %line)<CR>'"
```

### Example: Indextui with Editor Flag

If your tool supports passing the editor command as a flag (using `editor_flag` option), the plugin automatically includes it in the command:

```lua
external_tui.add({
  user_cmd = 'Indextui',
  cmd = 'indextui',
  editor_flag = '--editor',
})
```

This automatically passes the editor command when launching:
```bash
indextui --editor='nvim --server $NVIM --remote-send ...'
```

### Template Variables

The editor command uses template variables that the external tool should replace:
- `%file` - Full path to the selected file
- `%line` - Line number to jump to

These defaults can be overridden by passing the `file_format` and `line_format` options.

## How It Works

1. User invokes command (e.g., `:Scooter` or `:'<,'>Scooter`)
2. Optional pre-launch hook runs
3. Plugin launches external TUI in a floating terminal
4. User selects a file to edit in the TUI
5. TUI calls back to Neovim using `nvim --remote-send`
6. Plugin closes terminal and opens the file at the specified line
7. Optional post-callback hook runs

## Examples

### [Scooter](https://github.com/thomasschafer/scooter)
```lua
local external_tui = require('external-tui')

external_tui.add({
  user_cmd = 'Scooter',
  cmd = 'scooter',
  text_arg = '--search-text',
})
```

Then configure the editor command for scooter in `~/.config/scooter/config.toml`:
```toml
[editor_open]
command = "nvim --server $NVIM --remote-send '<cmd>lua EditLineFromScooter(\"%file\", %line)<CR>'"
```

### Indextui Integration with Auto-save

```lua
external_tui.add({
  user_cmd = 'Indextui',
  cmd = 'indextui',
  text_arg = '--search-text',
  editor_flag = '--editor',
  file_format = '{file}',
  line_format = '{line}',

  pre_launch = function(search_text)
    -- Save all modified buffers before launching
    vim.cmd('wall')
  end,

  post_callback = function(file_path, line)
    -- Center the line and highlight it briefly
    vim.cmd('normal! zz')
    vim.defer_fn(function()
      vim.cmd('normal! zvzz')
    end, 100)
  end,
})
```

## Requirements

- Neovim >= 0.9.0
- [snacks.nvim](https://github.com/folke/snacks.nvim) for terminal management

## Related Projects

- [scooter](https://github.com/gfontenot/scooter) - A fuzzy file and content searcher. The neovim integration for Scooter was the inspiration for this plugin.
