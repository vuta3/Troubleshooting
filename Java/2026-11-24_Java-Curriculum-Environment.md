# Java学習環境における実行・自動採点エラー

## 1. 基本情報

| 項目 | 内容 |
|---|---|
| 発生日 | 2026/11/24 |
| 分類 | 開発環境・OS互換性 |
| OS | Windows 11 |
| 使用エディター | Visual Studio Code |
| 使用シェル | PowerShell |
| 使用技術 | Java、Node.js、npm、Jest |
| 対応状況 | 解決済み |
| 対応方針 | 恒久的なシステム設定変更を避け、一時的な環境設定で対応 |

## 2. 発生事象

Java学習用カリキュラムの起動および自動採点時に、以下の問題が発生した。

- npmコマンドが実行できない。
- Javaのコンパイルエラーが文字化けする。
- 未初期化の変数を参照するとコンパイルエラーが発生する。
- 正しいJavaコードを記述しても自動採点で不合格になる。

## 3. 障害①：npmコマンドの実行エラー

### 発生事象

以下のコマンドを実行したところ、エラーが発生した。

```powershell
npm run practice
```

### エラーメッセージ

```text
PSSecurityException
UnauthorizedAccess
```

### 原因

PowerShellの実行ポリシーによって、npm.ps1の実行が制限されていた。

PowerShellからnpmを実行すると、npm.ps1が呼び出される場合がある。

### 対処方法

npm.cmdを明示的に指定して実行した。

```powershell
npm.cmd run practice
```

### 対応結果

正常に実行できることを確認した。

PowerShellの実行ポリシーを恒久的に変更する必要はない。

## 4. 障害②：コンパイルエラーの文字化け

### 発生事象

自動採点時に表示されるJavaのコンパイルエラーが文字化けし、原因を確認できなかった。

### 原因

Javaのエラー出力と、Node.jsなどの受信側で使用する文字コードが一致していなかったと推定される。

### 対処方法

PowerShellで以下のコマンドを実行した。

```powershell
chcp 65001

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

[Console]::InputEncoding = [System.Text.Encoding]::UTF8

$OutputEncoding = [System.Text.Encoding]::UTF8

$env:JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Dstdout.encoding=UTF-8 -Dstderr.encoding=UTF-8"
```

### 対応結果

Javaのコンパイルエラーが正常な日本語で表示されるようになった。

なお、どの設定が文字化けの解消に直接寄与したかについては、個別の検証を実施していない。

## 5. 障害③：未初期化変数の参照

### 発生事象

以下のJavaコードを実行したところ、コンパイルエラーが発生した。

```java
String name;
int age;
double height;

System.out.println(name);
System.out.println(age);
System.out.println(height);
```

### エラーメッセージ

```text
変数nameは初期化されていない可能性があります
```

### 原因

ローカル変数に値を代入せず、その変数を参照していた。

Javaでは、ローカル変数を参照する前に値が確実に代入されている必要がある。

### 対処方法

変数の宣言時に初期値を設定した。

```java
String name = "Taro";
int age = 25;
double height = 170.5;
```

本カリキュラムでは採点プログラムが出力処理を追加するため、当該問題では変数の宣言と初期化のみを記載した。

### 対応結果

コンパイルエラーが解消された。

## 6. 障害④：正しい解答が自動採点で不合格になる

### 発生事象

正しいJavaコードを記述しているにもかかわらず、Jestによる自動採点で不合格になった。

期待値と実測値は画面上では同じ文字列に見えていた。

```text
Expected: Taro
Received: Taro
```

### 原因

WindowsとLinuxの改行コードの違いに起因する可能性が極めて高い。

| OS | 改行コード |
|---|---|
| Windows | CRLF（\r\n） |
| Linux | LF（\n） |
| macOS | LF（\n） |

採点プログラムでは以下の処理によって出力結果を分割していた。

```typescript
const lines = output.split('\n');
```

Windows環境でJavaがCRLFを出力した場合、LFで分割してもCRが残存する。

```text
期待値：Taro
実測値：Taro\r
```

そのため、文字列比較で不一致が発生したと考えられる。

### 対処方法

PowerShellで以下の環境変数を設定した。

```powershell
$env:JDK_JAVA_OPTIONS = '-Dline.separator="' + "`n" + '"'
```

Javaの実行時に使用される改行コードをLFに変更した。

### 動作確認

以下のコマンドで再度テストを実行した。

```powershell
npx.cmd jest 1_2_variables
```

### 対応結果

```text
PASS

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
```

環境変数の設定後、自動採点に合格した。

### 恒久対策案

採点プログラムの管理者が対応する場合、以下のようにCRLFとLFの両方に対応させる方法が考えられる。

```typescript
const lines = output.split(/\r?\n/);
```

今回は採点プログラムを変更せず、実行環境側で対応した。

## 7. 次回起動時の対応手順

今回の環境変数は一時的な設定であるため、新しいターミナルを起動した際は以下を実行する。

```powershell
chcp 65001

[Console]::OutputEncoding = [System.Text.Encoding]::UTF8

[Console]::InputEncoding = [System.Text.Encoding]::UTF8

$OutputEncoding = [System.Text.Encoding]::UTF8

$env:JAVA_TOOL_OPTIONS="-Dfile.encoding=UTF-8 -Dstdout.encoding=UTF-8 -Dstderr.encoding=UTF-8"

$env:JDK_JAVA_OPTIONS = '-Dline.separator="' + "`n" + '"'
```

練習用Webサーバーの起動：

```powershell
npm.cmd run practice
```

ブラウザから以下にアクセスする。

```text
http://localhost:3333
```

個別問題のテスト：

```powershell
npx.cmd jest 1_2_variables
```

## 8. 学習事項

今回のトラブルシューティングを通じて、以下の技術について理解を深めた。

- PowerShellの実行ポリシー
- 文字コードとUTF-8
- CRLFとLFの違い
- 環境変数の設定方法
- Javaの変数宣言と初期化
- Javaのコンパイルエラー
- Jestによる自動採点
- OS間の互換性
- 一時対処と恒久対策の違い

## 9. 所感

プログラムの実行結果が想定と異なる場合、ソースコードだけでなく、OS、文字コード、改行コード、実行環境、自動テストプログラムについても調査する必要がある。

また、障害対応においては、影響範囲を把握し、必要最小限の設定変更によって問題を解決することが重要である。
