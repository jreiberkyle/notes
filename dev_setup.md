# Development Setup

Utilities for my development setup

## Terminal (Mac OSX)

Colorscheme: [Solarized](https://github.com/altercation/solarized).
Install directions in this [blog post](https://jakoblaegdsmand.com/en/blog/how-to-get-an-awesome-looking-terminal-on-mac-os-x/)

## Editor

MacVim installed using homebrew and overriding system vim

```
brew install macvim --with-override-system-vim
```
note that this requires Xcode to be installed (achieved by starting the XCode application)

error code if XCode isn't installed:
 ```
 xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
 ```

[pathogen](https://github.com/tpope/vim-pathogen) as package manager.

[solarized](https://github.com/altercation/vim-colors-solarized) color scheme.

