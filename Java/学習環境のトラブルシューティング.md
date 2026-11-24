# Java学習環境のトラブルシューティング

2026/11/24

Javaの学習を進めようとしたら、npmが実行できなかったり、文字化けしたり、正解なのに不合格になったり??

プログラム以外にもこんなところでエラーが起きるんだと、とってもびっくりした。

実務ではこういったトラブルにもスムーズに対応できるようになりたいな。

忘れないようにのこしとく

---

### ① npmが実行できない

**なぜ起こった？**

PowerShellの実行ポリシーによって、`npm.ps1` の実行が制限されていたため。

**対処コマンド**

```powershell
npm.cmd run practice
```

`npm.cmd` を指定することで、PowerShellの実行ポリシーを変更せずに実行できる。

実行ポリシーを確認したい場合は以下。

```powershell
Get-ExecutionPolicy -List
```

---

### ② エラーメッセージが文字化けする

**なぜ起こった？**

JavaとNode.jsなどの間で、文字コードの認識が一致していなかったと考えられる。

文字コードが異なると、同じデータでも正しく文字として読み取れなくなる。

**対処コマンド**

```powershell
chcp 65001

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

[Console]::InputEncoding = [System.Text.Encoding]::UTF8

$OutputEncoding = [System.Text.Encoding]::UTF8

$env:JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Dstdout.encoding=UTF-8 -Dstderr.encoding=UTF-8"
```

文字コードをUTF-8に統一することで、エラーメッセージが正常に表示されるようになった。

※ どの設定が直接的な解決につながったかは未検証。

---

### ③ 正解なのにテストで不合格になる

**なぜ起こった？**

WindowsとLinuxの改行コードの違いが原因と考えられる。

| OS | 改行コード |
|---|---|
| Windows | CRLF（\r\n） |
| Linux | LF（\n） |

採点プログラムが `\n` のみを基準に文字列を分割していたため、Windowsの `\r` が残ってしまった。

```text
期待値：Taro
実測値：Taro\r
```

見た目は同じでも、コンピューターは異なる文字列として判定する。

**対処コマンド**

```powershell
$env:JDK_JAVA_OPTIONS = '-Dline.separator="' + "`n" + '"'
```

Javaが使用する改行コードを一時的にLFへ変更する。

その後、再度テストを実行。

```powershell
npx.cmd jest 1_2_variables
```

結果は `PASS`！

---

## 覚えたコマンド

| コマンド | 意味 |
|---|---|
| `chcp` | コードページの確認 |
| `chcp 65001` | コンソールをUTF-8に変更 |
| `Get-ExecutionPolicy -List` | 実行ポリシーの確認 |
| `java -version` | Javaのバージョン確認 |
| `javac -version` | Javaコンパイラのバージョン確認 |
| `node -v` | Node.jsのバージョン確認 |
| `npm.cmd -v` | npmのバージョン確認 |
| `npm.cmd run practice` | 練習用Webサーバーの起動 |
| `npm.cmd test` | 全問題のテスト |
| `npx.cmd jest 01_variables` | 第1章のテスト |
| `$env:` | 環境変数の参照・設定 |
| `Ctrl + C` | 実行中の処理を停止 |

---

## 覚えたIT用語

| 用語 | 意味 |
|---|---|
| Execution Policy | スクリプトの実行を制限する仕組み |
| Encoding | 文字をデータとして表現する方式 |
| UTF-8 | 文字エンコーディングの一種 |
| CRLF | Windowsで一般的な改行コード |
| LF | Linuxで一般的な改行コード |
| 環境変数 | プログラムに設定情報を渡す仕組み |
| stdout | 標準出力 |
| stderr | 標準エラー出力 |
| JVM | Java仮想マシン |
| Jest | 自動テストフレームワーク |
| Compile | ソースコードを実行可能な形式などに変換すること |
| Debug | 不具合を調査・修正すること |
| Expected | 期待値 |
| Received | 実測値 |
| Root Cause | 根本原因 |
| Workaround | 一時的な回避策 |

---
