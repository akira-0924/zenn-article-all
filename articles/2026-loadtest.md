---
title: '負荷試験をやったことない4年目エンジニアがk6で実施するまで'
emoji: "🔧"
type: "idea"
topics: [負荷試験, k6]
published: false
---

## はじめに

業務で、本番リリースに向けた準備の一環として **負荷試験を初めて担当** しました。k6 自体も試験設計もほぼ未経験で、正直なところ「何から手をつければいいのか」が分からない状態からのスタートでした。

一般的な負荷試験のベストプラクティスやこうあるべき、みたいな書籍・記事はありふれていると思うので、この記事では「負荷試験やることになったけど何から手をつければいいかわからない」「どう進めるかわからない」など完全初心者に向けてその第一歩を後押しするような内容をより具体的に書ければと思います。

負荷試験をこれからやる方、何からやれば良いかわからない方、名前だけ聞いたことがあって距離を置きがちだった方の参考になれば嬉しいです。

---

## 今回のプロダクトの概要

今回負荷試験を行うプロダクトはLLMが絡んだ、「AI質問サービス」です。ユーザーは問題や答案の画像を撮影・アップロードし、ChatGPTのような形でAIに投げかけます。AIがヒント・解説を生成し、わかりやすく解説してくれる、というプロダクトになっています。
内部処理の細かい部分までは公開できませんが、今回の記事を理解するにあたって、最低限必要な機能として、
1. 「質問を投げる」APIと
2. 「質問されたものをLLMに投げかけて生成してもらい諸々の処理をして生成結果を含むレスポンス返す」API

の2つがあります。LLMの生成による待機時間を短くするためにストリーミングによる実現の例もありますが、このプロダクトはストリーミングではなく、質問を投げた後は非同期処理で生成が続き、クライアントからは一定間隔でポーリングを行い、生成結果があればそれを返す、なければ処理のステータスのみを返す、というような実装になっています。

また今回のプロダクトのインフラ構成を簡略化したものが下記になります。

![](https://static.zenn.studio/user-upload/63cc524f6afe-20260513.png)

CloudFront（Lambda@Edge 付き）が入口で、S3 から SPA と画像を配信し、API は VPC 内の API Lambda（Docker）に届きます。API は SQS のメインキューに処理を渡し、Worker Lambda（Docker）が プロダクションコードを実行し LLM 処理を行います。永続データは RDS Proxy 経由の Aurora Serverless v2になります。

## 進め方

負荷試験といってもそのタスクが完了となるまで何をどう進めて行けば良いのか、そこからです。
自分の場合、社内のSREチームがドキュメントを作成してくれていたので、それも参考にしながら下記のように進めました。

1. キャパシティプランニング
2. テストの目的を決める
3. 負荷をかけるAPI、シナリオを決める
4. 合格ラインを決める
5. 目的に適したテストの種類を考える
6. 負荷テストコード実装
7. サーバー側インフラの設定(環境作成)
8. テスト実行・計測
9. 評価・修正

以下、同じ順番でセクションを分けて書いていきます。

---

## 1. キャパシティプランニング

まずは「どのくらいの負荷がかかりうるのか」を把握するために数値に落としました。次のような情報を収集します。

1. **これまでの実トラフィック（あれば）や利用状況**
1日の合計アクセス数、月間平均、どの時間帯にアクセスが多いか、それは通常時の何倍ぐらいなのか 

2. **リリース後の想定ユーザー数・同時接続の目安**
1からわかる実際の利用状況から「どれくらいの期間でどれくらいのアクセスになるか」をざっくりと見積もる(とは言っても難しい部分もあるので、ある程度決め打ちでざっくりこのぐらいかなあを根拠を織り交ぜて出せれば良いかなと思います)

今回は、トライアルリリースを済ませていたので、その利用状況(DB接続状況や質問数)と現状のリクエスト数から「数ヶ月後、将来、これくらいのユーザー数、利用状況になることが想定できるよね」を出し、実際に数値化しました。ここは自分一人で全部進めても良いですが、チーム・有識者と相談しながらこの想定で大方問題なさそうかを話せると良いと思います。
具体的に今回のプロダクトに当てはめると、
前述した通り、「AI質問」プロダクトなので'質問する'という機能が今回のメインになります。その'質問'を利用(アクティブ)ユーザー
1. 1,000人
2. 5,000人
3. 10,000人

の3段階で何回なのかを出します。DBにあるデータを元に1ユーザーは1ヶ月で平均20回質問するだろう、と仮定できたので、
- 1,000人 → 20,000質問(1日666質問)
- 5,000人 → 100,000質問(1日3333質問)
- 10,000人 → 200,000質問(1日6666質問)

このぐらいざっくり出しました。

ここで粗くてもよいので **「試験で再現したい規模感」** が一文で説明できる状態を目指すと、その後のステップが迷いにくくなりました。


## 2. テストの目的を決める

キャパシティの数字が出たら、次は「**負荷試験で何を達成・担保したいか**」を明確にします。目的なしに実施してもなんとなくで終わってしまうのでここはしっかり定義しておきます。

これもいくつかあるかと思いますが、今回は、
「**キャパシティプランニングで見立てたアクセス数に基づき、正式リリース後の通常時/ピークタイムにシステムが想定通り動作することを担保する**」

を目的としました。他にも、ボトルネックを特定する、とか、性能を測る、とかもありかと思います。これはそもそも何で負荷試験をやろうとしているのかをそのまま目的にすれば良いです。
メインの目的を決めた上で、具体的に
- キャパシティプランニングの裏付けとして、1000人/5000人/10000人想定でサービスが正常に動作することを確認する
- 想定されるピークタイムのトラフィックを捌ける
- リソース使用率が適切な範囲内であることを確認する（CPU 50-70%, メモリ 50-60%）
- エラー率が許容範囲内であることを確認する
- ボトルネックの特定と対策

などをサブとして決めておきました。「どうやって負荷試験やるかわからないけどこれはわかるようになっておきたい」をこの段階で決めておくと楽です。

「ボトルネックを探す」のか「ピークに耐えればよい」のか、目的が違うとシナリオも合格ラインも変わります。ここは序盤にしっかり決めておきましょう。

## 3. 負荷をかけるAPI、シナリオを決める

次に実際の負荷のかけ方を決めます。ここで大事なことは「**まずはシンプルな負荷テストを心がける**」ことです。特に、今回が人生初めての負荷試験であった自分がいきなり複雑なシナリオをテストしようとしても大変だし、プロダクトとしても負荷試験を過去にやったことがないのでぶっちゃけ負荷傾向も分かりません。単一の負荷テストを動かすだけも、想定してない環境整備や設定が起こりうるものなので、まずは負荷テストを動かすことを考えます。

今回の場合は、「質問を投げる」APIという単一のエンドポイントに対してまずは負荷をかけることを検討しました。
結果として、よりリアルな試験を行うために「質問を投げる」APIを叩き、「生成結果を取得する」APIを一定間隔でポーリングするという、ここでいうシナリオベースのテストをすることになりました。(がそもそもがちょっと特殊なプロダクトだと思うので基本はやはり単一エンドポイントのテストからで十分かなと思います)


## 4. 合格ラインを決める

続いてどの状態になっていれば合格かを細かく決めておきます。2の具体的な目的で正常に動作する〜とか、エラー率が許容範囲内〜など書いていますが、数値してどこまでならOKなのかを決める工程です。SLIというやつですね。
SLO、SLAなどが要件として決まっているのであれば、やりやすい気もしますが、今回は特になかったので、一般的にUXを損なわない許容レベルを基準に決定しました。以下の感じです。

- APIレイテンシ
  - p95 < 2秒
- エラー率
  - HTTP 5xxエラー率: < 0.5%（1000リクエスト中5件未満）
  - HTTP 4xxエラー率: < 2%（認証エラー等のユーザー起因を除く ex. 429）
  - タイムアウトエラー: 0件
- スループット
  - キャパシティプランニングのピーク時（5分間）を維持できること
  - 1,000人: 100質問/5分 = 20質問/分 = 0.33req/秒
  - 5,000人: 500質問/5分 = 100質問/分 = 1.67req/秒
  - 10,000人: 1,000質問/5分 = 200質問/分 = 3.33req/秒

- リソース使用率
  - Lambda(API, Worker) メモリ使用率: 50-60%
  - Worker LambdaはLLM呼び出しをモックしているのであくまで参考値
  - Aurora ACU使用率: ピーク時 90%以下（8ACUの場合は7.2ACU以下）
  - RDS Proxy接続数: 最大接続数(2,000)の80%以下
  - SQSの最古メッセージの滞留時間が5分以内
  - DLQ転送0件
  - 負荷試験終了後に滞留メッセージ数が減少していくこと

ここら辺はSREチーム方にもみてもらったりドキュメントを参考にしながら決めました。結構一般論的な部分が多い気がしていたので、自分の場合はLLMと壁打ちしながらこのぐらいかなあみたいな叩きを作ってから詳細に詰めていきました。



## 5. 目的に適したテストの種類を考える

続いて実際にどんな負荷テストを行うか、種類を選定します。これは目的が決まっていればあまり悩むことはないかなと思います。
種類については下記が参考になります。

https://grafana.com/docs/k6/latest/testing-guides/test-types/


今回は、まず動くこと、キャパシティプランニングの数値を捌けること、がメインなのでsmoke test,
load testを行うことに決めました。ボトルネック、性能限界までみたい場合はbreakpoint testまでやれると良いかなと思います。

| 種類 | 役割（今回の位置づけ） |
| --- | --- |
| Smoke | スクリプトと環境の疎通。単一リクエスト〜極小負荷で壊れ方を確認 |
| Load | 想定通常／ピークに近い負荷で、レイテンシ・エラー率・スループットを見る |


## 6. 負荷テストコード実装

ここまでで準備整ったのでいよいよ実装に入ります。ツールは今回**k6** を採用しました。(社内推奨、知見蓄積という観点でほぼ脳死で決めてしまったのでツール比較まではこの記事では書きません！)

https://grafana.com/docs/k6/latest/get-started/running-k6/

Macの場合
```bash
brew install k6
```
で入ります。

### 構成

今回は下記のようなディレクトリ構成にしてみました。

```text
apps/loadtest/
├── scripts/              # シェルスクリプト
│   ├── run-k6.sh         # k6実行用ラッパースクリプト
│   └── list-scenarios.sh  # 利用可能なシナリオ一覧を表示
├── tests/                 # テストの種類と設定値
│   ├── smoke-test.ts     # Smoke Test
│   ├── load-test.ts      # Load Test
│   └── breakpoint-test.ts # Breakpoint Test
├── scenarios/            # テストシナリオ定義
│   ├── problem-then-question.ts   # POST '質問' → GET '生成結果' ポーリングのシナリオ
│   └── index.ts          # シナリオの一覧と型定義
├── utils/                # 共通ユーティリティ
│   ├── auth-client.ts    # 認証クライアント
│   ├── http-client.ts   # HTTPクライアント（送信・レスポンス処理・ヘッダー生成）
│   ├── setup.ts          # セットアップ関数
│   └── types.ts          # 共通型定義
└── package.json
```

`/tests`に種類ごとの設定、`/scenarios`にテストシナリオを定義しています。

### スクリプトについて

`/scripts`には実行するためのスクリプトを作成して`npm run smoke --scenario=xxx`のような形で実行できるようになっています。

```json
{
  "scripts": {
    "scenarios": "bash scripts/list-scenarios.sh",
    "smoke": "bash scripts/run-k6.sh run tests/smoke-test.ts",
    "load": "bash scripts/run-k6.sh run tests/load-test.ts",
    "breakpoint": "bash scripts/run-k6.sh run tests/breakpoint-test.ts",
    "lint": "eslint",
    "lint:fix": "npm run lint -- --fix",
    "typecheck": "tsc --noEmit"
  }
}
```

```bash
# run-k6.sh
# npm run smoke --scenario=xxx の場合、npm が npm_config_scenario を渡す
SCENARIO_NAME="${npm_config_scenario:-}"
K6_ARGS=()

while [[ $# -gt 0 ]]; do
  case $1 in
    --scenario=*)
      SCENARIO_NAME="${1#*=}"
      shift
      ;;
    --scenario)
      SCENARIO_NAME="$2"
      shift 2
      ;;
    *)
      K6_ARGS+=("$1")
      shift
      ;;
  esac
done

if [ -z "$SCENARIO_NAME" ]; then
  echo "❌ エラー: --scenario オプションが指定されていません"
  echo ""
  echo "利用可能なシナリオを確認するには:"
  echo "  npm run scenarios"
  echo ""
  echo "使用例:"
  echo "  npm run smoke --scenario=byoProblemThenQuestion"
  echo ""
  exit 1
fi

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOADTEST_DIR="$(cd "$SCRIPT_DIR/.." && pwd)"

ENV_FILE="$LOADTEST_DIR/.env.local"

if [ -f "$ENV_FILE" ]; then
  echo "📝 Loading environment variables from .env.local"
  set -a
  source "$ENV_FILE"
  set +a
else
  echo "⚠️  Warning: .env.local not found. Using environment variables from shell."
fi

export SCENARIO="$SCENARIO_NAME"

# k6を実行（残りの引数をすべて渡す）
exec k6 "${K6_ARGS[@]}"
```

### テストの種類と設定値(`/tests`)
#### smoke test

```typescript
import { setup } from '../utils/setup.ts';
import { runTest } from '../utils/runners/test-runner.ts';

export { setup };

/**
 * Smoke Test
 *
 * 目的: テストスクリプトと環境の正常性確認
 * - ごく少数のユーザー（1-5 VU）で短時間実行
 * - エラー率0%を確認
 */
export const options = {
  vus: 2, // 2ユーザー（同時ログイン判定の確認用）
  iterations: 2, // 各VUが1回ずつ実行（合計2回）
  thresholds: {
    // HTTP 5xxエラー率: 0%（smoke testは厳格にチェック）
    'http_req_failed{status:>=500}': ['rate==0'],
    // HTTP 4xxエラー率: 0%（smoke testは厳格にチェック）
    'http_req_failed{status:>=400,status:<500}': ['rate==0'],
    // タイムアウトエラー: 0件
    'http_req_failed{error:timeout}': ['rate==0'],
    // レスポンス時間: p95 < 2秒
    'http_req_duration{type:api}': ['p(95)<2000'],
  },
};

export default runTest;
```

ポイントの整理

`export { setup }`
- k6 のライフサイクルで テスト本体が動く前に 1 回だけ 実行される setup 関数です。ここで読んだ値は default 関数（このプロジェクトでは runTest）の引数として渡されます。

https://grafana.com/docs/k6/latest/using-k6/test-lifecycle/

`export default runTest`
1 イテレーションごとに実行される本体です。SCENARIO環境変数でどのシナリオを回すか切り替え、シナリオ定義に沿ってHTTPを叩きます。

`vus: 2 / iterations: 2`
同時ユーザー数とイテレーション数の組み合わせで、極小負荷・短時間に抑えています。コメントのとおり、内部的にはshared-iterationsに相当する動きになるので、細かい振る舞いはexecutorの文档が参考になります。

https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/shared-iterations/

thresholds
この実行が「成功」とみなせるかを数値で縛ります。一応自分で決めた値がありますが、k6の場合はここで負荷テストの結果から成功かどうかを判定してくれるみたいです。
細かい設定は下記を参照してください。

https://grafana.com/docs/k6/latest/using-k6/thresholds/


#### load test
options.scenarios で 複数の負荷プロファイルを 1 回の実行に並べています。constant-arrival-rateというexecutorを使用します。

https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/

```typescript
import { setup } from '../utils/setup.ts';
import { runTest } from '../utils/runners/test-runner.ts';
import type { ScenarioName } from '../scenarios/index.ts';

export { setup };

/**
 * Load Test
 *
 * 目的: 通常時やピーク時に想定されるアクセス数をかける
 * - 1000人シナリオ: 0.33 req/s、10分間
 * - 5000人シナリオ: 1.67 req/s、10分間
 * - 10000人シナリオ: 3.33 req/s、10分間
 * - 実行時間: 30分 + 待機時間
 */
const scenarioName = __ENV.SCENARIO as ScenarioName;
// 前のシナリオ終了後の待機時間（分）
// problemThenQuestion: シナリオ間でLLM処理待機（15分）
const startTimeDelayAfterPreviousScenario = scenarioName === 'problemThenQuestion' ? 15 : 0;

const createScenarios = () => {
  const DURATION = 10;
  // 5000-usersは1000-users終了後 + 待機時間後に開始
  const startTime5000Users = DURATION + startTimeDelayAfterPreviousScenario;
  // 10000-usersは5000-users終了後 + 待機時間後に開始
  const startTime10000Users = startTime5000Users + DURATION + startTimeDelayAfterPreviousScenario;

  const baseScenarios = {
    '1000-users': {
      executor: 'constant-arrival-rate',
      rate: 1,
      timeUnit: '3s', // 0.33 req/s
      duration: `${DURATION}m`,
      preAllocatedVUs: 10,
      maxVUs: 100,
    },
    '5000-users': {
      executor: 'constant-arrival-rate',
      rate: 5,
      timeUnit: '3s', // 約1.67 req/s
      duration: `${DURATION}m`,
      preAllocatedVUs: 20,
      maxVUs: 500,
      startTime: `${startTime5000Users}m`,
    },
    '10000-users': {
      executor: 'constant-arrival-rate',
      rate: 10,
      timeUnit: '3s', // 約3.33 req/s
      duration: `${DURATION}m`,
      preAllocatedVUs: 50,
      maxVUs: 1000,
      startTime: `${startTime10000Users}m`,
    },
  };

  return baseScenarios;
};

export const options = {
  scenarios: createScenarios(),
  thresholds: {
    // HTTP 5xxエラー率: < 0.5%
    'http_req_failed{status:>=500}': ['rate<0.005'],
    // HTTP 4xxエラー率: < 2%（429を含む）
    'http_req_failed{status:>=400,status:<500}': ['rate<0.02'],
    // タイムアウトエラー: 0件
    'http_req_failed{error:timeout}': ['rate==0'],
    // レスポンス時間: p95 < 2秒
    'http_req_duration{type:api}': ['p(95)<2000'],
  },
};

export default runTest;
```

constant-arrival-rate
「単位時間あたりに何回イテレーションを開始するか」をおおよそ一定に保ちます。VU の思考時間やレスポンスが長くても、目標レートを維持するために VU 数が増減します（上限は maxVUs）。スモークのように「合計イテレーション数」を決めるのではなく、キャパシティ試算で出した req/s に寄せるのに向いています。

rate と timeUnit
公式どおり「timeUnit あたり rate 回イテレーションを開始する」イメージです。ここでは timeUnit: '3s' なので、例えば rate: 1 は 3 秒に 1 回開始 → 約 0.33 回/秒、rate: 5 は 3 秒に 5 回 → 約 1.67 回/秒、rate: 10 は 約 3.33 回/秒 になります（コメントの数値と対応）。

duration
各シナリオは 10 分間、そのレートを維持します。3 段あるので 負荷をかけ続けている時間の合計は 30 分です。

preAllocatedVUs / maxVUs
事前確保する VU と、k6 がレート維持のために増やせる上限です。レートや応答時間が上がるほど VU が必要になるので、**途中で「VU が足りずに目標レートに届かない」**とならないよう、だいたいの余裕を見て決めます（公式の executor ページに挙動の説明があります）。

複数シナリオと startTime
1000-users → 5000-users → 10000-users を 同じ export default runTest のまま順番に実行するために、5000-users と 10000-users に startTime を付けています。k6 の「シナリオ」単位で開始時刻をずらす考え方は、公式の [Scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/) でも説明されています。

startTimeDelayAfterPreviousScenario（15分）
SCENARIO が problemThenQuestion のときだけ、各段のあとに15分の間隔を空けています。これはPOST からポーリングまでの1シナリオの最長が15分であり、前の段のリクエストがキューや DB に残ったまま次の段に入ると、見かけ上だけ負荷が積み上がることがあるため、その間を置いて系全体が追いつく時間を確保する、という意図です。

### (LLM呼び出しが関わる場合)
今回は実際のLLM呼び出しは行わず、モックする形にしました。
一番の理由はコストです。そのまま叩くと、試験の規模や再実行の回数に比例して費用が伸びるほか、試験たびに結果が変わりすぎて「インフラが耐えたか」の判断がしづらくなることもあると思います。そこで今回はLLM呼び出しをスタブし、応答までにランダムな待ち時間を挟む形にして、生成待ちを含むフローでも 極端に楽観的にならないようにしました。プロバイダ側のレート制限やクォータに当たったときの挙動は、別の論点になるので 今回のスコープからは外しています。



---

## 7. サーバー側インフラの設定（環境作成）

次に環境を用意します。今回は1シナリオをローカルからk6コマンドを叩くだけで、あとは接続先サーバーを開発環境にするのかステージング環境にするのかになります。原則として「本番環境と同じ構成、性能、データ量で確認できる」ことがベストかなと思います。

今回は負荷試験用の環境を新たに作成しました。

決め手は以下です。

- インフラはIaC(AWS CDK)で管理しており、環境の複製、削除が比較的楽
- 開発環境は構成や設定値が異なる部分が多い(一時的に設定変えることもできるが、構成も合わせてとなると専用の環境をゼロから作った方が楽だと判断)
- ステージング環境はAWSアカウントが同じだった。
  - アカウント単位の制限が本番環境に影響しうる(例: Lambdaの同時起動数)
  - メンテナンス時間を設けたり、深夜帯にやる選択肢もあるが、その時間で絶対に終わる見込みはなくリスクが大きい

上記に加えて後述するモニタリングも専用環境用に可視化された方がわかりやすいというのもあります。

## 8. モニターを作成

これはデータさえ取れていれば後に設定しても問題ないですが、今回は事前にモニタリングしやすいようにDatadogのダッシュボードを作成しておきました。もし本番環境用に運用しているものがあればそれをクローンする形でも良いと思いますし、いちいちAWSのコンソールからリソースごとのモニタリングをポチポチ確認するよりかは、一元管理してあると楽だと思うので、事前に作って実行中もリアルタイムで確認できる状態にしておくと良いのかなと思います。

![](https://static.zenn.studio/user-upload/6d21fbe1f9e8-20260512.png)



## 7. テスト実行・計測

ここまできたらあとは実行するだけです！
前述の通り、今回はローカルからコマンドを叩くだけなので、実行は

```bash
npm run smoke --scenario=xxx
npm run load --scenario=xxx
```

するだけですね。smokeは一瞬で終わりますが、loadテストは時間がかかるので優雅にコーヒーでも飲みながらメトリクスを観察しましょう。(本当は実行ログの画像貼っておきたかったんですが用意するのが大変だったので気が向いたら貼ります)

---

## 9. 評価・修正
結果は以下のようになりました。

### smoke test

```
█ THRESHOLDS 

    http_req_duration{type:api}
    ✓ 'p(95)<2000' p(95)=1.04s

    http_req_failed{error:timeout}
    ✓ 'rate==0' rate=0.00%

    http_req_failed{status:>=400,status:<500}
    ✓ 'rate==0' rate=0.00%

    http_req_failed{status:>=500}
    ✓ 'rate==0' rate=0.00%


  █ TOTAL RESULTS 

    checks_total.......: 19      0.101521/s
    checks_succeeded...: 100.00% 19 out of 19
    checks_failed......: 0.00%   0 out of 19

    ✓ status is expected

    HTTP
    http_req_duration................: avg=818.44ms min=600.26ms med=619.94ms max=2.75s p(90)=1.04s    p(95)=1.25s
      { expected_response:true }.....: avg=818.44ms min=600.26ms med=619.94ms max=2.75s p(90)=1.04s    p(95)=1.25s
      { type:api }...................: avg=716.28ms min=600.26ms med=616.43ms max=1.17s p(90)=937.79ms p(95)=1.04s
    http_req_failed..................: 0.00%  0 out of 20
      { error:timeout }..............: 0.00%  0 out of 0
      { status:>=400,status:<500 }...: 0.00%  0 out of 0
      { status:>=500 }...............: 0.00%  0 out of 0
    http_reqs........................: 20     0.106864/s

    EXECUTION
    iteration_duration...............: avg=3m7s     min=3m7s     med=3m7s     max=3m7s  p(90)=3m7s     p(95)=3m7s 
    iterations.......................: 1      0.005343/s
    vus..............................: 1      min=1       max=1
    vus_max..........................: 1      min=1       max=1

    NETWORK
    data_received....................: 105 kB 558 B/s
    data_sent........................: 41 kB  221 B/s


running (03m07.2s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [======================================] 1 VUs  03m07.1s/10m0s  1/1 shared iters
```

これは問題なさそうですね。うまく動くているし、エラーがないことも確認できます。


### load test 

```
THRESHOLDS 

    http_req_duration{type:api}
    ✓ 'p(95)<2000' p(95)=912.09ms

    http_req_failed{error:timeout}
    ✓ 'rate==0' rate=0.00%

    http_req_failed{status:>=400,status:<500}
    ✓ 'rate<0.02' rate=0.00%

    http_req_failed{status:>=500}
    ✓ 'rate<0.005' rate=0.00%


  █ TOTAL RESULTS 

    checks_total.......: 38549   10.619413/s
    checks_succeeded...: 100.00% 38549 out of 38549
    checks_failed......: 0.00%   0 out of 38549

    ✓ status is expected

    HTTP
    http_req_duration................: avg=648.26ms min=230.98ms med=599.16ms max=4.47s p(90)=869.76ms p(95)=923.09ms
      { expected_response:true }.....: avg=648.26ms min=230.98ms med=599.16ms max=4.47s p(90)=869.76ms p(95)=923.09ms
      { type:api }...................: avg=641.72ms min=230.98ms med=597.34ms max=4.47s p(90)=864.52ms p(95)=912.09ms
    http_req_failed..................: 0.00%  0 out of 39643
      { error:timeout }..............: 0.00%  0 out of 0
      { status:>=400,status:<500 }...: 0.00%  0 out of 0
      { status:>=500 }...............: 0.00%  0 out of 0
    http_reqs........................: 39643  10.920786/s

    EXECUTION
    dropped_iterations...............: 770    0.212118/s
    iteration_duration...............: avg=2m40s    min=22.32s   med=2m32s    max=5m42s p(90)=4m39s    p(95)=4m59s   
    iterations.......................: 1737   0.478506/s
    vus..............................: 435    min=0          max=532
    vus_max..........................: 815    min=50         max=815

    NETWORK
    data_received....................: 368 MB 101 kB/s
    data_sent........................: 94 MB  26 kB/s




running (1h00m30.1s), 0000/0815 VUs, 1737 complete and 696 interrupted iterations
1000-users  ✓ [======================================] 043/051 VUs    10m0s  0.33 iters/s
5000-users  ✓ [======================================] 218/262 VUs    10m0s  1.67 iters/s
10000-users ✓ [======================================] 0435/0532 VUs  10m0s  3.33 iters/s

```

こちらもk6の結果は成功しています。

```
1000-users  ✓ [======================================] 043/051 VUs    10m0s  0.33 iters/s
5000-users  ✓ [======================================] 218/262 VUs    10m0s  1.67 iters/s
10000-users ✓ [======================================] 0435/0532 VUs  10m0s  3.33 iters/s
```
リクエスト数も期待した結果になっていますね。
1000-users, 5000-users, 10000-users別に作成したDatadogのダッシュボードも見ながら確認してみます。

#### 1000-users

✅ APIレイテンシ: 600[ms]ぐらい（変化なし）

✅ エラー率: 0件

✅ スループット: 0.33[req/s] (3秒に1質問)を捌ける

✅ リソース使用率
- ✅ DB
  - ACU(MAX): 1.8
  - ACU使用率: 約22[%]
  - 接続数(Proxy-Aurora): 7connection

![](https://static.zenn.studio/user-upload/38c6dfe9efdf-20260513.png)

- ✅ API Lambda
  - メモリ使用率: 20[%]
  - 600[ms]ぐらい
  - 同時起動数: 1.3台
  - pollingのRPS(設定した0.33r/sは質問API単体、1VUでその後最大30回までpollingする)(MAX): 5.0[r/s]

![](https://static.zenn.studio/user-upload/6f5f1ead290e-20260513.png)

- ✅ Worker Lambda
  - 同時起動数MAX: 45.5台
  - DLQ転送: 0件
  - SQSの最古メッセージ滞留時間: 5分程度

![](https://static.zenn.studio/user-upload/7318c75a3ded-20260513.png)


#### 5000-users

✅ APIレイテンシ: 600~700[ms]ぐらい（変化なし）

✅ エラー率

DBのACU使用率が70%を超え、warningがくる(エラー通知に設定している閾値に到達しました)<br>
設定したSLIは守れているのでOK(&ここはmaxACUを上げるだけなので簡単に調整がきく)

✅ スループット: 1.67[req/s] を捌ける

✅ リソース使用率
- ⚠️ DB
  - ACU(MAX): 6.0
  - ACU使用率: 約75[%]まで上昇
  - 接続数(Proxy-Aurora): 12.5connection

![](https://static.zenn.studio/user-upload/c72c4453123a-20260513.png)

- ✅ API Lambda
  - メモリ使用率: 20[%]
  - 700[ms]ぐらい
  - 同時起動数: 3.8台
  - pollingのRPS(MAX): 28.0[r/s]

![](https://static.zenn.studio/user-upload/eb0d360ea056-20260513.png)

- ✅ Worker Lambda
  - 同時起動数MAX: 246.4台
  - DLQ転送: 0件
  - SQSの最古メッセージ滞留時間: 5分程度

![](https://static.zenn.studio/user-upload/ef36227cc7a8-20260513.png)

#### 10000-users

✅ APIレイテンシ: 630[ms]ぐらい（変化なし）

✅ エラー率: 0件

✅ スループット: 3.33[req/s] を捌ける

✅ リソース使用率(2つ目の山付近)
- ✅ DB
  - ACU(MAX): 4.66
  - ACU使用率: 58.3[%]まで上昇
  - 接続数(Proxy-Aurora): 31.5connection

![](https://static.zenn.studio/user-upload/83bf39a44610-20260513.png)

- ✅ API Lambda
  - メモリ使用率: 20[%]
  - 630[ms]ぐらい
  - 同時起動数: 5.75台
  - pollingのRPS(MAX): 57.0[r/s]

![](https://static.zenn.studio/user-upload/4cb2a3b0cf44-20260513.png)

- ✅ Worker Lambda
  - 同時起動数MAX: 508.4台
  - DLQ転送: 0件
  - SQSの最古メッセージ滞留時間: 5分程度

![](https://static.zenn.studio/user-upload/6594ab658984-20260513.png)

と、こんな感じでした。基本的に合格ラインを守れている項目がほとんどだなという結果になりましたが、気になる点でいうと5000-usersの時よりも10000-usersの時の方がDBのACUが若干下回っていました。原因はイマイチわからないのですが、キャッシュが効いていたのかも...?と思い、DBインスタンスを再起動して10000-usersのシナリオのみ、30分間再度実行してみることにしました。

結果は下記のようになりました。


✅ APIレイテンシ: 650[ms]ぐらい

✅ エラー率:

DBのACU使用率が70%を超え、warningがくる<br>
設定したSLIは守れているのでOK

✅ スループット: 3.33[req/s] を捌ける

✅ リソース使用率(2つ目の山付近)
- ⚠️ DB
  - ACU(MAX): 6.0
  - ACU使用率: 75[%]まで上昇
  - 接続数(Proxy-Aurora): 58.5connection

![](https://static.zenn.studio/user-upload/cf7515ab2a73-20260513.png)

- ✅ API Lambda
  - メモリ使用率: 20[%]
  - 630[ms]ぐらい
  - 同時起動数: 5.75台
  - pollingのRPS(MAX): 57.0[r/s]

![](https://static.zenn.studio/user-upload/13e2ff7d89b4-20260513.png)

- ✅ Worker Lambda
  - 同時起動数MAX: 508.4台
  - DLQ転送: 0件
  - SQSの最古メッセージ滞留時間: 5分程度

![](https://static.zenn.studio/user-upload/34d08dfc88bc-20260513.png)

のようになり、結果としては1回目の5000-usersと似たような結果になりました。ただ、Lambdaの同時起動数はしっかり倍になっているので負荷自体は想定通りかけられていると思います。

総じて、この結果からわかることをまとめると、
- 10,000人想定の試験では、1秒に3質問、500人が同時にアプリケーションを利用している想定で、**エラーが大量に発生する、レイテンシが極端に高くなる等のUX影響がありそうな問題は発生しなかったため「正式リリース後の通常時/ピークタイムにシステムが想定通り動作することを担保する」という目的は達成できたと言えます**

- DBのACUが一番ネックになりそうだということがわかりました。ダウンするまで負荷をかけていませんが、アクセスが増えた時にまず気にしないといけないのはDBのACUなんだなということが分かっただけでもかなりの功績だと思います。
    - 今回の場合は、すでに、maxACUは本番環境でもアラートを設定済み & 対応は値を8ACU→xACUに変更するだけなので容易に対応可能
    - ということで特に対応はなしと判断しました

- 次にネックになりそうなのはWorker Lambdaの同時起動数だというのも予想がつきました。
  - 517台まで使用したので約4倍のRPSになるとLambdaが原因でダウンする可能性がある(このAWSアカウントでは2,000台が上限)
  - ただこちらも直近ではクォータ引き上げまでは現時点でしなくても良いという判断
  - Lambdaの同時起動数はアラート設定を追加して気付けるようにする、という対応しました



と、細かく色々みてきましたが、意外と耐えているんだな、というのが正直な感想です。<br>やってみたら全然うまくいかない、逆に今回みたいに今の構成、設定のままでもなんとかなりそうなレベルなんだな、みたいな部分がざっくりわかるだけでも相当な価値があると思います。

---


## まとめ

初めて負荷試験を任されたとき、手を動かす前に一番時間がかかったのは **k6 の書き方よりも、その前の「何を試すか」を言葉と数字に落とす作業** でした。キャパシティの仮定、目的、叩く API とシナリオ、合格ライン、テストの種類、専用環境、モニタリングまで一通り決めてからスクリプトに触ると、迷いどころが減り、結果を読むときも「何のためにこのグラフを見ているか」に戻れます。

実際に試してみた結果、10,000 人想定のレートでもレイテンシやエラー率は想定内に収まり、当初掲げた目的は満たせたと言えそうです。一方で、DB の ACU や Worker Lambda の同時起動数のように、**次に伸びたときに最初に効いてきそうな箇所** が見えたことも大きな収穫でした。ブレークポイントまで踏み込んでいないので「絶対にここまで耐える」とは言えませんが、「本番で気にすべき順番」が手元に残っただけでも、試験を一回通した意味はあると感じています。

社内に手順やテンプレート、SRE とのすり合わせがあると、いきなり白紙から設計するよりずっと楽になります。とはいえ最終的に「この仮定でいいか」「この合格ラインで十分か」はプロダクト側の判断になるので、**負荷試験はインフラの話題に見えて、実はプロダクトと運用の前提を言語化する作業** に近い、という印象です。

この記事が、これから初めて負荷試験に向き合う方のチェックリスト兼メモのような位置づけになれば嬉しいです。細かいノウハウは公式ドキュメントや専門記事に譲りつつ、ここでは **「まず地図を描いてから k6 を書く」** という順番だけ持ち帰ってもらえたら幸いです。
