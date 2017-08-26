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

If you are getting this error:

 ```
 xcode-select: error: tool 'xcodebuild' requires Xcode, but active developer directory '/Library/Developer/CommandLineTools' is a command line tools instance
 ```
 
First install and open XCode. If the problem persists, try setting the command-line interface location as shown in this stack overflow [question](https://stackoverflow.com/questions/17980759/xcode-select-active-developer-directory-error/17980786)
 

[pathogen](https://github.com/tpope/vim-pathogen) as package manager.

[solarized](https://github.com/altercation/vim-colors-solarized) color scheme.

