---
layout: post
title:  '[Mac] iTerm2 + oh-my-zsh + powerlevel10k 安裝'
subtitle: 'Mac - iTerm2 + oh-my-zsh + powerlevel10k install'
background: '/img/posts/04.jpg'

date: 2020-05-22

tags: [Mac, Shell]
keywords: [zsh, iTerm2]
---
# 安裝 Homebrew
<a href="https://brew.sh/index_zh-tw" target="_blank">Homebrew</a>
```bash
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

# 安裝iTerm 2
<a href="http://iterm2.com/" target="_blank">iTerm2</a>  
下載後解壓縮，放到 應用程式。

# 安裝 oh-my-zsh
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

# 安裝字型 Hack Nerd Font
```bash
brew tap caskroom/fonts
brew cask install font-hack-nerd-font
```

# 安裝 powerlevel10k (oh-my-zsh 的 theme) 
```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
p10k configure
```

# 修改 zsh theme 為 powerlevel10k
```bash
vim ~/.zshrc

# 主題
ZSH_THEME="powerlevel10k/powerlevel10k"
```

# 修改 iTerm2 的 color scheme
```bash
Preferences > Profiles > Colors

右下角 Color Presets 選 Tango Dark 後
Foreground 改 c7c7c7
```

# 修改 iTerm2 的 Font
```bash
Preferences > Profiles > Text
點 Change Font 選 Hack Nerd Font
(選用) Size 用 18 
```

# CLI Syntax highlighting 高亮
```bash
git clone --depth=1 https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

vim ~/.zshrc

plugins=(
    zsh-syntax-highlighting  # 加上這行
)

# 修改設定要重啟zsh
```

# 自動命令推薦
- 要使用補完的指令 按 → 鍵

```bash
git clone --depth=1 https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

vim ~/.zshrc

plugins=(
    zsh-autosuggestions   # 加上這行
)

# 修改設定要重啟zsh
```

# 修改 iTerm2 跳至上下一個單字 熱鍵
Path : `Preferences` → `Profiles` → `Keys`
- 上一個單字 ⌥←

```
Keyboard Shortcut: ⌥←
Action: Send Escape Sequence
Esc+: b
```
- 下一個單字 ⌥→

```
Keyboard Shortcut: ⌥→
Action: Send Escape Sequence
Esc+: f
```