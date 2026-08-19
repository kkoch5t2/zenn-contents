---
title: "「今使えるGeminiモデルはどれ？」が一発で分かるツールをGeminiに作らせた"
emoji: "🔹"
type: "tech"
topics:
  - "gemini"
  - "googleai"
  - "geminiapi"
  - "googleantigravity"
published: true
published_at: "2026-08-16 04:47"
---

## はじめに
Google AI StudioのAPIを使って学習していると、こんな経験はないでしょうか。

> 昨日まで動いてたモデルが、今日いきなり使えなくなった……

Geminiのモデルは頻繁にアップデートされます。新しいモデルが追加される一方で、既存モデルが予告なく非推奨になったり、APIから姿を消すこともしばしば。
エンジニアなら公式ドキュメントを読めばいい話なんですが……正直、**面倒くさい**。

というわけで、「**今この瞬間、自分のAPIキーで使えるモデルはどれか**」を自動で調べて、ついでにコーディング能力まで評価してくれるツールを**AIコーディングアシスタント（Antigravity）に作らせました**。

🔗 **リポジトリ**: [kkoch5t2/gemini-model-evaluator](https://github.com/kkoch5t2/gemini-model-evaluator)

---

## 開発の動機

### APIモデルの「突然死」問題
Google AI Studioは学習用途で無料枠が使えるので、勉強するうえでとても便利です。
ただ、こんな落とし穴があります。

- **モデル名の変更**: 昨日まで `gemini-pro` だったものが、いつの間にか別名に
- **突然のアクセス制限**: レート制限やリージョン制限で急に呼べなくなる
- **新モデルの追加**: 知らないうちに新しいモデルが増えている

公式ドキュメントを毎日チェックするのは現実的ではないし、モデルが動くかどうかは**実際にAPIを叩いてみないとわからない**ことも多い。

> じゃあ自動でチェックすればいいじゃん！

そう思ったので、AIコーディングアシスタントの **Antigravity** に作ってもらいました。

---

## どんなツール？
Jupyter Notebook（`.ipynb`）をセルごとに実行するだけで、指定したAPIで使えるGeminiモデルが分かるツールです。
以下の4ステップを実行していきます。

### Step 1: 環境セットアップ
```python
pip install -r requirements.txt
```
必要なパッケージ（`google-generativeai`, `pandas`, `python-dotenv` など）がインストールされます。

### Step 2: モデルの死活確認（Health Check）
```python
import health_check_gemini_models
health_check_gemini_models.main()
```
APIにアクセスして、`generateContent` をサポートしている全モデルに対して以下のテストを実施します。
```python
test_prompt = "テストです。「OK」とだけ返信してください。"
```
このシンプルなプロンプトで、

- ✅ **応答が返ってくるか**（アクセス可能か）
- ✅ **指示に正確に従えるか**（「OK」以外の余計なことを喋らないか）

を同時にチェック。通過したモデルだけが `working_models.json` に記録されます。

### Step 3: コーディング能力の評価（LLM-as-a-Judge）
**LLM-as-a-Judge** とは、LLM（大規模言語モデル）自身を「審査員」として使い、他のモデルの出力を評価させる手法です。
人間がいちいち採点しなくても、AIが自動で品質を判定してくれます。

ここが面白いところ。死活確認を通過した「精鋭モデル」たちに、こんな**激辛コーディング課題**を出題します。

```python
target_prompt = """与えられた日本語の文章から、ひらがなだけを抽出し、
それを五十音順の逆順（ん〜あ）に並び替えて結合するPython関数 
`reverse_hiragana` を書いてください。
ただし、以下の制約を必ず守ること：
1. for文やwhile文などのループ構文は一切使用禁止
2. コメントや解説などの日本語テキストは一切出力せず、
   Pythonコードのみを出力すること。挨拶も厳禁。"""
```

この課題のポイントは3つ。

| チェック項目 | 判定方法 | 減点 |
|:---|:---|:---|
| 余計なテキスト（解説等）の有無 | Pythonで機械的にチェック | -4点 |
| ループ構文（`for`/`while`）の使用 | Pythonで機械的にチェック | -4点 |
| ロジックの正しさ | 審査員モデル（LLM）が判定 | -4点 |

**ハイブリッド判定**（プログラムによる機械チェック + LLMによるロジック評価）で、10点満点の減点方式でスコアリングします。
LLMだけに任せるとガバガバ判定になりがちなので、形式チェックはPythonで厳密に行うのがミソです。

……とはいえ、**正直に言うと判定精度はまだまだ微妙**です。
Pythonによる機械チェック（余計なテキストやループの検出）は確実に動きますが、LLMによるロジック判定は「明らかに間違っているコードにOKを出す」「正しいコードなのに減点する」といったブレが普通に発生します。
LLM-as-a-Judgeの限界というか、**審査員側も無料枠のLLMを使っているので、判定の質にも限界がある**のが現実ですね。

### Step 4: 結果の表示
```python
import pandas as pd
df = pd.read_csv('evaluation_results.csv')
df_sorted = df.sort_values(by='score_num', ascending=False)
display(df_sorted[['model', 'status', 'score', 'reason']])
```

評価結果がランキング形式で表示されます。

#### 実行結果の例
実際に実行した結果がこちら（一部抜粋）

| 順位 | モデル | スコア | 理由 |
|:---:|:---|:---:|:---|
| 1 | gemini-flash-lite-latest | 10/10 | Unicodeの範囲指定で正しく実装 |
| 2 | gemini-3.1-flash-lite-preview | 10/10 | filter関数とsorted関数を正しく使用 |
| 3 | gemini-3.5-flash-lite | 10/10 | filterとsortedでひらがな抽出・逆順ソートが正確 |
| 4 | gemini-robotics-er-2-preview | 10/10 | ループ未使用で正しく実装 |
| 5 | gemini-3.1-flash-lite | 6/10 | ループ使用(-4)で減点 |

---
## 技術的なポイント

### 1. 死活確認で「指示遂行能力」も同時にテスト
単に「APIが200を返すか」ではなく、**「OKとだけ返せるか」** までチェックしています。これにより、API自体は生きているが指示を無視するモデルも弾けます。
```python
if output.strip(" .\"'").upper() != "OK":
    print("× [NG] 失敗 (余計な出力あり)")
```

### 2. LLMの判定に頼りすぎない「ハイブリッド採点」
LLM-as-a-Judgeは便利ですが、LLM自身が形式チェックを間違えることがあります。
そこで、

- **形式チェック**（余計なテキスト・ループ使用）→ **Pythonで機械的に判定**
- **ロジックの正しさ**→ **審査員モデル（LLM）に判定させる**

という役割分担にしています。

### 3. 審査員モデルのリトライ機能
審査員モデルが不正なJSONを返す場合に備えて、最大3回のリトライ機能を実装しています。
```python
max_retries = 3
for attempt in range(max_retries):
    try:
        eval_response = judge_model.generate_content(
            eval_prompt,
            generation_config=genai.types.GenerationConfig(
                response_mime_type="application/json",
            )
        )
        eval_data = json.loads(eval_response.text)
        break  # 成功したらリトライループを抜ける
    except json.JSONDecodeError as e:
        if attempt < max_retries - 1:
            time.sleep(2)
        else:
            raise e
```

### 4. レートリミット対策
各APIリクエストの間に `time.sleep()` を挟んで、Google AI Studioの無料枠のレートリミットに引っかからないようにしています。

---

## Antigravityで作らせた感想
今回のツールは、AIコーディングアシスタント「**Antigravity**」に指示を出しながら開発しました。
やったことは基本的に「**こういうものが欲しい**」を伝えただけ。具体的には、

- 「Gemini APIで使えるモデルを全部取得して、動くかチェックしたい」
- 「動くモデルにはコーディング課題を出して評価したい」
- 「LLMだけの判定だとガバるから、Pythonでも機械的にチェックして」

こうした要件を伝えると、ファイル構成からコードの実装まで一気に生成してくれました。
**公式ドキュメントを読み解く時間 vs AIに作らせる時間**、圧倒的に後者のほうが速い。
もちろん出力されたコードのレビューは必要ですが、ゼロから書くよりは遥かに効率的です。

---

## 使い方（クイックスタート）
```bash
# 1. クローン
git clone https://github.com/kkoch5t2/gemini-model-evaluator.git
cd gemini-model-evaluator
# 2. 仮想環境作成
python -m venv env
# 3. .envにAPIキーを設定
echo GEMINI_API_KEY=あなたのAPIキー > .env
# 4. VS Codeでノートブックを開く
# 5. 右上の「Select Kernel」から作成した仮想環境（env）を選択
# 6. セルを上から順に実行
```

---

## まとめ
- **Google AI StudioのAPIモデルは頻繁に変わる**ので、手動で追跡するのは大変
- **自動で死活確認＋品質評価**できるツールを作れば、「今使えるベストなモデル」が一目でわかる
- **AIコーディングアシスタント（Antigravity）** を使えば、こうしたツールも効率的に開発できる

🔗 **リポジトリ**: [kkoch5t2/gemini-model-evaluator](https://github.com/kkoch5t2/gemini-model-evaluator)