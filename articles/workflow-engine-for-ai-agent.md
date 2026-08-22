---
title: "AIエージェントの実行基盤を比較——自前実装 vs Temporal vs AWS Durable Functions"
emoji: "⚙️"
type: "tech"
topics: ["ai-agent", "workflow", "idempotency", "typescript", "temporal"]
published: true
---

2026年に入り、AIエージェントの実装フレームワークは激変しています。OpenAIのAgents SDK、AnthropicのClaude Code、GoogleのADKなど、エージェントを組みやすくするライブラリが次々に登場し、「AIに何を判断させるか」は以前よりずっと簡単になりました。

一方で、AIが判断した結果を「本番で安全に動かす」という問題は、フレームワークでは解決されません。承認待ち、再起動、再試行、二重実行防止——これらはAIの推論そのものではありませんが、実運用には必須です。

この記事では、ワークフローエンジンがこの実行基盤の問題をどこまで解決してくれるのか、同じ処理を自前で実装したあとTemporalとAWS Lambda Durable Functionsで置き換えて比較します。

## サンプルコードについて

今回のコードは仕組みを比較するための最小例です。本番運用に必要な認証、永続化、監視、詳細なエラー処理などは省略しています。

動かし方はシンプルです。Temporalは`npx @temporalio/cli server start-dev`でローカル開発サーバーを起動し、Workerを登録して実行します。AWS Lambda Durable Functionsは`@aws/durable-execution-sdk-js`をインストールし、AWS環境にデプロイします。

題材は単純です。

> 在庫が少なくなったらAIが補充計画を作り、人間が承認したら外部の在庫APIを実行する。

処理の流れはこうなります。

```text
在庫を検知
  ↓
AIが補充の要否を判断
  ↓
補充計画を作る
  ↓
人間が承認
  ↓
外部APIを実行
  ↓
結果を通知
```

今回見たいのはAIの推論ではなく、その前後にある実行部分です。

たとえば、次のような問題があります。

| 問題 | 考えること |
|---|---|
| 承認待ち | 人間の返答をどう待つか |
| 再起動 | プロセスが落ちても、どこから続けるか |
| 再試行 | 一時的な失敗をどうやってやり直すか |
| 二重実行 | 同じ外部APIを2回実行しないために何をするか |
| 古い承認 | ユーザーが見た計画と実際の実行内容をどう一致させるか |
| 長時間の待機 | 待っている間、処理状態をどう保持するか |

この問題自体は、すでに企業の技術記事でも扱われています。AWS Compute Blogで紹介された例では、AIエージェント、人間の承認、外部サービス連携を含む長時間処理をLambda Durable Functionsで実装しています。

この記事で見たいのは、その1つ手前の疑問です。

> ワークフローエンジンを使うと、自前で書いていたコードのうち、具体的に何が消えるのか？

## 自前で全部実装してみる

自分で処理状態を持ち、承認を待ち、期限を管理し、外部APIの再実行も制御します。


```ts
import assert from "node:assert";
import { createHash } from "node:crypto";

// 処理の1ステップを表す型。
// 実際には「在庫確保」「通知」など、外部に作用する処理が入る。
type Step = {
  id: string;
  name: string;
  effect: string;
};

// ワークフローの進行状態を自分で持つ。
type Task = {
  id: string;
  state: "waiting_approval" | "running" | "done";
  planHash: string | null;
  approvalDeadline: number;
  owner: string | null;
  ownerDeadline: number;
};

// 承認を待てる時間。デモでは60秒にする。
const APPROVAL_TTL_MS = 60_000;

// 担当者が処理を保持できる時間。止まったら別の担当者が再開できるようにする。
const EXECUTION_TTL_MS = 30_000;

// 計画の内容からハッシュ値を作る。
// ハッシュアルゴリズム自体は今回の主題ではないため、Node.js標準のcryptoを利用する。
const hashPlan = (plan: Step[]) => {
  return createHash("sha256")
    .update(JSON.stringify(plan))
    .digest("hex");
};

const step = (id: string, name: string): Step => ({
  id,
  name,
  effect: `${name}(${id})`,
});

// AIが作った計画を保存し、ユーザーに承認を求める状態にする。
function requestApproval(
  task: Task,
  plan: Step[],
  now = Date.now(),
) {
  task.planHash = hashPlan(plan);
  task.approvalDeadline = now + APPROVAL_TTL_MS;
  task.state = "waiting_approval";

  return {
    taskId: task.id,
    planHash: task.planHash,
  };
}

// ユーザーが承認した計画と、現在保存されている計画が同じかを確認する。
// 同時に、承認期限と処理担当の期限も確認してから実行状態へ進める。
function approve(
  task: Task,
  submittedPlanHash: string,
  workerId: string,
  now = Date.now(),
) {
  const ok =
    task.state === "waiting_approval" &&
    task.approvalDeadline > now &&
    task.planHash === submittedPlanHash &&
    (task.owner === null || task.ownerDeadline < now);

  if (!ok) return false;

  task.state = "running";
  task.owner = workerId;
  task.ownerDeadline = now + EXECUTION_TTL_MS;

  return true;
}

// 外部API側が持つ「同じキーなら二度適用しない」という仕組みの簡易モック。
//
// これはワークフロー基盤ではなく、外部サービス側の責任。
const executed = new Map<string, string>();

async function callExternalApi(
  taskId: string,
  stepId: string,
  effect: string,
) {
  // 再試行しても、同じ処理には同じキーを使う。
  const idempotencyKey = `${taskId}:${stepId}`;
  const previous = executed.get(idempotencyKey);

  if (previous !== undefined) {
    return { idempotencyKey, result: previous, replayed: true };
  }

  // 実際にはここで外部APIを呼ぶ。
  executed.set(idempotencyKey, effect);

  return { idempotencyKey, result: effect, replayed: false };
}

// ---- 動作確認 ----

const task: Task = {
  id: "task-1",
  state: "waiting_approval",
  planHash: null,
  approvalDeadline: 0,
  owner: null,
  ownerDeadline: 0,
};

const plan = [step("s1", "reserve_stock")];
const approval = requestApproval(task, plan);

// ユーザーが見た計画のハッシュを返して承認する。
assert.equal(
  approve(task, approval.planHash, "worker-1"),
  true,
);

// 同じ処理を2回呼んでも、外部側では1回しか適用されない。
const first = await callExternalApi(
  task.id,
  plan[0].id,
  plan[0].effect,
);

const second = await callExternalApi(
  task.id,
  plan[0].id,
  plan[0].effect,
);

assert.equal(first.replayed, false);
assert.equal(second.replayed, true);

console.log("自前実装: アサーションをすべて通過しました");
```

このコードで実装しているのは、単なる業務ロジックだけではありません。

状態、承認期限、処理担当の期限、計画のハッシュ値、外部APIの冪等キーまで、自分で考えて持つ必要があります。

さらに本番では、これをプロセス再起動や複数インスタンスに対応させる必要があります。

## Temporal の場合

次は同じ要件をTemporalで実装します。

ここで比較を厳密にするため、Temporal側でも次のものは残します。

- `hashPlan`の実装
- 承認した計画と現在の計画を一致させる処理
- 外部API側の冪等性

つまり、Temporalが提供していない部分を意図的に削らないようにします。

```ts
import { createHash } from "node:crypto";

import {
  condition,
  defineSignal,
  proxyActivities,
  setHandler,
} from "@temporalio/workflow";

// 処理の1ステップを表す型。
type Step = {
  id: string;
  name: string;
  effect: string;
};

// 実際の処理はActivityに分離する。
const { createPlan, executePlan } = proxyActivities<{
  createPlan(taskId: string): Promise<Step[]>;
  executePlan(plan: Step[], planHash: string): Promise<void>;
}>({
  startToCloseTimeout: "1 minute",
  retry: {
    maximumAttempts: 5,
  },
});

// 承認時にユーザーが見た計画のハッシュ値を受け取る。
export const approve = defineSignal<[string]>("approve");

// 計画の内容からハッシュ値を作る。
// これは業務上必要な処理なので、自前実装と同じものを残す。
export function hashPlan(plan: Step[]) {
  return createHash("sha256")
    .update(JSON.stringify(plan))
    .digest("hex");
}

export async function restockWorkflow(taskId: string) {
  // AIによる計画作成はActivityとして実行する。
  const plan = await createPlan(taskId);
  const planHash = hashPlan(plan);

  let approvedHash: string | null = null;

  // 外部から届く承認を受け取る。
  setHandler(approve, (submittedHash) => {
    approvedHash = submittedHash;
  });

  // 承認が届くまでWorkflowを待つ。
  // 自前実装の APPROVAL_TTL_MS と同じ60秒で期限を切る。
  // 待機中にWorkerプロセスが落ちても、Temporalが状態を保持して再開する。
  const approved = await condition(
    () => approvedHash !== null,
    "60 seconds",
  );

  // 期限までに承認が届かなかったら、このWorkflowは失敗として終わる。
  if (!approved) {
    throw new Error("approval deadline exceeded");
  }

  // ユーザーが承認した計画と、現在の計画が同じかを確認する。
  if (approvedHash !== planHash) {
    throw new Error("approved plan does not match current plan");
  }

  await executePlan(plan, planHash);
}

// ---- Activity ----

export async function createPlan(taskId: string): Promise<Step[]> {
  // 実際にはここでAIを呼び出して計画を作る。
  return [
    {
      id: "s1",
      name: "reserve_stock",
      effect: `reserve_stock(${taskId})`,
    },
  ];
}

export async function executePlan(
  plan: Step[],
  planHash: string,
) {
  for (const currentStep of plan) {
    await callExternalApi(planHash, currentStep);
  }
}

// 外部API側の冪等性を簡易的に再現するモック。
// Temporalではなく、あくまで外部サービス側の責任。
const executed = new Map<string, string>();

async function callExternalApi(
  planHash: string,
  currentStep: Step,
) {
  // 同じ計画の同じステップには同じキーを使う。
  // Temporalの再試行とは別に、外部API自身にも重複実行を防ぐ仕組みが必要。
  const idempotencyKey = `${planHash}:${currentStep.id}`;
  const previous = executed.get(idempotencyKey);

  if (previous !== undefined) {
    return { idempotencyKey, result: previous, replayed: true };
  }

  // 実際にはここで外部APIを呼ぶ。
  executed.set(idempotencyKey, currentStep.effect);

  return {
    idempotencyKey,
    result: currentStep.effect,
    replayed: false,
  };
}
```

自前実装にあった「状態を自分で保存する」「承認待ちの状態を自分で持つ」「再起動したらどこから続けるかを自分で判断する」といった処理が、ワークフローエンジンから消えました。

なお、冪等キーの設計は意図的に変えています。自前実装では`taskId`ベースで「同じタスクの同じステップは1回だけ」を保証し、Temporal・Durable Functionsでは`planHash`ベースで「同じ計画内容なら再適用しない」ことを保証します。計画が変われば別キーになるため、古い計画の実行結果が新しい計画に流用されない点が違いです。

一方で、`hashPlan`と外部APIの冪等性は残っています。

これは重要な比較です。

「Temporalを入れたらコードが全部消える」わけではありません。

## AWS Lambda Durable Functions でも書いてみる

AWS Lambda Durable Functionsは、Lambdaの実行に、状態保存、再試行、待機、途中からの再開などを組み込んだ仕組みです。[公式ドキュメント](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)でも、人間の承認待ちを含む長時間の処理が扱われています。

今回の要件を同じ粒度で書くと、次のようになります。

```ts
import { createHash } from "node:crypto";
import {
  withDurableExecution,
  type DurableContext,
} from "@aws/durable-execution-sdk-js";

// 処理の1ステップを表す型。
type Step = {
  id: string;
  name: string;
  effect: string;
};

type Approval = {
  planHash: string;
};

// 計画の内容からハッシュ値を作る。
// これは業務上必要な処理なので、自前実装と同じものを残す。
function hashPlan(plan: Step[]) {
  return createHash("sha256")
    .update(JSON.stringify(plan))
    .digest("hex");
}

// AIが作った計画を返す。
async function createPlan(taskId: string): Promise<Step[]> {
  // 実際にはここでAIを呼び出して計画を作る。
  return [
    {
      id: "s1",
      name: "reserve_stock",
      effect: `reserve_stock(${taskId})`,
    },
  ];
}

// 人間に承認を依頼する。
// 実際にはここで承認UIやバックエンドへ
// callbackId、taskId、plan、planHashなどを通知する。
async function requestApproval(
  callbackId: string,
  taskId: string,
  plan: Step[],
  planHash: string,
) {
  console.log("approval requested", {
    callbackId,
    taskId,
    plan,
    planHash,
  });
}

// 外部API側が持つ「同じキーなら二度適用しない」という仕組みの簡易モック。
// これはDurable Functionsではなく、あくまで外部サービス側の責任。
const executed = new Map<string, string>();

async function callExternalApi(
  planHash: string,
  currentStep: Step,
) {
  // 同じ計画の同じステップには同じキーを使う。
  // Durable Functionsの再試行とは別に、外部API自身にも重複実行を防ぐ仕組みが必要。
  const idempotencyKey = `${planHash}:${currentStep.id}`;
  const previous = executed.get(idempotencyKey);

  if (previous !== undefined) {
    return { idempotencyKey, result: previous, replayed: true };
  }

  // 実際にはここで外部APIを呼ぶ。
  executed.set(idempotencyKey, currentStep.effect);

  return {
    idempotencyKey,
    result: currentStep.effect,
    replayed: false,
  };
}

async function executePlan(
  plan: Step[],
  planHash: string,
) {
  for (const currentStep of plan) {
    await callExternalApi(planHash, currentStep);
  }
}

export const handler = withDurableExecution(
  async (
    event: { taskId: string },
    context: DurableContext,
  ) => {
    // AIによる計画作成。
    // Stepの結果はDurable Executionによって保持される。
    const plan = await context.step(
      "create-plan",
      async () => createPlan(event.taskId),
    );

    const planHash = hashPlan(plan);

    // 人間の承認を待つ。
    // callbackIdを外部の承認システムへ渡し、
    // 承認結果がcallbackされるまで待機する。
    const approval = await context.waitForCallback<Approval>(
      "approval-callback",
      async (callbackId) => {
        await requestApproval({
          callbackId,
          taskId: event.taskId,
          plan,
          planHash,
        });
      },
      {
        timeout: {
          seconds: 7 * 24 * 60 * 60,
        },
      },
    );

    // ユーザーが承認した計画と、現在の計画が同じかを確認する。
    if (approval.planHash !== planHash) {
      throw new Error("approved plan does not match current plan");
    }

    // 実際の外部API呼び出し。
    // Stepとして実行結果を保持する。
    await context.step(
      "execute-plan",
      async () => executePlan(plan, planHash),
    );

    return {
      taskId: event.taskId,
      planHash,
      status: "done",
    };
  },
);
```

ここでも、状態保存や待機、再開、再試行のためのコードはワークフローエンジンから消えています。

一方で、`hashPlan`や外部APIの冪等性など、業務側で決める部分は残ります。

AWSの場合は、この仕組みがLambdaの実行環境に組み込まれているため、独立したワークフローエンジンを運用する必要がなく、大きな違いとなります。

AWS自身も2026年6月に、AIエージェント、人間の承認、外部サービス連携を含む長時間処理をLambda Durable Functionsで実装する例を公開しました（[AWS Compute Blog](https://aws.amazon.com/blogs/compute/building-fault-tolerant-multi-agent-ai-workflows-with-aws-lambda-durable-functions/)）。

## どれだけコードが減ったのか

ここでは、比較の条件を揃えるため、3つとも実装に必要な補助コードを含めて考えます。

行数は、空行やコメントを含むため絶対的な指標ではありません。

ただ、コードの中身を見ると、どこに実装の責務が移ったのかがもっとはっきりします。

自前実装では、業務ロジック以外に次の処理を書いていました。

```text
状態を持つ
↓
承認待ちの状態を持つ
↓
承認期限を確認する
↓
担当者の期限を確認する
↓
処理を再開できるようにする
↓
再試行を管理する
```

TemporalやAWS Durable Functionsでは、この部分の多くがワークフローの実行基盤に移ります。

### 消えたコードは何だったのか

| 自前で実装したもの | ワークフローエンジンに任せられるもの |
|---|---|
| 処理状態の保存 | ワークフローの状態管理 |
| 承認待ちの状態管理 | シグナルやCallbackと待機 |
| 再起動後の再開 | 実行履歴やチェックポイントからの再開 |
| 一時的な失敗への再試行 | Activity / Stepの再試行 |
| 待機中のプロセス管理 | 長時間待機の実行管理 |

逆に、以下は残りました。

| 残るもの | なぜ残るのか |
|---|---|
| `hashPlan` | 何を承認したかは業務上のルールだから |
| 外部APIの冪等性 | 外部サービスとの契約だから |
| 承認対象の定義 | 何をユーザーに承認してもらうかは業務仕様だから |
| AIの呼び出し方 | ワークフローエンジンは業務判断をしないから |

## 何をワークフローエンジンに任せ、何を自分で考えるのか

3つの実装を並べると、責務の境界が見えてきます。

| 領域 | 自前実装 | Temporal | Durable Functions |
|---|---|---|---|
| 状態の保存 | 自分で実装 | 任せられる | 任せられる |
| 承認待ち | 自分で実装 | 任せられる | 任せられる |
| 再起動後の再開 | 自分で実装 | 任せられる | 任せられる |
| 再試行 | 自分で実装 | 任せられる | 任せられる |
| 長時間の待機 | 自分で実装 | 任せられる | 任せられる |
| 計画のハッシュ化 | 自分で実装 | 自分で実装 | 自分で実装 |
| 外部APIの冪等性 | 自分で実装 | 自分で実装 | 自分で実装 |
| 承認対象の定義 | 自分で実装 | 自分で実装 | 自分で実装 |
| AIの判断 | 自分で実装 | 自分で実装 | 自分で実装 |

ワークフローエンジンはAIエージェントの「頭脳」を提供しているわけではありません。

AIが作った処理を、長い時間をかけて安全に進めるために必要だった実行基盤を引き受けています。

## 補足: 運用面・コスト面・監視の違い

コードの削減だけでなく、実際の運用で違いが出るポイントを整理します。

| 項目 | 自前実装 | Temporal | AWS Durable Functions |
|---|---|---|---|
| **運用の手間** | 状態保存・再起動・スケーリングを全て自分たちで運用する | Temporal Server（クラスタ）の運用が必要。Temporal Cloudを使えば運用を委託できる | Lambda自体がマネージド。サーバー運用は不要 |
| **コスト構造** | インフラ運用費 + 開発工数 | Temporal Cloud: アクション数 + ストレージ課金。セルフホストならEC2/ECS代 | Lambda実行回数 + 実行時間 + ストレージ課金。利用量に応じて増減 |
| **監視・可観測性** | 自分でログ・メトリクス・アラートを一から設計する | Web UIでWorkflowの実行状況を確認可能。Temporal UIが標準提供 | CloudWatch Logs / X-Ray でLambda実行を追跡 |
| **障害対応** | プロセス再起動・手動リトライ・データ復旧を自分で設計する | Workflow のリプレイ機能により、途中から自動再開 | Lambda の再試行 + DynamoDB への状態永続化で自動復旧 |
| **スケーリング** | 自前でキュー・ワーカーを管理 | Workerを横並びに追加するだけでスケール | Lambdaが自動スケール。バースト時は同時実行数の制限に注意 |

監視、可観測性、コストの観点では、ワークフローエンジンを選ぶことで「運用基盤を自分で作る工数」と「障害時の復旧コスト」を大幅に削減できます。

一方で、Temporalはクラスタ運用のコスト、AWS Durable Functionsはベンダーロックインのリスクといったトレードオフも存在します。

## まとめ

AIエージェントを作るとき、「どうやって止まらずに動かすか」という問題は以前より解きやすくなりました。

Temporalのようなワークフロー基盤や、AWS Lambda Durable Functionsのようにクラウド側へ組み込まれた仕組みによって、状態保存、待機、再開、再試行といった処理を自分で書かずに済むからです。

その分、アプリケーション側で考えるべきことがはっきりしてきます。

```text
AIに何を判断させるか
        ↓
何を人間の承認対象にするか
        ↓
承認した内容をどう固定するか
        ↓
外部サービスに何を実行させるか
        ↓
失敗したとき、何をもう一度実行してよいか
```

つまり、AIエージェントを動かすための基盤が整ってきたことで、「動かし続ける仕組み」そのものよりも、「何を動かしてよいのか」を設計することの重要性が上がっています。

ワークフローエンジンを使うと、AIエージェントの実装が簡単になります。

ただし、本当に簡単になるのは、AIの判断そのものではありません。

AIの判断を現実の処理へつなぐときに必要だった、状態管理や待機、再開、再試行といった基盤部分です。

そこを理解したうえで、どこから先を自分で設計するのかを決めることが、これからのAIエージェント開発では重要になると考えています。

## 参考文献

- [Temporal](https://temporal.io/)
- [Temporal Documentation](https://docs.temporal.io/)
- [AWS Lambda Durable Functions](https://aws.amazon.com/lambda/durable-functions/)
- [AWS Lambda Durable Execution SDK Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-sdk.html)
- [AWS Lambda Durable Execution SDK — TypeScript](https://docs.aws.amazon.com/durable-execution/sdk-reference/languages/typescript/)
- [AWS Compute Blog: Building fault-tolerant multi-agent AI workflows with AWS Lambda Durable Functions](https://aws.amazon.com/blogs/compute/building-fault-tolerant-multi-agent-ai-workflows-with-aws-lambda-durable-functions/)
- [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
