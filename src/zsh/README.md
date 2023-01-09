# zsh

このドキュメントではzshに関する設定やプラグインなどを書いていきます

## 前提
- OSはmacOS
- `~/.zshrc`に設定を記述する

## zsh-autosuggestions
[zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) は、ターミナルのコマンド履歴に基づいて、コマンド入力中にコマンド候補を表示してくれるプラグインです

### インストール方法
```sh
brew install zsh-autosuggestions
```

ターミナルに表示されたコマンドを`~/.zshrc`に追記する

```diff:~/.zshrc
+ source /usr/local/share/zsh-autosuggestions/zsh-autosuggestions.zsh
```

![](images/01.png)

