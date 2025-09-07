# dotfiles

Personal Arch Linux desktop configuration using git bare repository method.

## System Overview

| Category | Application |
|----------|------------|
| **Window Manager** | `niri` (Wayland compositor) |
| **Editor** | `neovim` |
| **Terminal** | `rio` / `wezterm` |
| **IME** | `kime` (Korean input) |
| **Status Bar** | `eww` |
| **Shell** | `fish` |
| **Prompt** | `starship` |

## Repository Structure

```
.
├── .cargo/
│   └── config.toml          # Cargo configuration for sccache
├── .config/
│   ├── fish/                # Fish shell configuration
│   │   ├── config.fish
│   │   └── functions/       # Custom fish functions (aliases)
│   ├── niri/                # Niri WM configuration
│   │   ├── config.kdl
│   │   └── *.jpg           # Wallpapers
│   ├── nvim/                # Neovim configuration
│   │   ├── init.lua
│   │   └── lua/
│   │       ├── plugins/     # Plugin configurations
│   │       ├── settings/    # Editor settings & LSP
│   │       └── snippets/    # Code snippets
│   ├── starship.toml       # Shell prompt configuration
│   ├── vivid/               # LS_COLORS themes
│   └── wezterm/             # Wezterm terminal configuration
├── .local/
│   ├── bin/                 # User binaries and scripts
│   │   ├── sc-gcc          # sccache wrapper for gcc
│   │   └── sc-g++          # sccache wrapper for g++
│   └── share/
│       └── applications/    # Desktop entries
└── .gitconfig               # Git configuration
```

## Installation

### Prerequisites

- Arch Linux base system installed
- Git available in the system

### Phase 1: Base System Installation

Use the provided installation script or manually install with `pacstrap` using the `pkglist` file.

### Phase 2: User Environment Setup

1. **Clone and install dotfiles:**
```bash
# Remove existing dotfiles (backup if needed)
rm -rf .*

# Run the install script
./install

# The install script performs:
# - Creates git bare repository at ~/.dotfiles
# - Clones dotfiles-contents repository
# - Checks out all configuration files to home directory
# - Configures git to ignore untracked files
```

2. **Install packages and tools:**
```bash
./install-packages

# This script installs:
# - Rust toolchain via rustup
# - sccache for compilation caching
# - Command-line tools (ripgrep, eza, fd, zoxide, etc.)
# - Development tools (tokei, tree-sitter-cli, typst-cli)
# - Clones and prepares source repositories for:
#   - niri, kime, swww, rio, wezterm, sirula
```

3. **Build from source:**

After logging out and back in (to apply fish shell environment):
```bash
# Build window manager and tools
cd ~/.local/source/niri && cargo build --release
cd ~/.local/source/kime && cargo build --release
# ... continue for other tools

# Link binaries to ~/.local/bin
ln -sf $PWD/target/release/niri ~/.local/bin/
```

4. **Final setup:**
- Install fonts from USB (proprietary fonts not included)
- Configure GitHub CLI: `gh auth login`
- Start niri session

## Configuration Details

### Fish Shell
- Custom functions for common commands (ls, vim, etc.)
- `dot` function for managing dotfiles repository
- Integration with fnm, juliaup, tree-sitter

### Neovim
- LSP support for multiple languages
- Tree-sitter based syntax highlighting
- Custom snippets with LuaSnip
- Telescope for fuzzy finding
- Custom color scheme (mytilus)

### Niri (Window Manager)
- Configured wallpapers for different seasons
- Custom keybindings in config.kdl
- Wayland native configuration

### Development Environment
- **Languages configured:** Rust, Python, C/C++, Julia, Haskell, TypeScript, Lua, Prolog, Racket, Typst
- **LSP servers:** rust-analyzer, pyright, clangd, tinymist, and more
- **Build acceleration:** sccache wrappers for gcc/g++

## Managing Dotfiles

After installation, use the `dot` command (fish function) to manage configurations:

```bash
# Check status
dot status

# Add new configuration
dot add .config/some-app/config
dot commit -m "Add some-app configuration"
dot push

# Update from remote
dot pull
```

## Notes

- System uses dual-boot with Windows (rEFInd bootloader)
- Locale is not set during installation to prevent font rendering issues
- Most applications are built from source and installed to `~/.local/`
- Minimal packages from pacman, preferring source builds for user applications
