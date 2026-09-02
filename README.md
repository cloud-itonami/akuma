# akuma — probe を「計画」するだけの、実行しないアクター境界

**名乗り**: `akuma`（悪魔）は機能を示さないメタファ名である。この repo が実際に
持っているのは、authorized red team probing アクターの **純粋な `.cljc` 計画境界
1 本だけ** —— `akuma.murakumo`（`src/akuma/murakumo.cljc`）。

**この repo に probe を実行するコードは無い。** ネットワークに触る関数も、
ファイルを読む関数も、シェルを起動する関数も 1 つも無い。`akuma.murakumo` は
入力を受けて **effect の *記述*** を返す純関数の集まりで、その記述を誰かが実行
するかどうかは、この repo の外側の話である。依存は `clojure.string` だけ。

## この repo に在るもの（`git ls-files`、2026-09-03 実測）

| path | 何か |
|---|---|
| `src/akuma/murakumo.cljc` | 唯一の実装。下記の計画境界 |
| `test/akuma/murakumo_test.cljc` | その契約テスト（9 tests / 161 assertions） |
| `deps.edn` | `:test`（cognitect test-runner）/ `:lint`（clj-kondo） |
| `actor-manifest.jsonld` | アクター identity + governance の宣言 |
| `.well-known/did.json` | 公開 DID 文書 |
| `CLAUDE.md` | **この repo ではなく上流デプロイの説明**（下記） |
| `MIGRATION-TODO.md` | 移行時の残タスク（未消化） |
| `NOTICE` | Apache-2.0 + etzhayyim Charter Rider v3.1 |

## この repo に**無い**もの —— CLAUDE.md の読み方

`CLAUDE.md` は K8s LangServer pod・SvelteKit CF Worker・Rego 認可ポリシー・
XRPC lexicon・Alembic migration を説明し、それらを「Key Files」として列挙して
いる。**そこに挙がっている path は 1 つもこの repo に存在しない**（2026-09-03 実測）:

```
00-contracts/policies/etzhayyim/akuma/scope/policy.rego   MISSING
00-contracts/lexicons                                      MISSING
90-docs/adr                                                MISSING
70-tools/scripts/akuma                                     MISSING
50-infra/k8s/akuma-langserver                              MISSING
30-graph                                                   MISSING
```

CLAUDE.md が記述しているのは上流モノレポ（`etzhayyimcojp/20-actors` から
2026-05-21 に移設、`NOTICE` 参照）に在るデプロイであって、この repo の中身では
ない。**したがって CLAUDE.md の `Run: opa test 00-contracts/policies/... (11/11 PASS)`
はここでは踏めない。** ここで踏める手順は
[`docs/operator-quickstart.md`](docs/operator-quickstart.md) が正本。

## `akuma.murakumo` が答えること

manifest 由来の **cell**（11 個）ごとに「いま effect を出してよいか」を判定し、
出してよければ MST への put-record effect を組み立てる。

- **cell**: 11 個（`:runprobe` `:scope` `:finding` `:risk` `:indicator` `:koji`
  `:kyumei` `:shinka` `:shinkaevolution` `:shinkaknowledge` `:domain-knowledge`）。
  各 cell はちょうど 1 collection、phase は全て `:event`、murakumo node は
  全て `reuben`。
- **gate**: 全 cell 共通で 7 本（`:council-charter-attestation`
  `:no-platform-held-key-baseline` `:no-probing-baseline`
  `:murakumo-only-inference-baseline` `:did-primary-baseline`
  `:append-only-gate-baseline` `:kotoba-only-substrate-baseline`）。
- **deny-by-default**: attestation が 1 つも無いとき、11 cell すべてが
  `:status :blocked` を返し、**生成される effect は合計 0 件**。gate が 1 本でも
  欠けていれば `:effects []` で、部分実行は起きない。

```
(cell-plan :scope {})                                  ;; => :blocked, effects 0
(cell-plan :scope {:attestations <7 gates> ...})       ;; => :ready,   effects 1
```

この「欠けたら止まる」性質はテストが両方向で押さえている
（`cell-plan-blocks-when-gates-missing` / `cell-plan-ready-when-gates-satisfied`）。
テストは cell 名を直書きせず `cell-specs` を走査するので、manifest が cell を
足しても契約は成り立ち続ける。

## Identity —— DID が 2 つあり、片方は解決しない

**2026-09-03 に実測した現在地**（コミットメッセージの引き写しではない）:

| DID | 引ける URL | 結果 |
|---|---|---|
| `did:web:etzhayyim.com:actor:akuma` | `https://etzhayyim.com/actor/akuma/did.json` | **200**、`id` が `.well-known/did.json` と一致 |
| `did:web:etzhayyim.github.io:com-etzhayyim-akuma` | `https://etzhayyim.github.io/com-etzhayyim-akuma/did.json` | 404 |
| `did:web:akuma.etzhayyim.com` | `https://akuma.etzhayyim.com/.well-known/did.json` | **NXDOMAIN**（ホストが存在しない） |

`.well-known/did.json` が公開しているのは 1 行目（解決する方）で、これは
2026-07-27 の `1c7124f fix(identity): restore the did:web that actually resolves`
が直したもの。

⚠ **一方 `src/akuma/murakumo.cljc` の `actor-did` は 3 行目**
（`did:web:akuma.etzhayyim.com`）**を持っており、そのホストには DNS レコードが
無い。** `actor-manifest.jsonld` の `@id` と CLAUDE.md も同じ 3 行目を使っている。
つまり **生成される全 effect の `:actor` は、解決しない DID を名乗る。**

これは既知の未修正の食い違いであって、この README が新たに導入したものではない。
ここでは**記録するにとどめ、直していない** —— 直すと effect の中身が変わるので、
identity の正本をどちらにするかの決定（と `actor-manifest.jsonld` の追随）が要る。
検証コマンドは quickstart の「4. DID を検証する」に置いた。

## 使う

[`docs/operator-quickstart.md`](docs/operator-quickstart.md) —— fresh clone から
test / lint / nbb 実行 / DID 検証まで、**実際に踏んだコマンドだけ**を載せてある。

## Status

`actor-manifest.jsonld` は `status: active` を主張するが、CLAUDE.md 自身が
`production_live_pending: true` と書いており、そこに挙がる 5 つの人手ステップ
（authority key の払い出し・K8s apply・migration・PDS deploy・E2E smoke）は
**どれもこの repo では実行できない**（対象のファイルが無い）。この repo 単体で
検証できるのは、上の計画境界が gate を閉じることだけである。

## License

Apache License 2.0 + etzhayyim Charter Compliance Rider v3.1（`NOTICE` 参照）。
