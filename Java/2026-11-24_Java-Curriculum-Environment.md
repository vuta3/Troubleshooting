# Java学習環境のトラブルシューティング

- 発生日：2026/11/24
- OS：Windows 11
- エディター：VSCode
- ターミナル：PowerShell
- 使用技術：Java / Node.js / npm / Jest
- 対応状況：解決済み

---

## 1. 次回起動時にやること

恒久的な環境設定は変更せず、毎回ターミナルを起動した際に以下の設定を行う。

### ① 文字コードの設定

```powershell
chcp 65001

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

[Console]::InputEncoding = [System.Text.Encoding]::UTF8

$OutputEncoding = [System.Text.Encoding]::UTF8

$env:JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Dstdout.encoding=UTF-8 -Dstderr.encoding=UTF-8"
```

### ② 改行コードの設定

```powershell
$env:JDK_JAVA_OPTIONS = '-Dline.separator="' + "`n" + '"'
```

### ③ 練習用Webサーバーの起動

```powershell
npm.cmd run practice
```

ブラウザで以下にアクセスする。

```text
http://localhost:3333
```

### ④ 個別問題のテスト

例：問題1-2のテスト

```powershell
npx.cmd jest 1_2_variables
```

正常に採点されれば `PASS` と表示される。

※ Webサーバーを起動したまま個別テストを実行する場合は、新しいターミナルを開いて①②の設定を行う。

---

## 2. トラブル①：npmが実行できない

### 何が起きた？

以下のコマンドを実行したが、エラーになった。

```powershell
npm run practice
```

エラー内容：

```text
PSSecurityException
UnauthorizedAccess
```

### なぜ起きた？

PowerShellには、スクリプトの実行を制限する「実行ポリシー」という機能がある。

今回、npmを実行した際に `npm.ps1` というPowerShellスクリプトが呼び出され、実行が拒否された。

```text
npmコマンドを入力
        ↓
npm.ps1を呼び出す
        ↓
PowerShellの実行ポリシーに引っかかる
        ↓
エラー
```

### どう直した？

`npm.ps1` ではなく、`npm.cmd` を呼び出すようにした。

```powershell
npm.cmd run practice
```

テストを実行する際も同様。

```powershell
npx.cmd jest 1_2_variables
```

### 覚えておくこと

- `.ps1` はPowerShellスクリプト。
- `.cmd` はWindowsのコマンドスクリプト。
- PowerShellの実行ポリシーを変更しなくても対応できた。
- セキュリティ設定はむやみに緩和しない。

---

## 3. トラブル②：エラーメッセージが文字化けする

### 何が起きた？

Javaのコンパイルエラーが文字化けして、何が間違っているのか分からなかった。

### なぜ起きた？

Javaが出力した文字コードと、Node.jsなどの受信側が読み取る文字コードが一致していなかったと考えられる。

文字コードとは、コンピューターが文字を数値として扱うためのルールのこと。

異なるルールで文字を読み取ると、文字化けが発生する。

### どう直した？

文字コードをUTF-8に合わせるため、以下のコマンドを実行した。

```powershell
chcp 65001
```

コンソールのコードページをUTF-8に変更する。

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

コンソールの出力文字コードをUTF-8に設定する。

```powershell
[Console]::InputEncoding = [System.Text.Encoding]::UTF8
```

コンソールの入力文字コードをUTF-8に設定する。

```powershell
$OutputEncoding = [System.Text.Encoding]::UTF8
```

PowerShellから外部プログラムへ文字列を渡す際の文字コードを設定する。

```powershell
$env:JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Dstdout.encoding=UTF-8 -Dstderr.encoding=UTF-8"
```

Javaの文字コードに関する設定を変更する。

| 設定 | 意味 |
|---|---|
| file.encoding | デフォルトの文字コード |
| stdout.encoding | 標準出力の文字コード |
| stderr.encoding | 標準エラー出力の文字コード |

### 結果

文字化けしていたコンパイルエラーが、日本語で正常に表示されるようになった。

どの設定が直接的に文字化けを解消したのかは、まだ個別に検証していない。

---

## 4. トラブル③：正解なのにテストで不合格になる

### 何が起きた？

Javaのコードは正しく、コンパイルも成功しているのに、Jestによる自動採点で `FAIL` になった。

期待値と実測値は、見た目では同じだった。

```text
Expected: Taro
Received: Taro
```

同じ文字なのになぜ不正解になるのか分からなかった。

### なぜ起きた？

WindowsとLinuxで改行コードが異なることが原因と考えられる。

#### 改行コードとは？

コンピューターは、改行も特殊な文字として扱っている。

| 名称 | 表記 | 主な使用環境 |
|---|---|---|
| CR | \r | 制御文字 |
| LF | \n | Linux・macOS |
| CRLF | \r\n | Windows |

例えば、以下のJavaコードを実行する。

```java
System.out.println("Taro");
System.out.println(25);
```

Windowsでの出力は通常、以下のようになる。

```text
Taro\r\n25\r\n
```

Linuxでは以下のようになる。

```text
Taro\n25\n
```

画面上では同じように見えるが、内部的には異なる文字列になっている。

#### 採点プログラムを確認

Jestのテストコードには以下の記述があった。

```typescript
const lines = output.split('\n');
```

これは、改行文字 `\n` を基準に文字列を分割する処理。

しかし、Windowsでは改行に `\r\n` が使われているため、分割しても `\r` が残ってしまう。

```text
期待値：Taro
実測値：Taro\r
```

人間には見えない文字だが、コンピューターは異なる文字列として判定する。

その結果、不合格になったと考えられる。

### どう直した？

Javaが出力する改行コードを一時的にLFへ変更した。

```powershell
$env:JDK_JAVA_OPTIONS = '-Dline.separator="' + "`n" + '"'
```

| コマンドの要素 | 意味 |
|---|---|
| $env: | 環境変数を操作する |
| JDK_JAVA_OPTIONS | Java起動時のオプションを設定する環境変数 |
| -D | Javaのシステムプロパティを設定する |
| line.separator | Javaが使用する改行文字 |
| `n | PowerShellにおけるLFの表現 |

### 結果

再度テストを実行した。

```powershell
npx.cmd jest 1_2_variables
```

結果：

```text
PASS

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
```

無事に合格した。

### 本来の修正方法

採点プログラム側でWindowsとLinuxの両方の改行コードに対応することもできる。

変更前：

```typescript
const lines = output.split('\n');
```

変更案：

```typescript
const lines = output.split(/\r?\n/);
```

ただし、今回は提供された教材を勝手に変更したくなかったため、実行環境側で一時的に対応した。

---

## 5. Javaの学習中に発生したエラー

### 変数が初期化されていない

以下のコードを記述したところ、コンパイルエラーになった。

```java
String name;
int age;
double height;

System.out.println(name);
System.out.println(age);
System.out.println(height);
```

エラー内容：

```text
変数nameは初期化されていない可能性があります
```

### 原因

変数を宣言しただけで、値を代入していなかった。

変数はデータを入れる箱のようなもの。

箱を用意しても、中身を入れなければローカル変数を使用することはできない。

### 修正

```java
String name = "Taro";
int age = 25;
double height = 170.5;
```

今回の問題では採点プログラムが出力処理を自動的に追加するため、System.out.println()を書く必要はなかった。

これは環境の不具合ではなく、Javaの仕様によるエラー。

---

## 6. 今回覚えたコマンド

| コマンド | 用途 |
|---|---|
| chcp | コンソールのコードページ確認 |
| chcp 65001 | UTF-8への変更 |
| Get-ExecutionPolicy -List | PowerShellの実行ポリシー確認 |
| java -version | Javaのバージョン確認 |
| javac -version | Javaコンパイラのバージョン確認 |
| node -v | Node.jsのバージョン確認 |
| npm.cmd -v | npmのバージョン確認 |
| npm.cmd run practice | 練習用Webサーバーの起動 |
| npm.cmd test | 全問題のテスト |
| npx.cmd jest 01_variables | 第1章のテスト |
| npx.cmd jest 1_2_variables | 個別問題のテスト |
| Ctrl + C | 実行中の処理を停止 |

---

## 7. 今回覚えたIT用語

| 用語 | 意味 |
|---|---|
| Execution Policy | PowerShellのスクリプト実行制限 |
| Encoding | 文字コードの変換・表現方式 |
| UTF-8 | 文字コードの一種 |
| CRLF | Windowsで一般的な改行コード |
| LF | Linuxなどで一般的な改行コード |
| 環境変数 | プログラムに設定情報を渡す仕組み |
| stdout | 標準出力 |
| stderr | 標準エラー出力 |
| JVM | Java仮想マシン |
| Jest | 自動テストフレームワーク |
| Compile | ソースコードを実行可能な形式などに変換すること |
| Debug | プログラムの不具合を調査・修正すること |
| Workaround | 一時的な回避策 |
| Root Cause | 根本原因 |

---

## 8. 今回の反省・気付き

- プログラムが正しいからといって、必ず正常に動作するとは限らない。
- OSの違いによって実行結果に差異が発生することがある。
- 文字コードと改行コードは別物。
- エラーメッセージが読めなければ、まず読める状態にする。
- ExpectedとReceivedを比較することが重要。
- 設定変更を行う際は、影響範囲を意識する。
- 一度に複数の設定を変更すると、どの設定が有効だったのか分からなくなる。
- エラーが発生しても、安易に再インストールしない。
- コンパイルエラーと自動採点エラーを区別する。

今後も学習環境では、可能な限り恒久的な設定変更を避け、必要な設定を一時的に適用する。

毎回コマンドを実行することで、コマンドの意味や実行環境の仕組みを覚えていきたい。
