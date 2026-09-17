# Lab 2: 長い時間軸で Agent を動かす

今朝は Sparkles agent を構築し、その一挙手一投足を見守りました。
午後には、お店は誰も見ていない状態でソフトウェアを構築してほしいと考えています。
そして、その成果物を信頼できるかどうかも知りたいのです。

今回の仕事は、Sparkles がカウンター用の注文 kiosk ページを必要としている、というものです。
1 つの Claude が計画し、別の Claude が構築し、3 つ目が結果を判定して、合格するまで差し戻すループを実行します。
その後 Foundry を使って、このパターンを適切に実行します。検索可能なツールカタログ、すべてのステップのトレース (traces)、そして Claude が判定者になる評価 (evaluations) を使います。

この lab を終えると、次のことができるようになります。

- 仕込まれたバグを検出して修正する planner、generator、evaluator のループを実行する
- Web search に基づいて計画を根拠付けし、Claude が catalog を検索してツールを見つけられるようにする
- Foundry portal ですべての model call、token count、latency を確認する
- Foundry に evaluator を登録し、実際の実行を採点して、結果に Claude の書面による reasoning を表示する

**この lab の進め方。** すべては、すでに 'sparkles-loop/' と 'sparkles-evals/' にあるスクリプトから実行します。
それらを読み、実行し、動作を変更します。各 module は checkpoint で終わります。

**ここから新しく始める場合** は、'sparkles-agent/snapshots/agent-module-1.4.py' を 'agent.py' にコピーし、'.env' がまだ入力済みであることを確認してください。
それ以外の方は Module 2.1 から始めます。

**午後を通して扱う考え方:** 書く agent と判定する agent は同じではありません。
この考え方に、仕上げの度合いが異なる 3 つのレベルで出会います。

---

## Module 2.1: 自分の作業をチェックする agent (30 minutes)

**これが重要な部分です。** 1 分間動く agent にはよい prompt が必要です。
1 時間動く agent には、誰も各ステップを見ていないため、自分がまだ正しい方向に進んでいるかを判断する方法が必要です。
これこそが、単一の call と agent の違いです。何かが作業をチェックし、もう一度進めるかどうかを決めなければなりません。

パターンは常に同じ形です。開始する前に「完了」がどのような状態かを書き出します。
構築します。結果を意見ではなく、その完了条件と照らしてチェックします。
失敗をフィードバックして、合格するか round が尽きるまで再実行します。
この lab の他のすべて、toolbox、tracing、evaluations は、この loop にぶら下がっています。

![Three agents, one loop](images/05.0-loop-visual.png)

左から 1 行の intent が入ります。planner はそれを spec に変換します。仕事が完了したときに何が真でなければならないかであり、どう構築するかではありません。
generator はその spec に合わせて構築します。evaluator は結果をチェックし、critique とともに PASS または FAIL を返します。
FAIL は critique を generator に戻し、別の round を開始します。PASS は loop を終了します。

図の例では game 2048 を構築しており、そこでは evaluator が完成したページをブラウザーで開き、実際に遊べるかを確認します。
今回扱うのは cupcake kiosk で、チェックはページを読み取って中身を数えるスクリプトで行います。
役割は同じで、証拠が違うだけです。loop の中に、構築対象に固有のものはありません。それがこのパターンの要点です。

3 つのスクリプト、3 つの役割、2 つの Claude deployments があります。

- **planner.py**: 1 行の intent を入力し、testable な spec を出力します。Sonnet です。
- **generator.py**: spec を入力し、単一ファイルの kiosk ページを出力します。より高速で低コストな tier である Haiku です。圧倒的に多くの tokens を書きますが、何かを判定するのではなく spec から作業しています。
- **evaluator.py**: spec と hard evidence を入力し、critique とともに PASS または FAIL を出力します。作業の品質は judge の品質を超えないため、Sonnet を使います。

まずそれぞれを単独で実行して何を生成するかを確認し、その後 loop を実行します。

```
cd sparkles-loop
```

### Step A: the planner (5 minutes)

**実行する前に 'planner.py' を開いてください。** その仕事は、request を、人が見なくてもチェックできる項目のリストに変換することです。

- **'PROMPT'** は request です。今日の flavors、special、running order count を備えた kiosk ページです。人間には明確ですが、スクリプトが test できるものはその中にありません。
- **'SPEC_SCHEMA'** は、回答を prose ではなく固定 JSON として返させます。
- **'SYSTEM'** は、すべての acceptance criterion がページ上の 'data-testid' を指定しなければならない、と Claude に伝えます。これらは後で evaluator が探す名前です。

```
python planner.py
```

出力される spec を読んでください。3 つまたは 4 つの features があり、それぞれに acceptance criterion があり、'workspace/spec.json' に保存されます。

すべての criterion は 'data-testid' に結び付いています。これは要素に付ける label であり、スクリプトがページの見た目を気にせず見つけられるようにするものです。
spec は常に次の 5 つを含みます。

| 'data-testid' | The element | What has to be true |
|---|---|---|
| 'title' | 上部の shop name | 存在すること |
| 'flavor-list' | 今日の flavors | 存在し、3 つの flavors を保持し、それぞれが独立した element であること |
| 'special' | special of the day | 存在すること |
| 'order-btn' | Place Order button | 存在すること |
| 'order-count' | running count of orders | 存在すること |

このリストが、この module の残りの contract です。generator はこれらの名前を正確に使うよう指示され、'checks.py' がそれらを数え、evaluator がそれらに基づいて page を pass または fail します。
これは Module 2.4 で code evaluator が採点するのと同じリストでもあります。

### Step B: the generator, with a planted bug (5 minutes)

generator は kiosk を書きます。planner がいま生成した spec を受け取り、shop counter 用の単一 HTML ページを返します。今日の flavors、special、order を送る button を含みます。
1 ファイルで、build step も install もありません。

**'generator.py' を開いてください。**

- **'SYSTEM'** が generator の全 brief です。spec はここにあり、self-contained な HTML file を 1 つ返す、という内容です。
- sprint 間で memory はなく、evaluator の reasoning は決して見ません。返される critique text だけを受け取ります。これが、自分自身の作業を採点するのを防ぎます。
- **'seed()'** は、生成する代わりに first draft を読み込みます。

**The seed.** '--seed' は 'seeds/kiosk_buggy.html' を読み込みます。これは、私たちが 2 つの mistake を入れて書いたページです。
VS Code で開き、2 つの 'BUG' comments を見つけてください。

- **flavor が 1 つしか listed されていません。** spec は 3 つを求めています。
- **order counter がありません。** JavaScript は 'order-count' という element を更新しようとしますが、その element はページに追加されていません。まず element が存在するかを確認しているため、何も壊れません。count が表示されないだけです。

ブラウザーではページは完成して見えます。どちらの問題も、spec と照合しなければ見つけられません。

**なぜ壊れた状態から始めるのでしょうか。** generator が first draft を書くと正しくできてしまうかもしれず、その場合、観察する loop がありません。
間違っていると分かっているページは、毎回 first round で fail します。

```
python generator.py --seed
```

これにより page が 'workspace/index.html' にコピーされます。下流のすべてがこれを読みます。

### Step C: evidence and the evaluator (10 minutes)

generator が作成したページが本当に spec を満たしているか、何かが判断する必要があります。
それは 2 つの部分で行われます。まず script がページを測定し、次に Claude がその測定結果で十分かを判断します。

**'checks.py' を開いてください。** ここには Claude はありません。'workspace/index.html' を開き、ページ上のものを数える普通の Python です。

- **'evidence()'** は、見つかった 'data-testid' 名と、各 list に何 item あるかを報告します。
- 2 回実行すると、2 回とも同じ答えが返ります。generator にページを正しく作ったか尋ねると意見が返りますが、これは count を返します。

```
python checks.py
```

report は、どの test ids が存在するか、各 list に何 item あるか、ページが external scripts を読み込むかを一覧表示します。
Step A の spec と比較してください。flavor list は 3 items あるべきですが、1 つしかありません。

**'evaluator.py' を開いてください。** これは再び Claude ですが、新しい session で、2 つのものを受け取ります。spec と、いま実行した report です。

- HTML は見ず、generator が自分の作業について述べたことも見ません。author の説明を先に読む reviewer は、それに同意しがちです。
- **'VERDICT_SCHEMA'** は、loop が処理できる固定の形で、written critique とともに PASS または FAIL を答えさせます。

```
python evaluator.py
```

flavor list と missing order counter に対して FAIL し、何を変更すべきかを正確に述べた critique を書くはずです。
その critique が次の step で generator に渡されます。

### Step D: the whole loop (10 minutes)

**'run_loop.py' を開いてください。** これは短いです。いま実行した 3 つの scripts が作業を行うためです。
この script は、次に何が起こるかを決めます。

- **'MAX_SPRINTS'** は、3 rounds 後に停止させます。limit がないと、evaluator が決して accept しないページは永遠に loop してしまいます。

```
python run_loop.py
```

Sprint 1 は seeded draft を読み込み、fail します。Sprint 2 は critique を generator に渡し、generator がページを書き直します。
evaluator が再度チェックし、pass します。

**では、何が構築されたかを見てください。** 'workspace/index.html' をブラウザーで開きます。

![The finished kiosk page](images/05.1-kiosk.png)

- seed にあった単一行ではなく、すべての flavor がそれぞれ独立した row にある flavor list。3 つより多く見えることがあります。spec は下限を設定しているのであって、上限ではありません
- その下に示された special of the day
- button の横にある order counter。0 を示しています
- **Place Order** をクリックすると、count が 1 になります

'seeds/kiosk_buggy.html' を横に開いて、どこから始まったかを確認してください。flavor は 1 つで、counter はまったくありませんでした。

**これは mock であり、実際に動く till ではありません。** button は screen 上の number に 1 を足すだけで、それ以外は何もしません。
どこにも order は送られず、何も保存されず、MCP server や実際の shop への connection もありません。
loop が示したのは、spec に合わせて page を構築し、自分の mistake を検出して、誰もチェックしなくても修正できるということです。
本物の kiosk は次の仕事になり、同じ方法で構築されます。まず spec を書き、その後 loop にそれへ向けて作業させます。

> なぜこれが重要なのでしょうか。agent が review できる量を超える code を書くと、review が bottleneck になります。
> 修正策は writer の prompt を改善することではありません。独自の evidence を持つ別の judge です。
> loop の品質は常にその judge の品質次第なので、努力はそこに注ぎます。

**Checkpoint 8.** 仕込まれた bug が Claude evaluator によって検出され、Claude generator によって修正されました。human review はありません。

> 試してみましょう: seed を使う代わりに generator 自身に first draft を構築させるには、'python run_loop.py --fresh' を実行します。

---

## Module 2.2: 新鮮な research とより大きな toolbox (15 minutes)

Claude on Foundry の機能をさらに 2 つ使うと、loop の形を変えずに賢くできます。

### Fresh knowledge: web search

今朝、agent は shop が知っていること (Foundry IQ) を学びました。今度は world が知っていることを学びます。
**Web search** は Claude on Foundry に組み込まれています。

**まず 'websearch.py' を開いてください。** 'QUESTION' が尋ねる内容であり、編集できます。
Claude は search を実行し、server side で results を読みます。そのため script 自体が web page を fetch することはありません。

```
python websearch.py
```

Claude が検索し、いくつかの results を読み、bottom に listed sources とともに special を recommend します。
隣の人と比較してください。誰か違う結果になったか見てみましょう。

### Research before planning

今度は planner にも、spec を書く前に同じことをさせます。

```
python planner.py --research
```

research notes が先に表示され、その後 spec が表示されます。
flavors と special は live results から来るため、あなたの kiosk は隣の人のものと一致しません。
'python run_loop.py --research' は、full loop の中で同じことを行います。

### Tool search

Sparkles の tool catalog は増え続けています。すべての request にすべての tool を読み込むと context を消費し、model を混乱させます。
**tool search** では、tools に 'defer_loading' を付け、Claude が必要なものを検索します。

**'toolsearch.py'** を開き、**'CATALOG'** を見てください。12 個の tools があり、それぞれ name と one-line description を持っています。

- 12 個すべてを毎回送ると context を使い、model に選ばせる間違った options が増えます。
- **'defer_loading'** はそれらを後回しにします。Claude は descriptions を検索し、question に必要な tools だけを読み込みます。

```
python toolsearch.py "How many loyalty points does Priya have?"
```

output は 3 行を表示します。Claude が何を検索したか、tool search がどの tools を返したか、そしてどれを call したかです。
別の question、たとえば 'Is the shop open on Sunday?' を試し、別の tool を選ぶ様子を見てください。

**Checkpoint 9.** live citations 付きの recommendation、research に基づいた schema-valid spec、そして 12 個のうち正しい tool を context にすべて持たずに見つける agent です。

---

## Module 2.3: Foundry observability ですべての実行内容を見る (20 minutes)

いま実行した loop は、6 回ほど model calls を行いました。どの agent が最も時間を取りましたか。
fast model は smart model と比べて何 tokens 使いましたか。evaluator の score は round ごとに実際に改善しましたか。
Application Insights は、数行の code からそれらすべてに答えます。

Note: Foundry portal の Traces tab は、Foundry で hosted されている agents を表示します。
この script は laptop 上で動き、Claude を直接呼び出すため、その traces は Azure portal の Application Insights に保存されます。
これは Claude on Foundry を使う external application にとって通常の path です。

### Find your Application Insights resource

すでに持っているかもしれません。Foundry project を作成すると、Application Insights resource が一緒に作成されることがあります。
別のものを作る前に確認してください。

```
az monitor app-insights component show --query "[].{name:name, rg:resourceGroup}" -o table
```

**ちょうど 1 つ返ってきた場合** は、その connection string を読み取ります。

```
az monitor app-insights component show --query "[0].connectionString" -o tsv
```

**複数返ってきた場合** は、Foundry project と同じ resource group にあるものを選び、名前を指定します。
両方の値を自分のものに置き換えてください。

```
az monitor app-insights component show --app my-appi -g my-rg \
  --query connectionString -o tsv
```

**何も返ってこなかった場合** は、Foundry project がある resource group に 1 つ作成します。
両方の値を自分のものに置き換えてください。

```
az extension add --name application-insights --upgrade

az monitor app-insights component create \
  --app my-appi -g my-rg -l eastus --application-type web

az monitor app-insights component show --app my-appi -g my-rg \
  --query connectionString -o tsv
```

connection string は 'InstrumentationKey=' で始まる 1 行の長い文字列です。
次に必要になる value はこれです。

### Turn tracing on

'.env' で次を設定します。

```
ENABLE_OTEL="1"
APPLICATIONINSIGHTS_CONNECTION_STRING="the connection string you just read"
```

変更するのはこれだけです。scripts は残りをすでに行います。
'sparkles-loop/common.py' にあります。これは、Claude client、model names、tracing のために 3 つすべてが import する shared file です。

- **'setup_tracing()'** は loop の開始時に実行されます。これら 2 つの settings を確認し、どちらかが missing であれば理由を表示して tracing なしで続行します。
- **'span'** は、3 つの scripts が Claude への各 call の周囲に置く小さな wrapper です。timer を開始し、どの model が実行されたか、何 tokens が入出力されたかを記録し、call が戻ると閉じます。
- 各 span は、それを開いた script にちなんで 'planner'、'generator'、または 'evaluator' と named されます。これらが portal で探す names です。
- **'session_span()' and 'run_span()'** は trace に shape を与えます。run 全体を囲む span が 1 つあり、その中に各 round を囲む span が 1 つずつあります。

#### Step 1: Run the loop again

```
python run_loop.py
```

output の最初の行は `Tracing on: spans go to Application
Insights.` であるはずです。

run 全体が 1 つの trace になります。planner が最初にあり、各 round がその下に nested されます。
`POST` rows は HTTP client から自動的に captured されます。残りは `common.py` の `session_span()`、`run_span()`、`span()` から来ます。

```
sparkles-session
  planner          -> POST /anthropic/v1/messages
  sparkles-run (round 1)
    generator      -> POST /anthropic/v1/messages
    evaluator      -> POST /anthropic/v1/messages
  sparkles-run (round 2)
    generator      -> POST /anthropic/v1/messages
    evaluator      -> POST /anthropic/v1/messages
```

planner は rounds の外に置かれます。どの round より前に 1 回だけ実行されるからです。

loop が実行されている間に `common.py` を見て、`span` class を見つけてください。
すべての agent call に付加される 3 つのものに注目してください。model、token counts、そして evaluator の score など loop が `sp.set(...)` に渡すすべてのものです。

traces が portal に表示されるまで 2〜5 分かかります。
loop が終了し、数分経ってから Step 2 に進んでください。

#### Step 2: Look at the run as a trace

1. Application Insights resource で **Investigate > Search** を開きます。
2. time range を **Last 30 minutes** に設定します。
3. **View as traces** を選択します。各 `sparkles-session` card は script の 1 回の run です。何かを開く前に card header を見てください。run の duration、spans の数、run 全体の token badge (たとえば `12,400t`) が表示されます。Azure は `gen_ai.usage.*` attributes を読み取り、合計してくれます。
4. card の header line (下の "Matching Dependency" box ではなく、trace ID と `sparkles-session` name) をクリックします。end-to-end transaction page が timeline として開きます。上部に planner、その下に各 round、さらにその中に generator と evaluator、そしてそれぞれの下に実際の Claude call があります。
5. その timeline で、**evaluator** bar (下の POST ではありません) をクリックします。右側の panel に、その span の properties が表示されます。model、`gen_ai.usage.input_tokens`、`gen_ai.usage.output_tokens`、そして loop が recorded した `sparkles.*` values です。panel が短く見える場合は、そこにある "show all" または "leave simple view" link を探してください。

![Investigate > Search, cards](images/06-search-cards.png)

![Investigate > Search, one run open](images/07-search-trace.png)

この view から答える questions:

- どの agent が最も遅いですか。予想したものですか。
- generator の tokens と planner の tokens を比較してください。なぜ generator は fast model 上にあるのでしょうか。
- 各 round の evaluator を順に開いてください。score は上がっていますか。

#### Step 3: Query across rounds

1. **Monitoring > Logs** を開きます。2 つの settings により、KQL editor にまっすぐ進めます。どちらも account に保存されます。
   - **Queries hub** dialog で **Always show Queries hub** を off にし、X で閉じます。
   - page の右上にある **Agent** toggle を off にします。

   > Optional, before you switch the Agent off: **Observability Agent** は KQL を書いてくれます。
   > "show me evaluator spans with their
   > sparkles.score by round" と尋ね、生成されたものを下の queries と比較してください。
2. 次を editor に貼り付け、**Run** を選択します。agent と model ごとの cost picture が得られます。

   ```kusto
   dependencies
   | where timestamp > ago(1h)
   | where name in ("planner", "generator", "evaluator")
   | summarize calls = count(),
       avg_seconds = round(avg(duration) / 1000, 1),
       input_tokens = sum(toint(customDimensions["gen_ai.usage.input_tokens"])),
       output_tokens = sum(toint(customDimensions["gen_ai.usage.output_tokens"]))
       by name, model = tostring(customDimensions["gen_ai.request.model"])
   ```

3. 次は、この lab 全体の問いです。loop は改善しましたか。これを実行し、自動的に chart が render されない場合は result を **Chart** に切り替えます。

   ```kusto
   dependencies
   | where timestamp > ago(1h)
   | where name == "evaluator"
   | extend score = todouble(customDimensions["sparkles.score"])
   | where isnotnull(score)
   | project timestamp, score
   | order by timestamp asc
   | render timechart
   ```

   `sparkles.score` は evaluator が pass した acceptance criteria の数で、`sparkles.criteria` はその総数です。そのため、ある問題を 1 つ修正した round は 4 分の 3 から 4 分の 4 に進みます。
   evaluator は verdict を parse した直後に、`evaluator.py` で両方を設定します。

   上昇する line は、evaluator が generator に改善を強制していることを示します。flat line は、criteria が簡単すぎるか、feedback が generator に届いていないことを意味します。

![Trace tree and the score chart](images/08-trace-tree-chart.png)

**Checkpoint 10.** planner、generator、evaluator の run が Application Insights に表示され、最も遅い span を名指しできます。

traces は agent が何をしたかを教えてくれます。それが良かったかどうかは教えてくれません。
それが最後の module です。

---

## Module 2.4: Claude によって judged される evaluations (25 minutes)

Module 2.1 では、judge を手作業で構築しました。Foundry には組み込み版があります。
**evaluations** です。evaluator を登録し、runs の dataset を指し示すと、portal がすべての row を採点し、history を保持します。
この module では 2 つの evaluators を登録します。1 つは model をまったく使わず、もう 1 つは Claude が judge になるものです。そして 4 つの実際の Sparkles runs を採点します。

> Foundry には、built-in AI-assisted evaluators (relevance、task
> adherence など) も付属しており、それらには独自の judge model があります。
> ここでは custom evaluators を使います。Claude を judge にするためであり、また、自分たちのものをどう持ち込むかを見るためです。これは多くの teams が最終的に行うことです。

### Setup

```
cd sparkles-evals
az login
```

'.env' に 'AZURE_AI_PROJECT_ENDPOINT' と 'EVAL_ENDPOINT_CONNECTION' があることを確認してください。
どちらも [SETUP.md](SETUP.md) step 5 で設定済みです。

'sample_runs.jsonl' を見てください。4 rows あり、それぞれが保存された Sparkles run です。
4 つの fields があります。customer が尋ねたこと、agent が返答したこと、その run が生成した kiosk page の static report、そして印刷した receipt です。

**2 つの evaluators は、各 row の別々の半分を読みます。** Step A の code-based evaluator は 'report' と 'receipt' だけを見ます。
Step B の Claude judge は 'query' と 'response' だけを見ます。どちらも相手の evidence は見ません。だから同じ run について意見が分かれることがあります。

| Row | The conversation | The kiosk page | The receipt | Planted problem |
|---|---|---|---|---|
| 1 | Party order: 50 cupcakes, over budget, nut allergies, hazelnut. agent は total、budget、allergy、bulk-order rules を検出します | 5 つすべての elements が存在 | valid | なし: good な状態の見本です |
| 2 | Two chocolate cupcakes. agent は stock を確認し、ordering 前に tree-nut policy を指摘します | **order counter が missing** | valid | 壊れた page |
| 3 | What flavors do you have today? agent は flavors を listed しますが、その一部は shop で売っていません | **special of the day が missing** で、page が別 site から script を読み込みます | **'not an order'** なので parse するものがありません | 壊れた page と間違った answer |
| 4 | Cupcakes arrived crushed, can I get a refund? agent は all sales are final と言います | 5 つすべての elements が存在 | valid | **answer が wrong**: shop の policy は damaged orders に refund を与えます |

refund row は注意しておくべきものです。page や receipt には何も
wrong なところがないため、code-based evaluator は objection する材料がありません。
唯一 wrong なのは、agent が customer に伝えた内容です。

### Step A: a code-based evaluator (10 minutes)

'grade_sparkles.py' を開いてください。これは普通の Python の 'grade()' function です。
score の 60 percent は kiosk report に required test ids が存在するか、40 percent は receipt が parse でき、すべての required key を持っているかに割り当てます。
model は関与しません。

2 つの scripts があり、それぞれ違うことをします。

**`register_code_evaluator.py`** は `grade_sparkles.py` を Foundry
project に upload し、name を付けます。登録は実行ではありません。project に「ここに使える evaluator があります」と伝え、portal に表示され、後で任意の dataset に向けられるようにするものです。
これは 1 回だけ行います。

**`run_cloud_eval.py code`** は evaluation を開始します。
`sample_runs.jsonl` を読み、rows と evaluator name を project に渡し、Foundry がすべての row を採点する間待ちます。
作業は cloud で行われ、machine 上では行われません。そのため結果は terminal output ではなく report URL です。

```
python register_code_evaluator.py
python run_cloud_eval.py code
```

run は report URL を表示します。portal で開いてください。

![Code evaluator report](images/09-eval-code-report.png)

**score の計算方法。** `grade_sparkles.py` は各 row に 0 から 1 までの number を与えます。2 つの parts があります。

- **kiosk に 0.6**。5 つの required test ids に分割されます。存在するものはそれぞれ 0.12 の価値があります。
- **receipt に 0.4**。all or nothing です。JSON として parse でき、すべての required key を持ち、total が 0 より大きい必要があります。

**threshold は score とは別です。** `run_cloud_eval.py` で 0.9 に設定されており、pass が fail に変わる境界を決めます。
score は変更しません。線を動かすだけです。0.9 では、row はすべての test id と valid receipt の両方を持たなければならず、これは Module 2.1 の loop が enforced した standard と同じです。

**表示されるはずのもの:**

| Row | Score | Why | |
|---|---|---|---|
| 1 party order | 1.00 | page に 5 つすべての elements があり、receipt が parse できる | pass |
| 2 chocolate | 0.88 | `order-count` が missing で、0.12 失います。Receipt は fine | **fail** |
| 3 flavors | 0.48 | `special` が missing で、`not an order` は parse できず、0.4 全体を失います | **fail** |
| 4 refund | 1.00 | structurally wrong なものはありません | pass |

4 つ中 2 つなので、report は 50% と表示します。

chocolate row は立ち止まる価値があります。order counter が missing で、Module 2.1 で evaluator が検出したのと同じ fault ですが、それでも 0.88 を得ます。
threshold を 0.5 に設定すると、この kiosk は ship されます。weighted score は、missing element 1 つを rounding error のように見せます。

そして refund row は、customer に refund policy について false なことを伝えながら 1.00 で pass します。
checks は answer を決して読まないため、object するものがありません。

### Step B: an endpoint-based evaluator, Claude as judge (10 minutes)

rules では採点できないものがあります。agent は customer に refund policy について真実を伝えましたか。
static check では答えられません。そのためには、agent が読むべき同じ policy document を読んだ model が必要です。

[SETUP.md](SETUP.md) step 5 で、小さな service を deploy しました (repo の 'eval-endpoint/' を参照)。
これは各 row を受け取り、Claude deployment に rubric に照らして response を採点させ、score と one-sentence reason を返します。
judge には Foundry IQ knowledge base に供給されるのと同じ store document が与えられるため、policy claims を source と照合できます。
Foundry は project connection を通してそれを call します。

> **まだ deploy していませんか。** 約 20 分見込んでください。その大半は container
> build と deploy です。別の terminal で開始し、実行中は読み進めてください。

Step A と同じ 2 つの scripts ですが、Python file ではなく endpoint を指します。
**`register_endpoint_evaluator.py`** は、project 内で code を実行するのではなく、connection を通じて endpoint を呼び出す evaluator を登録します。
**`run_cloud_eval.py endpoint`** は同じ 4 rows をそれで採点するため、同一 data に対する 2 つの judges を比較できます。

何かを実行する前に endpoint が動作していることを確認してください。これが Step B が失敗する最も一般的な理由です。

```
curl $(grep EVAL_ENDPOINT_URL ../.env | cut -d'"' -f2 | sed 's|/evaluate|/health|')
```

model name が返ることを期待します。たとえば
`{"ok":true,"model":"claude-sonnet-5"}` です。empty model は、endpoint が Claude に到達できず、すべての row が score ではなく **Error** として返ることを意味します。
先に進む前に、[eval-endpoint/README.md](../eval-endpoint/README.md) の Troubleshooting section で修正してください。

```
python register_endpoint_evaluator.py
python run_cloud_eval.py endpoint
```

report を開いてください。各 row は score と **reason** を持ち、reason は response についての Claude 自身の sentence です。

![Claude-judged report](images/10-eval-llm-report.png)

> **あなたの numbers はこれらと正確には一致しないかもしれません。** model judge は deterministic ではありません。同じ row を 2 回実行すると score が動くことがあり、threshold 近くの row はどちら側にもなり得ます。
> party order は、正しく判断すべきことが多いため、最も違いが出やすいものです。row の score が異なる場合は、number ではなく reason を読んでください。そこに judge が実際に何に objection したかが示されています。
>
> production でこれをより安定させる必要がある場合、lever は、より capable な judge model、解釈の余地が少ない rubric、または各 row を 2〜3 回採点して median を取ることです。3 つとも row あたりの cost が増えます。それが trade です。

Step A report と並べてください。

| Row | Code evaluator | Claude judge | |
|---|---|---|---|
| 1 party order | pass, 1.00 | pass | kiosk page と receipt は complete で、answer は right |
| 2 two chocolate | **fail, 0.88** | pass, 0.90 | page は order counter を missing していますが、agent の answer は良好でした |
| 3 flavors | fail, 0.48 | fail, 0.30 | page と receipt が broken で、agent は shop にない flavors を listed しました |
| 4 refund | pass, 1.00 | **fail, 0.00** | page は fine ですが、agent は wrong refund policy を伝えました |

すべての組み合わせが表れています。両方が accept する 1 row、片方だけが objection するものがそれぞれ 1 row、そして両方が reject する 1 row です。

**これらは同じものについての 2 つの opinions ではありません。** それぞれが run の異なる部分を見ています。

- code evaluator は **kiosk page と receipt** をチェックします。5 つの elements が page にあるか、receipt が parse できるかです
- Claude judge は **agent が customer に言ったこと** をチェックします。それが true で、store policy と一致しているかです

したがって、run は両方が pass したときに good であり、片方が fail したときには、どの部分を fix すればよいかが分かります。

- **The party order** — 両方 pass です。fix するものはありません。
- **The chocolate order** — page を fix します。order counter が missing です。agent が言ったことは fine でした。
- **The flavors question** — 両方を fix します。page は special of the day が missing で、別 site から script を読み込み、agent は shop が売っていない flavors を listed しました。
- **The refund** — page は fine です。agent は customer に all sales are final と伝えましたが、shop の policy は damaged orders に refund を与えます。agent を fix します。

refund row は覚えておくべきものです。page や receipt には何も wrong なところがないため、static check では検出できません。
見つける唯一の方法は、answer を policy と照らして読む何かを用意することです。

### Step C: what you just did (5 minutes)

3 つの judges、同じ idea です。

1. Module 2.1: loop の中で手動で実行した Claude evaluator
2. Step A: rule-based checks。Foundry に登録され、scale して実行され、history が保持されます
3. Step B: Claude as judge。自分が所有する endpoint の背後にあり、Foundry の evaluation service 内で動きます

ここから platform が引き継ぎます。同じ evaluators を sampled production traffic に対して継続的に実行できるため、「agent はまだ良いのか」という question が、誰も transcripts を読まなくても毎日 answer されます。

**Checkpoint 11.** 2 つの evaluators が登録され、同じ 4 つの sample
conversations がそれぞれによって Foundry portal で採点され、results に Claude の written reasoning が含まれます。

### Wrap up

今朝は agent を構築しました。午後には、自分の work を check させ、searchable toolbox を与え、portal ですべての step を観察し、evaluations で採点しました。
評価 (evaluations) のない autonomy は希望にすぎません。評価のある autonomy は engineering です。
Show and Tell に kiosk を持っていきましょう。
