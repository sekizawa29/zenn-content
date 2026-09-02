---
title: "Claude Code の auto モードで rm -rf 事故を止める PreToolUse hook を書いた"
emoji: "🛡️"
type: "tech"
topics: ["claudecode", "claude", "python", "cli", "shell"]
published: true
---

## この記事で分かること

Claude Code を auto モードや `--dangerously-skip-permissions` で回すと、確認なしにコマンドが走ります。便利な反面、`rm -rf` の一発で数日分の作業が消えるリスクを常に抱えることになります。

この記事では、`rm` / `rmdir` だけを狙って割り込み、**allow / ask / deny の3段階**で判定する PreToolUse hook を紹介します。Python 標準ライブラリだけで動き、ソースは 250 行程度です。

コード: https://github.com/sekizawa29/claude-guard-delete

## 起きうる事故

以下は私の体験談ではなく、AI エージェントに shell を任せていれば**一般的に起きうるパターン**として挙げるものです。

- `rm -rf ./dist` のつもりが cwd がズレていて、プロジェクト直下で走る
- `rm -rf $BUILD_DIR` の変数が空で、`rm -rf /` 相当になる
- `find . -name '*.log' | xargs rm` の起点が `~` になっている
- `~/.claude` や `~/.ssh` を「使ってなさそうな設定」と判断して消す

いずれも悪意ではなく**文脈の取り違え**で起きます。そして auto モードでは、人間が読む前に実行が終わっています。

「permissions で `rm` を deny する」手もありますが、それだと `/tmp` の掃除まで毎回止まり、結局 bypass したくなります。欲しいのは「**危ないものだけ止め、判断できるものは人間に聞き、無害なものは通す**」ガードでした。

## hook の仕組み

Claude Code の hook は、特定イベントで外部コマンドを叩き、stdin に JSON を流す仕組みです。`PreToolUse` はツール実行の直前に発火し、Bash ツールならこういう JSON が届きます。

```json
{"tool_name": "Bash", "tool_input": {"command": "rm -rf ./dist"}, "cwd": "/Users/me/app"}
```

hook 側は stdout に JSON を返すだけで、実行の可否を決められます。

```python
def emit(decision, reason):
    print(json.dumps({
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": decision,   # "allow" | "ask" | "deny"
            "permissionDecisionReason": reason,
        }
    }))
```

- `allow` … 確認をスキップして実行
- `ask` … 理由を表示して人間に確認を求める。**auto モードでも止まる**
- `deny` … 実行しない。理由は Claude にも見えるので、次の行動を変えられる
- 何も出力しない … hook は関与しない

`ask` が使えるのがポイントで、「削除は人間が判断する。ただし何を消すかは hook が列挙して見せる」という運用ができます。

## なぜ shlex 解析なのか

素朴に `"rm" in command` で判定すると、すぐ困ります。

- `grep -n "rm" src/` … rm は検索パターン
- `echo "rm -rf /"` … 文字列
- `cat form.txt` … `form` の中に rm がある

逆に単語境界を厳しくすると、`sudo rm` や `find | xargs rm` を見逃します。

そこで `shlex` を `punctuation_chars=True` で使い、シェルと同じルールでトークン化します。

```python
lex = shlex.shlex(cmd, posix=True, punctuation_chars=True)
lex.whitespace_split = True
tokens = list(lex)
# 'echo "rm -rf /"'      -> ['echo', 'rm -rf /']          ← 1トークン。rm はコマンドではない
# 'find . | xargs rm'    -> ['find', '.', '|', 'xargs', 'rm']
# 'cd x && rm -rf y'     -> ['cd', 'x', '&&', 'rm', '-rf', 'y']
```

トークン列を `;` `&&` `||` `|` で**セグメント**に分け、各セグメントの先頭（`VAR=val` と `sudo`/`env`/`xargs` などのラッパーを剥がした後）が `rm` / `rmdir` かどうかを見ます。これで「rm が**コマンドとして**呼ばれているか」だけを判定でき、引用符の中は自動的に無視されます。

ラッパーを剥がすのは検出のためで、剥がした結果 `wrapped=True` なら deny です。`sudo rm` を許す理由はなく、`xargs rm` は stdin 次第で何でも消せるからです。

## 3段階の判定表

| 判定 | 条件 | 例 |
|---|---|---|
| **deny** | `/`、ホーム、`/Users`、保護ルート、`~/.ssh` `~/.claude` `~/.config` `~/.aws` `~/.gnupg`、システム領域、`--no-preserve-root` | `rm -rf ~`、`rm -rf ~/.ssh` |
| **deny** | 対象を列挙できない: 複合コマンド・パイプ・ラッパー・リダイレクト・glob・複雑なブレース・未解決の変数・パース失敗 | `find . \| xargs rm`、`rm -rf build/*`、`rm $DIR` |
| **ask** | 単独の rm/rmdir で、対象パスを全部列挙できる | `rm foo.txt`、`rm -rf ./dist`、`rm a.{txt,md}` |
| **allow** | 全対象が `/tmp` 配下、または対象なし | `rm -rf /tmp/x` |
| 対象外 | rm がコマンドとして呼ばれていない | `grep rm file`、`echo "rm -rf /"` |

判定の順序は「列挙できるか → 致命的か → 全部一時領域か → ask」です。glob や変数を deny にするのは不便に見えますが、**hook は実行時の展開結果を知れない**ので、通せば「何を消すか分からないまま許可」になります。展開が絡む削除は人間がやる、と割り切りました。

`ask` のメッセージには、正規化した対象パスとフラグを載せます。

```
以下を削除しようとしています。実行してよいですか？ [再帰(-r)／強制(-f)] 1件: ~/app/dist
```

保護ルート（例: `~/projects`）の直下、つまりプロジェクト1個丸ごとが対象なら `⚠ プロジェクト全体を含みます` を付けます。

もう1つ、**fail-closed** です。hook 内で例外が起きたら deny。安全装置が壊れて素通りは本末転倒なので。

## 導入手順

1. スクリプトを `~/.claude/hooks/guard_delete.py` に置く
2. `~/.claude/settings.json` に追記する

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 ~/.claude/hooks/guard_delete.py", "timeout": 10 }
        ]
      }
    ]
  }
}
```

3. 保護したいディレクトリがあれば `GUARD_DELETE_PROTECTED="~/projects:~/work"` を `settings.json` の `env` に入れる
4. 動作確認

```bash
echo '{"tool_input":{"command":"rm -rf ~"},"cwd":"/"}' | python3 ~/.claude/hooks/guard_delete.py
```

deny されたら、Claude Code のプロンプトで `!` を先頭に付けて、hook を通さず**自分の手で**実行します。

```
! rm -rf build/*
```

deny の理由文にもこれを書いてあるので、Claude は「人間にやってもらう」と判断して提案側に回ります。「AI が削除を提案し、人間が `!` で実行する」分業を hook で強制している、とも言えます。

## 制限

正直に書いておきます。

- **`rm` / `rmdir` 以外は見ていません。** `find -delete`、`git clean -fdx`、`rsync --delete`、`> file` の上書きは素通りします。（`/bin/rm` のようなフルパス呼び出しは basename で判定するのでカバーしています。）
- `bash -c "rm -rf ~"` や `python -c "shutil.rmtree(...)"` など、別インタプリタに文字列で渡す形も見えません。
- シンボリックリンクは解決しません。
- Bash ツール経由だけが対象で、MCP サーバーの削除には効きません。

つまりこれは「よくある事故を1枚で減らすネット」であって、最後の砦ではありません。大事なものは git とバックアップで守るのが前提です。それでも、auto モードを常用していると、この1枚があるかないかで安心感がかなり違います。

余談。この記事の作業を Claude Code に任せていたら、README をヒアドキュメントで書き出すコマンド（`rm` の例がぎっしり入った `&&` 連結）がこの hook 自身に止められました。設計どおりですが、自分のガードに自分が引っかかるのは少し可笑しかったです。

## まとめ

- PreToolUse hook は stdout の JSON で allow / ask / deny を返せる
- `shlex` でトークン化してセグメント先頭を見れば、コマンドとしての rm だけ捕まえられる
- 列挙できない削除と致命的な対象は deny、`/tmp` は allow、それ以外は ask
- deny されたら `!` で人間が実行する

保護ディレクトリを足すだけで使えるはずです。よければ試してみてください。
