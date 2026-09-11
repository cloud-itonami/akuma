# Operator quickstart

**ここに載っているコマンドは、2026-09-03 に fresh clone に対して実際に実行した
ものだけである。** 出力は貼った時点の実測値。踏めなかった手順は載せていない
（CLAUDE.md にある `opa test` / `kubectl apply` / `pnpm db:migrate` は、その対象
ファイルがこの repo に無いので**ここでは踏めない**。理由は [README](../README.md)
の「この repo に**無い**もの」を参照）。

所要は 5 分弱。JVM を起こすのは 2. と 3. だけで、4. は nbb（Node）で走る。

## 0. 前提

| 道具 | 用途 | 確認 |
|---|---|---|
| `git` | clone | `git --version` |
| `clojure` | 2. test / 3. lint | `clojure --version` |
| `nbb` | 4. 実行（JVM 不要） | `nbb --version` |
| `curl` / `dig` | 5. DID 検証 | 標準 |

`clojure` と `nbb` は独立している。**片方しか無くても 1〜5 のうち踏める分は踏める。**

## 1. clone

```bash
git clone git@github.com:cloud-itonami/akuma.git
cd akuma
```

この workspace の west checkout から使う場合は
`orgs/cloud-itonami/akuma`（remote 名は `origin` ではなく `cloud-itonami`）。

## 2. テストを走らせる

```bash
kbb -M:test
```

```
Running tests in #{"test"}

Testing akuma.murakumo-test

Ran 9 tests containing 161 assertions.
0 failures, 0 errors.
```

初回は依存（cognitect test-runner）の解決で 1〜2 分かかる。2 回目以降は数秒。

## 3. lint

```bash
kbb -M:lint
```

```
src/akuma/murakumo.cljc:131:14: warning: unused binding input
linting took <N>ms, errors: 0, warnings: 1
```

（所要 ms は貼っていない —— このマシンでは並行セッションの負荷で 5 秒にも 22 秒にも
なった。見るのは `errors: 0, warnings: 1` の方。）

**warning 1 件は既知で、exit 0 である**（`:lint` alias は `--fail-level error`）。
`records-for` が `:as input` を束縛して使っていない。ここを直すのは lint の
仕事であってこの quickstart の仕事ではないが、**exit 1 になったら warning では
なく error が増えている**ので、その差分を見ること。

## 4. gate が閉まることを自分で確かめる（JVM 不要）

この repo の主張は「attestation が揃わなければ effect を 1 つも出さない」である。
**それを信じずに、その場で両方向を出す。**

```bash
kbb --backend sci --classpath src -e '
(ns probe (:require [akuma.murakumo :as m]))
(let [all (into {} (map (fn [g] [g true]) m/common-gates))
      one-short (dissoc all (first m/common-gates))]
  (println "gates required:" (count m/common-gates))
  (println "no attestation   ->" (:status (m/cell-plan :scope {}))
           "effects:" (count (:effects (m/cell-plan :scope {}))))
  (println "one gate missing ->" (:status (m/cell-plan :scope {:attestations one-short}))
           "effects:" (count (:effects (m/cell-plan :scope {:attestations one-short}))))
  (println "all 7 attested   ->" (:status (m/cell-plan :scope {:attestations all :request-id "req-1"}))
           "effects:" (count (:effects (m/cell-plan :scope {:attestations all :request-id "req-1"}))))
  (println "fleet-wide, zero attestation -> total effects:"
           (reduce + (map #(count (:effects %)) (vals (m/all-cell-plans {}))))))'
```

```
gates required: 7
no attestation   -> :blocked effects: 0
one gate missing -> :blocked effects: 0
all 7 attested   -> :ready effects: 1
fleet-wide, zero attestation -> total effects: 0
```

**3 行目が `:ready` になり、1〜2 行目が `:blocked` になることの両方を見ること。**
`:blocked` しか出ない実行は、gate が閉まっている証拠ではなく、単に何も動いて
いない証拠かもしれない（例えば classpath を間違えて別の名前空間を読んでいても
0 件は出る）。**7 本目を 1 本抜いただけで止まる**ことが、この境界の主張そのもの。

## 5. DID を検証する

`.well-known/did.json` が公開している DID と、コードが名乗る DID は**別物**
（[README](../README.md) の Identity 節）。両方をその場で引く:

```bash
for u in "https://etzhayyim.com/actor/akuma/did.json" \
         "https://etzhayyim.github.io/com-etzhayyim-akuma/did.json"; do
  printf '%-58s %s\n' "$u" "$(curl -sS -o /dev/null -w '%{http_code}' --max-time 20 "$u")"
done
dig +short akuma.etzhayyim.com
```

```
https://etzhayyim.com/actor/akuma/did.json                 200
https://etzhayyim.github.io/com-etzhayyim-akuma/did.json   404
                                     <- akuma.etzhayyim.com は NXDOMAIN（空行）
```

`dig` が**何も返さない**のが期待値である（レコードが無い）。ここに IP が出るように
なったら、`src/akuma/murakumo.cljc` の `actor-did` が指す先が実在し始めたという
ことなので、README の Identity 節を測り直すこと。

## 踏めないもの（なぜ載っていないか）

| CLAUDE.md のコマンド | ここで踏めない理由 |
|---|---|
| `opa test 00-contracts/policies/etzhayyim/akuma/scope/ -v` | `00-contracts/` がこの repo に無い |
| `kubectl apply -k 50-infra/k8s/akuma-langserver/` | `50-infra/` がこの repo に無い |
| `pnpm db:migrate` | `30-graph/` と `package.json` がこの repo に無い |
| `70-tools/scripts/akuma/provision-authority-key.sh` | `70-tools/` がこの repo に無い |
| `npx wrangler deploy`（PDS） | Worker のソースがこの repo に無い |

これらは上流モノレポのデプロイ手順である。**この repo を clone しただけの
オペレータには実行できない**ので、ここには書かない。
