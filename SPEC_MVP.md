<!-- 本書は Workflow(5観点並列設計→統合) の成果。更新時は research §7g とこの位置づけを保つ -->
> **位置づけ（重要）**: 本書は松雪(Matsuyuki)「SEA版リクルートHD」構想の縦**「不動産PF」の本番MVP設計書**。
> - エスクロー/デポジット保護は別縦 **trust-rail(組合・3層分離)** が担う＝本MVPのスコープ外（整合確認済）。
> - 並走セッション**「不動産AI」**(別リモートセッション・同じ不動産の縦)のhandoff到着後、その知見(業界情報/AI機能)を本書へ統合する。**2026-06-18時点＝統合待ち**。
> - マスター設計: `workspace-config/sessions/2026-06-18_matsuyuki_recruit_HD_skeleton_brief.md`
> - プロト(営業道具・使い捨て): `Umagon-G/kh-property-proto`(SPEC.md・v0.7)

# カンボジア構造化不動産プラットフォーム — 本番MVP 統合アーキテクチャ設計書

> **3行サマリー**
> 1. プノンペン版SUUMO/Zillowを狙う構造化不動産PF。差別化＝「必須開示gateをDB層(`CHECK`制約+status遷移トリガー)で物理強制し、`negotiable`を入力不能にする」design-protection。エージェントは排除せず、KYC verified でないと物件を公開できない構造でheadhunter co-optする。
> 2. 本番スタック確定＝Next.js(App Router)+Supabase(Postgres/Auth/Storage/RLS+PostGIS)+地図API+Vercel+PWA+next-intl(EN/JA/KH)。新規repo `kh-property`、プロト`kh-property-proto`は営業道具として温存。供給サイド(エージェント登録→gate投稿→問い合わせ応答)を需要側UIより先に、W1-W4の4週で構築。
> 3. 規制方針＝免許は回避でなく真正クメール法人がPrakas 064を取得して運営。MVPは決済・エスクロー・課金を一切入れず「無料で供給資産(gate充足物件DB)を積むヘッジ」。title/メーター/税務は当社が判定・保証せず「自己申告＋Verify.gov.kh導線＋事実開示バッジ」に限定(研究正本7f: 設計保護P1≠金銭保護P3)。

関連正本: `workspace-config/sessions/2026-06-17_property_platform_research.md`(規制・競合リサーチ) / `PROPERTY_RENTAL_PLAYBOOK.md`(gate条件§1-§5) / プロト `Umagon-G/kh-property-proto`(SPEC.md・v0.7)

---

## 1. 概要 — 何を作るか・差別化の核

### 1.1 事業
プノンペン(カンボジア)版の構造化不動産プラットフォーム。Khmer24/FB Groups(無検証・紹介ベース・"negotiable"だらけ・Telegram移動でUX崩壊)が支配する「Craigslist段階」を、AI+gate条件スキーマ+検証で「Zillow段階」へleapfrogする。

### 1.2 差別化の核 = design-protection（3つの不変条件）
プロダクトの心臓は「構造上そうするしかできない」をUIではなく**DB層で物理的に強制**すること。アプリのバリデーションは回避され得るが、`CHECK`制約と`status遷移トリガー`はDBに刺さっていれば管理画面・直APIもバイパスできない＝design-protectionの最深層。

1. **gateが埋まらない限り公開できない**: `listing_status`の`published`遷移時に全Universal gate+業種モジュール必須項目+agent KYC verifiedを検査するトリガー。1つでもNULL/未充足なら`RAISE EXCEPTION`。
2. **`negotiable`をスキーマから消す**: `price_negotiable`カラムを存在させない。`rent_usd_month`は`published`時`NOT NULL`かつ`> 0`。「交渉可」を表現する手段をデータ型レベルで消す。
3. **エージェントの自由文は事実を覆せない場所にしか置けない**: 価格・gate値はスキーマ、自由文は物件詳細最下層の「出品者コメント枠」にのみ隔離(後述§4.7)。

### 1.3 戦略 = headhunter co-opt（エージェントを味方化）
エージェントを排除せず「在庫の持ち主」として遇する。`agents.kyc_status='verified'`でないと物件を公開できない＝エージェント登録が在庫公開の必須条件になる構造。罰ではなく「あなたの在庫がもっと回る集客ツール」として口説く。

### 1.4 二刀流の前提（白石承認済）
| | 現行プロト | 本番MVP |
|---|---|---|
| repo | `Umagon-G/kh-property-proto`(GitHub Pages) | `kh-property`(新規) |
| 性質 | 営業道具・使い捨て前提・**本番化しない** | 別途新規構築 |
| 技術 | 単一HTML/Leaflet+turf+OSM/ダミー1,740件 | Next.js+Supabase+PWA |
| 用途 | 投資家/松崎デモ(指なぞり録画) | β→本番 |

### 1.5 保護の2層（研究正本7f）
- **P1 = 設計保護**(MVP実装): 無料・liability無・「バッジ＝賠償」問題が消える。提示された選択肢からしか選べない構造。
- **P3 = 金銭保護**(将来): エスクロー/保証＝有料・賠償を負う・要第三者PSI免許。**今回スキーマに入れない**。

---

## 2. データモデル（Supabase / Postgres）

### 2.1 設計原則
- **Universal必須gateは物理カラム**(JSONBに逃がさない)。理由＝`published`時の`NOT NULL`/`CHECK`制約は物理列にしか刺さらない。可変な業種別モジュールだけJSONBに分離。
- **構造化gate値(数値/bool/enum)は言語非依存**＝listings本体に1回だけ持ち翻訳不要。これがSUUMO式「条件＝公表事実(数字)」のDB表現で、多言語コストをほぼゼロにする。
- **gate強制・KYC・連絡先leakage・事実開示/賠償の分離を全てDB+RLSで担保**。

### 2.2 テーブル一覧（14テーブル + ENUM群）

| # | テーブル | 役割 |
|---|---|---|
| 1 | `profiles` | 全ユーザー(auth.users 1:1拡張) |
| 2 | `agents` | エージェント事業者(KYC状態) |
| 3 | `agent_members` | profile↔agent 多対多(所属) |
| 4 | `listings` | 物件コア・全gateカラム |
| 5 | `listing_translations` | EN/JA/KH 自由文の翻訳 |
| 6 | `listing_business_modules` | 業種別モジュール宣言値(JSONB) |
| 7 | `media` | 写真(公開Storage参照) |
| 8 | `documents` | title/契約書/税書類(機密Storage) |
| 9 | `title_verifications` | Verify.gov.kh照合監査 |
| 10 | `verification_badges` | 検証バッジ(事実開示・賠償でない) |
| 11 | `inquiries` | 問い合わせスレッド(1物件1借主=1本) |
| 12 | `messages` | スレッド内メッセージ |
| 13 | `locations` | 行政区域マスタ(クラスタ/集計) |
| 14 | `audit_log` | 全状態遷移・検証アクションの監査 |

補助ENUM: `user_role`, `agent_kyc_status`, `listing_status`, `deal_type`, `usage_type`, `property_type`, `module_type`, `meter_type`, `utility_pricing`, `media_type`, `doc_type`, `verification_kind`, `inquiry_status`。

### 2.3 状態機械（3エンティティ）
- **Agent**: `unverified → kyc_draft → kyc_submitted → kyc_approved / kyc_rejected`
- **Listing**: `draft → pending_review(=ready_to_publish) → published(live) → (rejected / archived(paused) / rented / expired)`
- **Inquiry**: `open(new) → responded(replied) → closed / spam`
- 不変条件: `listing.status ∈ {published以降}` には**全gate必須項目=非null**が前提(DB制約+UI二重ガード)。

### 2.4 主要テーブル定義

#### profiles（全ユーザー）
```
id            uuid PK  references auth.users(id) on delete cascade
role          user_role NOT NULL default 'seeker'   -- seeker|agent_owner|agent_staff|landlord|admin
full_name     text
phone         text                                   -- KYC用(E.164)
phone_verified bool NOT NULL default false
preferred_locale text NOT NULL default 'en'          -- en|ja|km
created_at    timestamptz NOT NULL default now()
```
RLS: 本人(`id = auth.uid()`)のみ select/update。admin全権。`role`昇格はトリガーで監査。

#### agents（供給サイドの主役）
```
id              uuid PK default gen_random_uuid()
owner_id        uuid NOT NULL references profiles(id)
company_name    text NOT NULL
license_no      text                                    -- Prakas 064 不動産サービス免許番号
kyc_status      agent_kyc_status NOT NULL default 'pending'  -- pending|submitted|verified|rejected
kyc_id_doc_id   uuid references documents(id)           -- 身分証(機密)
business_reg_doc_id uuid references documents(id)       -- 事業者登録(機密)
contact_email   text NOT NULL
contact_phone   text NOT NULL
verified_at     timestamptz
verified_by     uuid references profiles(id)            -- admin
created_at      timestamptz NOT NULL default now()
```
KYC強制: 物件を`published`にするトリガーが`kyc_status='verified'`を要求。未KYCはdraft投稿可・公開不可＝headhunter co-optの入口。

#### agent_members（所属・複数スタッフ運用）
```
agent_id    uuid references agents(id) on delete cascade
profile_id  uuid references profiles(id) on delete cascade
role_in_agency text NOT NULL default 'staff'   -- owner|manager|staff
PRIMARY KEY (agent_id, profile_id)
```

#### listings（コア・全gateカラムを物理列で）
プロトの Khmer24準拠2軸(取引×用途)+物件タイプ10種を継承。
```
-- 識別・所有
id              uuid PK default gen_random_uuid()
agent_id        uuid NOT NULL references agents(id)
created_by      uuid NOT NULL references profiles(id)
status          listing_status NOT NULL default 'draft'

-- 取引×用途 2軸
deal_type       deal_type NOT NULL           -- rent | sale
usage_type      usage_type NOT NULL          -- residential | business | land
property_type   property_type NOT NULL       -- house|villa|apartment|condo|studio|office|shop|shophouse|warehouse|land(10種)

-- §1 商業条件 gate（publish時 NOT NULL を CHECK で強制）
rent_usd_month      numeric(12,2)            -- rent時必須・>0・negotiable列は無し
sale_price_usd      numeric(14,2)            -- sale時必須・>0
deposit_months      numeric(4,1)             -- デポ月数(必須)
deposit_refund_terms text                    -- 返金条件(必須・自由文→翻訳テーブルへも)
lease_term_years     numeric(4,1)            -- 契約年数(rent時必須)
rent_fixed_full_term bool                    -- 全期間賃料固定か(rent時必須)
renewal_rent_clause  text                    -- 更新時据置の明記有無(rent時必須)
vat_invoice_available bool NOT NULL          -- 税インボイス(VAT)発行可否 ← 必須gate
withholding_tax_bearer text                  -- 源泉税10%負担者 'landlord'|'tenant'|'shared'

-- §2 物件適合 gate(Universal)
commercial_use_allowed bool                  -- 商業利用許可/用途(business時必須)
area_sqm            numeric(10,2) NOT NULL    -- 専有面積(必須)
parking_slots       int
available_from      date NOT NULL             -- 入居可能日(必須)
internet_available  bool

-- §3 光熱費 gate(握れない構造を作らない核心)
elec_meter_type     meter_type NOT NULL       -- dedicated|sub|fixed|none ← 専用メーター
water_meter_type    meter_type NOT NULL
elec_pricing        utility_pricing NOT NULL  -- govt | landlord_markup
elec_unit_usd       numeric(6,4)              -- markup時必須
water_pricing       utility_pricing NOT NULL
water_unit_usd      numeric(6,4)
meter_reading_method text

-- §4 法務 gate
title_deed_type     text                      -- hard|soft|strata|null
contract_reviewable bool                      -- 署名前全文レビュー可
notarization_supported bool

-- 地図検索(PostGIS)
location_id     uuid references locations(id)
geom            geography(Point,4326)         -- 緯度経度・GiSTインデックス
address_line    text

-- ライフサイクル
published_at    timestamptz
expires_at      timestamptz                   -- 鮮度管理(Zillow式)
created_at      timestamptz NOT NULL default now()
updated_at      timestamptz NOT NULL default now()
```

**publish強制トリガー(抜粋)** — gate強制の心臓:
```sql
-- status='published' の時のみ発火
IF NEW.status = 'published' THEN
  IF NEW.deal_type='rent' AND (NEW.rent_usd_month IS NULL OR NEW.rent_usd_month<=0
      OR NEW.deposit_months IS NULL OR NEW.lease_term_years IS NULL
      OR NEW.rent_fixed_full_term IS NULL OR NEW.renewal_rent_clause IS NULL)
    THEN RAISE EXCEPTION 'GATE_FAIL: commercial terms incomplete';
  IF NEW.deal_type='sale' AND (NEW.sale_price_usd IS NULL OR NEW.sale_price_usd<=0)
    THEN RAISE EXCEPTION 'GATE_FAIL: sale price required';
  IF NEW.elec_meter_type IS NULL OR NEW.water_meter_type IS NULL
    THEN RAISE EXCEPTION 'GATE_FAIL: meter disclosure required';
  IF NEW.area_sqm IS NULL OR NEW.available_from IS NULL OR NEW.vat_invoice_available IS NULL
    THEN RAISE EXCEPTION 'GATE_FAIL: universal fields incomplete';
  IF NOT fn_business_module_complete(NEW.id, NEW.usage_type) THEN
    RAISE EXCEPTION 'GATE_FAIL: business module incomplete'; END IF;
  IF (SELECT kyc_status FROM agents WHERE id=NEW.agent_id) <> 'verified'
    THEN RAISE EXCEPTION 'GATE_FAIL: agent not KYC-verified';
END IF;
```

> **gate粒度の整合メモ**: データモデル(§2)は上記の全項目を`published`必須としてトリガーに書ける設計。一方、事業計画(§6)のW2スコープでは**公開最低要件を4項目(写真・賃料最終値・面積・外国人契約可否)に絞り、残りgateは「上位表示誘導」**で運用負荷を下げる立て方を採る。**MVP v1の起動時パラメータとして「強制する必須gateの集合」を絞れる設計**(トリガーが参照する必須キー集合を運用可変にする)を採用し、供給が積まれるに従って必須範囲を広げる。この閾値設計は§7の未決事項。

インデックス: `GIST(geom)`(地図/囲い検索), `(status, deal_type, usage_type)`複合, `(location_id)`, `(expires_at) WHERE status='published'`。

#### listing_translations（多言語・別テーブル方式 / 確度85%）
| 方式 | 採否 | 理由 |
|---|---|---|
| 別テーブル(採用) | ✅ | ①locale単位の部分翻訳・AI翻訳追記(中国語後付け)が自然 ②`translation_source`(human/ai)と`is_reviewed`を行で持てる→ネイティブレビュー必須要件に直結 ③per-locale FTSインデックス ④欠落locale=行が無いだけでfallback容易 |
| JSONB | ✗ | 翻訳元/レビュー状態をフィールド毎に持てない・部分更新が重い・per-locale FTS不可 |

```
listing_id   uuid references listings(id) on delete cascade
locale       text NOT NULL                 -- en|ja|km|zh(後付け)
title        text NOT NULL
description   text
deposit_refund_terms_i18n text
renewal_clause_i18n text
translation_source text NOT NULL default 'human'  -- human|ai_gemini|ai_other
is_reviewed  bool NOT NULL default false    -- ネイティブレビュー済か
reviewed_by  uuid references profiles(id)
PRIMARY KEY (listing_id, locale)
```
原則: 翻訳テーブルは**自由文だけ**。構造化gate値はlistings本体に1回。

#### listing_business_modules（業種別モジュール・ここだけJSONB）
```
listing_id   uuid references listings(id) on delete cascade
module_type  module_type NOT NULL    -- food_logistics|fnb|retail|clinic|office|residential
attrs        jsonb NOT NULL           -- {floor_load_pallets:.., container_40ft_access:bool, gas_supply:bool ...}
PRIMARY KEY (listing_id, module_type)
```
`fn_business_module_complete()`がmodule_type別の必須キー集合(例: food_logistics → `floor_load_pallets`/`container_40ft_access`/`cold_storage`)をJSONBの`? key`演算子で検査。必須定義は`module_required_keys`小テーブルで持ち、業種要件追加をコード変更なしで運用可能にする。**MVP v1は3種(F&B/小売/オフィス)を実装、残り業種はP2**。

#### media（公開Storage）
```
id          uuid PK
listing_id  uuid references listings(id) on delete cascade
storage_path text NOT NULL            -- Supabase Storage bucket 'listing-media'
media_type  media_type NOT NULL       -- photo|floorplan|video
sort_order  int NOT NULL default 0
width/height int
created_at  timestamptz default now()
```
Storage: bucket `listing-media` = **public read**。writeは所属エージェントのみ(パスprefix=`{listing_id}/`を`agent_members`照合)。

#### documents（機密Storage）
```
id          uuid PK
owner_scope text NOT NULL             -- 'agent'|'listing'
agent_id    uuid references agents(id)
listing_id  uuid references listings(id)
storage_path text NOT NULL            -- bucket 'secure-docs' (PRIVATE)
doc_type    doc_type NOT NULL         -- title_deed|lease_contract|tax_invoice|kyc_id|business_reg
uploaded_by uuid references profiles(id)
created_at  timestamptz default now()
```
Storage: bucket `secure-docs` = **private**。署名付きURL発行Edge Function経由のみ。title deed・身分証・契約書は匿名閲覧不可。

#### title_verifications（Verify.gov.kh 照合監査）
```
id          uuid PK
listing_id  uuid references listings(id) on delete cascade
document_id uuid references documents(id)
result      text NOT NULL                          -- original|fake|inconclusive
verify_source text NOT NULL default 'verify.gov.kh'
checked_by  uuid references profiles(id)
checked_at  timestamptz NOT NULL default now()
raw_response jsonb
```
title検証は**事実の記録**であって賠償保証ではない(研究正本7f)。

#### verification_badges（バッジ=事実開示・賠償でない）
```
id          uuid PK
listing_id  uuid references listings(id) on delete cascade
kind        verification_kind NOT NULL   -- title_verified|dedicated_meter|govt_utility_rate|vat_invoice_ok|fixed_rent|notarizable
granted_by  uuid references profiles(id)
evidence_ref uuid                         -- title_verifications.id 等
granted_at  timestamptz default now()
revoked_at  timestamptz
```
バッジは検証/gate値から**導出**(例: `elec_meter_type='dedicated'`→`dedicated_meter`)。「保証」文言を持たせない。金銭保証(P3)は将来`escrow_holds`テーブルで追加。

#### inquiries / messages（leakage対策の要）
```
-- inquiries
id          uuid PK
listing_id  uuid NOT NULL references listings(id)
seeker_id   uuid NOT NULL references profiles(id)
agent_id    uuid NOT NULL references agents(id)   -- 非正規化(高速権限判定)
status      inquiry_status NOT NULL default 'open'
created_at  timestamptz default now()
UNIQUE(listing_id, seeker_id)                       -- 1物件1借主=1スレッド

-- messages
id          uuid PK
inquiry_id  uuid references inquiries(id) on delete cascade
sender_id   uuid references profiles(id)
body        text NOT NULL
contact_shared bool NOT NULL default false   -- 連絡先開示の監視(leakage)
created_at  timestamptz default now()
```
**leakage対策(Airbnb式)**: エージェント実連絡先は`agents`に持つが、`v_public_listings` Viewから除外。問い合わせ成立後にプラットフォーム内messagingで会話。`contact_shared`で場外共有を監視。

#### locations（行政区域マスタ・クラスタ集計）
```
id          uuid PK
name_en/name_km/name_ja text
admin_level int                      -- 1=province,2=khan/district,3=sangkat/commune
parent_id   uuid references locations(id)
boundary    geography(MultiPolygon,4326)  -- 行政区域ポリゴン
centroid    geography(Point,4326)
```
プロトの「行政区域クラスタ+件数」をDB化。`ST_Within(listing.geom, location.boundary)`で集計、地図ズームで`admin_level`切替。

#### audit_log（監査）
```
id          uuid PK
actor_id    uuid references profiles(id)
entity      text NOT NULL            -- listing|agent|title_verification|...
entity_id   uuid NOT NULL
action      text NOT NULL            -- status_change|kyc_verify|title_check|badge_grant|publish_reject
before/after jsonb
created_at  timestamptz default now()
```
gate拒否(`GATE_FAIL`)・KYC承認・title検証・publish/rejectを全記録。Prakas 064監督下の説明責任+design-protection証跡。

### 2.5 地図検索（PostGIS採用 / 確度90%）
| 方式 | 採否 | 理由 |
|---|---|---|
| PostGIS `geography`+GiST(採用) | ✅ | プロトの「指でなぞる囲い検索」=任意ポリゴン。`ST_Within(geom, ST_GeomFromGeoJSON(:polygon))`でturf.jsの`booleanPointInPolygon`をサーバ側で正確再現。bbox一次絞り→ポリゴン精査の2段。Supabaseは`create extension postgis`で即有効化 |
| bboxのみ | △補助 | 矩形のみ。一次絞り込みとして併用 |

RPC: `search_listings_in_polygon(geojson, filters jsonb)`をSupabase Functionで公開。フロントはturfで描いた座標列をGeoJSONで渡すだけ。**MVP v1は実装簡略化のためbounds(矩形/円)検索から開始し、turf自由図形→PostGIS ST_Within はv2で本格移植**(§6 X7参照)。

### 2.6 RLS 全体方針
| テーブル | 匿名read | seeker | agent所属 | admin |
|---|---|---|---|---|
| listings(published) | ✅(連絡先除くView) | ✅ | 自社全status | 全件 |
| listings(draft等) | ✗ | ✗ | 自社のみ | ✅ |
| profiles | ✗ | 本人 | 本人 | ✅ |
| agents | 公開情報のみView | read | 自社編集 | ✅ |
| documents/secure-docs | ✗ | ✗ | 自社のみ | ✅ |
| inquiries/messages | ✗ | 自分の | 自社宛 | ✅ |
| title_verifications | バッジ経由の結果のみ | ✗ | 自社read | ✅ |

公開面は必ず`SECURITY DEFINER`の薄いView/RPC(`v_public_listings`)経由にし、連絡先・機密doc・draftをRLSとViewの二重で遮断。**全テーブルRLS ON必須**(RLS無しのロール制御は穴)。

---

## 3. 技術スタック & リポジトリ構成

### 3.1 確定スタック
- **package manager**: pnpm / **Node 22 LTS** / **TypeScript strict**
- **UI**: Tailwind + shadcn/ui(地図以外を速く作る。プロトのネイビー`#0B2138`/アンバーをdesign tokenに移植)
- **本番repo**: `kh-property`(新規)。プロト`kh-property-proto`は営業道具として温存
- **DBスキーマ正本** = `supabase/migrations/`のSQL。zod schema(`features/listings/schema.ts`)はフォーム/server action入力検証の正本。両者を1対1対応させ、gate条件を「DB NOT NULL制約」と「zod required」の二重で強制＝設計保護をコードで担保。

### 3.2 ディレクトリ構成（route groupで公開/認証/管理を分離）
```
kh-property/
├─ src/
│  ├─ app/
│  │  ├─ [locale]/                    # 全ページlocale配下(next-intl必須構造)
│  │  │  ├─ layout.tsx                # setRequestLocale + NextIntlClientProvider
│  │  │  ├─ (public)/                 # 未認証で見える層(=需要側UI。枠だけ先に切る)
│  │  │  │  ├─ page.tsx               # 地図検索トップ(turf囲い検索の移植先)
│  │  │  │  ├─ search/page.tsx
│  │  │  │  └─ listings/[id]/page.tsx # 物件詳細(全gate条件開示=agent-killer)
│  │  │  ├─ (agent)/                  # ★最優先実装。エージェント供給サイド
│  │  │  │  ├─ layout.tsx             # requireRole('agent') ガード
│  │  │  │  ├─ dashboard/page.tsx
│  │  │  │  ├─ listings/new/page.tsx  # gate条件必須スキーマ投稿フォーム
│  │  │  │  ├─ listings/[id]/edit/page.tsx
│  │  │  │  └─ inquiries/page.tsx
│  │  │  ├─ (auth)/                   # ログイン/サインアップ/KYC
│  │  │  │  ├─ login/page.tsx ├─ signup/page.tsx
│  │  │  │  └─ onboarding/page.tsx    # エージェントKYC
│  │  │  └─ (admin)/                  # 物件審査/エージェント承認/検証バッジ付与
│  │  │     └─ layout.tsx             # requireRole('admin')
│  │  ├─ api/                         # 例外的にREST必要な所のみ
│  │  │  ├─ webhooks/[provider]/route.ts
│  │  │  └─ og/[id]/route.ts          # 物件OGP画像動的生成
│  │  ├─ manifest.ts                  # PWA manifest(App Router native)
│  │  └─ sitemap.ts / robots.ts
│  ├─ proxy.ts                        # ★ next-intl middleware(Next16でmiddleware.ts→proxy.ts)
│  ├─ i18n/{routing,request,navigation}.ts
│  ├─ messages/{en,ja,km}.json        # 翻訳(名前空間別にネスト)
│  ├─ features/                       # ドメイン機能(コロケーション)
│  │  ├─ listings/{actions.ts, schema.ts(zod gate正本), components/}
│  │  ├─ map/  ├─ agents/  ├─ inquiries/  └─ verification/
│  ├─ lib/
│  │  ├─ supabase/{server,client,middleware}.ts  # @supabase/ssr 3分割
│  │  └─ auth/requireRole.ts
│  └─ components/ui/                   # shadcn/ui
├─ supabase/migrations/ + seed.sql     # RLSポリシー含む
├─ scripts/translate.ts               # AI翻訳パイプライン
├─ next.config.ts                     # next-intl plugin + PWA(Serwist)
└─ vercel.json
```
route group命名: `(public)`/`(agent)`/`(auth)`/`(admin)`。括弧付きはURLに出ない。各group `layout.tsx`でロールガードを集約。

### 3.3 Server Actions vs Route Handlers
| 技術 | 用途 |
|---|---|
| Server Actions(既定) | フォーム送信全般=物件投稿/編集、エージェント登録、問い合わせ送信/返信、検証申請。RLSと相性良くCSRF/型安全 |
| Route Handlers(例外のみ) | ①外部Webhook受信(決済PSI/KYCプロバイダ/メール) ②OGP動的画像 ③将来のモバイル/第三者向けpublic API ④Vercel Cron |

原則: 画面内の変更は全部server action、外部システムが叩く口だけRoute Handler＝APIサーフェス最小化。

### 3.4 認証フロー（Supabase Auth・3ロール）
ロールは`app_metadata.role`(JWTクレーム)に保持(`user_metadata`はユーザー改竄可で不可)。`profiles`にも冗長保持しRLSで参照。

| ロール | 取得経路 | 権限 |
|---|---|---|
| `seeker`(一般) | サインアップ即時 | 検索/閲覧/問い合わせ送信 |
| `agent` | サインアップ→KYC提出→admin承認で昇格 | 自分の物件CRUD・問い合わせ応答。**承認前は公開不可** |
| `admin` | 手動付与(seed/Supabaseダッシュボード) | 物件審査・検証バッジ付与・エージェント承認 |

- **認証方式**: メール+パスワード/Magic Link両対応。Google OAuthを一般向けに初期から(摩擦最小)。電話OTP(SMS)は将来。
- **@supabase/ssr**でcookieベースセッション。3分割(`server,client,middleware`)が公式パターン。`proxy.ts`で next-intl ルーティング→続けて`supabase.auth.getUser()`でcookieリフレッシュ。
- **ロールガード=2層**: ①server layoutの`requireRole()`(UX用リダイレクト) ②Supabase RLS(本丸の防御)。

### 3.5 地図API — 推奨と未確定の明示
> **⚠️ セクション間で推奨が対立。本書では「描画はMapbox推奨だが、コスト最優先のMVP起動時はMapLibre+MapTilerでも可」とし、最終確定を白石判断(§7)に送る。**

- **テックリード推奨(確度80%) = Mapbox GL JS**。理由: ①プロトのturf囲い検索がWebGL+GeoJSONでほぼ無改造移植 ②大量ピン(1,700+)がWebGLで段違いに軽い ③無料枠5万loads/月・超過$5/1,000(Google比約30%安)。Googleが勝つ唯一の軸=プノンペンのPOI/店舗名/Street Viewだが、本PFは「物件を緯度経度で持つ」モデルでGoogleのPOI依存が低い。**住所オートコンプリートだけGoogle Places APIを部分併用**(描画=Mapbox/住所補完=Google)。
- **事業計画側のコスト最小案 = MapLibre + MapTiler/OSM(無料)**。プロトのLeaflet資産と親和。カンボジアOSMデータは道路網が実用十分。
- **整合の立て方**: どちらも「turf互換・GeoJSONベース・OSSレンダラ寄せ可能」で移植ロジックは共通。**MapLibre+MapTilerで$0開始 → トラフィック/POI要件が出たらMapboxへスタイル差替**という段階移行が両案を両立させる。最終1本化は§7。
- 環境変数: `NEXT_PUBLIC_MAPBOX_TOKEN`(or MapTiler key・URL制限付きpublic), `GOOGLE_PLACES_API_KEY`(server側プロキシ)。

### 3.6 turf囲い検索の移植方針（プロトのコア資産）
プロト`index.html`の load-bearing ロジック(実コード確認済):
- `inArea(p){return area?turf.booleanPointInPolygon(turf.point([p.lng,p.lat]),area):true;}`
- pointerdown/move/up で freehand → polygon(始点を末尾にpushして閉じる)
- `SETS={deal,use,ptype}` のSetベース複数選択フィルタ+ヘッダー↔パネル同期

移植方針(地図ライブラリ移行・ロジック温存):
1. **freehand描画**: Leaflet pointer→地図APIのcanvas座標。描画中は`dragPan.disable()`。離したら閉ポリゴンGeoJSON。
2. **点内判定 = turf.js は1行も変えない**(地図ライブラリ非依存)。サーバ側でも同じturfを使い、囲いポリゴンをserver actionに渡しDB側で絞る＝1,740件超でも破綻しない設計に格上げ。
3. **段階アーキ**: 初期はクライアント側turf即時プレビュー → データ増でPostGIS `ST_Within`へ。二段構え。
4. **複数選択フィルタ**: Setパターン温存・状態管理はURL検索パラメータ+nuqsに載せ替え(共有URL/戻る進む/SSR可)。
5. **「デモエリア」ワンタップ円**も営業デモ用に温存移植。

### 3.7 i18n（next-intl v4・EN/JA/KH）
- `i18n/routing.ts`: `locales:['en','ja','km'], defaultLocale:'en', localePrefix:'as-needed'`(英語prefix無し、ja/kmは`/ja` `/km`)。
- `proxy.ts`で`createMiddleware(routing)`→Supabaseセッションrefresh。matcherでAPI/静的を除外。
- 各layout/pageで`setRequestLocale(locale)`を呼びSSG可能に(物件詳細はSEOで静的化)。
- 翻訳ファイルは名前空間ネスト(`agent`/`listing`/`gate`)。gateラベル(専用メーター/税インボイス可否等・プロトの🏛️/⚙️タグ説明)は`gate`名前空間に集約。
- **AI翻訳パイプライン**(`scripts/translate.ts`):
  1. 正本=`messages/en.json`(英語で書く)。
  2. CIでキー差分検出→Claude API(`claude-haiku`で十分)で「不動産PF・クメール/日本語の自然訳」プロンプトで欠損キーのみ埋める(人手修正を上書きしないマージ)。
  3. **クメール語はネイティブレビュー必須**(`is_reviewed:false`フラグ運用。memory: project_bfc_website_translation_review 準拠)。
  4. 物件の自由記述文は投稿時にserver actionでAI翻訳し`listing_translations`保存。中国語は`zh` locale追加+同パイプラインで拡張。

### 3.8 PWA化
- `app/manifest.ts`(native・型付き): name/icons(192/512/maskable)/`display:standalone`/`theme_color:#0B2138`。
- service worker = **Serwist**(next-pwa後継・App Router対応)。地図タイル/static は stale-while-revalidate、物件APIは network-first。
- オフライン方針: 初期は「最後に見た物件/お気に入りの閲覧」程度。**ネイティブアプリは後期**、まずホーム画面追加体験まで。
- 注意: Vercel+Serwistはビルド時SW生成。`next.config.ts`でnext-intl pluginと合成(chain順序に注意)。

### 3.9 Vercelデプロイ / 環境変数 / CI
- ホスティング=Vercel。Supabaseは別管理(Supabase Cloud)。商用はVercel Pro必須($20/月/席)。
- 環境変数(3環境分離):
  | 変数 | 公開 | 用途 |
  |---|---|---|
  | `NEXT_PUBLIC_SUPABASE_URL`/`..._ANON_KEY` | public | クライアントSupabase |
  | `SUPABASE_SERVICE_ROLE_KEY` | server only | server action/admin処理 |
  | `NEXT_PUBLIC_MAPBOX_TOKEN`(or MapTiler) | public(URL制限) | 地図描画 |
  | `GOOGLE_PLACES_API_KEY` | server only | 住所補完プロキシ |
  | `ANTHROPIC_API_KEY` | server only | AI翻訳 |
- プレビュー: Vercel Preview Deployment(PR毎URL)。Supabaseはpreview用に別プロジェクト/branch DB(本番データを触らせない)。
- CI: GitHub Actions 1ワークフロー(`pnpm install→typecheck→lint→build→i18nキー差分チェック`)。テストは初期サーバアクションのzod単体(vitest)のみ。`supabase db diff`でマイグレーション漏れ検出、本番DB適用は手動承認(階層2)。デプロイはVercel Git連携(二重ビルド回避)。

---

## 4. 供給サイド（エージェント）UXフロー

### 4.1 設計の核
1. gate条件が埋まらない限り「公開」ボタンは物理的に押せない(disable+理由表示)。`negotiable`はフィールドとして存在させない。
2. エージェントを「在庫の持ち主」として遇する(headhunter co-opt)。自分の物件が地図に載り反響が来るダッシュボードを与える。
3. エージェントの自由文は「構造化開示を上書きできない場所」にしか置けない。

### 4.2 画面一覧（供給サイド）
| # | 画面 | 目的 | 主要ステート |
|---|------|------|------------|
| S0 | オンボーディングLP | 「ここにあなたの物件が載る」 | 未ログイン |
| S1 | サインアップ(電話/メール+OTP) | アカウント生成 | `unverified` |
| S2 | KYC入力(本人+事業者情報) | Prakas064準拠の本人確認 | `kyc_draft`→`kyc_submitted` |
| S3 | KYC審査ステータス | 審査の見える化・離脱防止 | `kyc_submitted`/`approved`/`rejected` |
| S4 | エージェント・ダッシュボード | 物件/問い合わせ/反響の集約 | 常時 |
| S5 | 物件投稿フォーム(gate必須・5ステップ) | 構造化開示の強制入力 | `listing_draft` |
| S6 | 投稿プレビュー&公開確認 | 公開前最終確認 | `ready_to_publish` |
| S7 | 物件管理(一覧/個別) | 公開審査状況・編集・停止 | `in_review`/`live`/`paused`/`rejected`/`rented` |
| S8 | 問い合わせ受信トレイ | 需要側問い合わせ束ね | `new`/`open`/`replied`/`closed` |
| S8a | 個別スレッド | 開示を壊さない返信 | スレッド状態 |
| S9 | 反響アナリティクス | 表示/保存/問い合わせ率で継続動機化 | 集計のみ |
| S10 | プロフィール/設定 | 表示名・バッジ・通知・言語 | — |

EN/JA/Khmer 3言語トグルは全画面ヘッダ常設。

### 4.3 フロー全体図
```
[S0 LP] →口説き→ [S1 SignUp:unverified]
   → [S2 KYC:kyc_draft] →submit→ [kyc_submitted]
       → 審査 → kyc_approved → [S4 Dashboard]
              → kyc_rejected → S3で理由+再提出
[S4] →新規投稿→ [S5 listing_draft]
   (gate全項目クリアまで "公開" disabled)
   → [S6 ready_to_publish] →公開→ [in_review]
       → 検収 → live   (地図/検索に表示)
              → rejected → S5で修正再投稿
[live] →需要側問い合わせ→ [S8 inbox:new] → [S8a thread] →構造化返信→ replied
[live] →成約→ agentが rented にマーク(在庫鮮度の核)
```
KYC審査と公開審査は**別ゲート**。KYC=人の信頼(1回)、公開審査=物件1件ごとの開示完全性チェック(毎回)。

### 4.4 S2 KYC入力（Prakas064/研究§6準拠・最小化）
金融eKYCでなく「出品者本人確認」レベル(資金を扱わない＝重いPSI型KYC不要)。

| グループ | フィールド | 必須 | 保存方針 |
|---------|----------|------|---------|
| 本人 | 氏名(KH/ローマ字) | ● | 平文可(公開は表示名のみ) |
| 本人 | 携帯(OTP済) | ● | 保存・公開は「PF経由」優先 |
| 本人 | 身分証種別+画像 | β=任意→検証バッジ申請時=必須 | 暗号化・非公開・表示しない・OCR後破棄も検討 |
| 本人 | 顔写真(セルフィー) | ○ | なりすまし抑止 |
| 事業者 | 立場(個人/会社/オーナー直/開発者) | ● | 平文・広告主体の明示(Prakas064) |
| 事業者 | 会社名/ライセンス番号 | 条件●(会社) | 平文 |
| 事業者 | 事業所住所 | ● | 平文 |
| 同意 | 開示ルール同意(最終条件のみ・虚偽掲載で停止) | ● | 同意日時・バージョン記録 |

設計上の鍵: KYC審査中でもS5物件投稿フォームは下書き可(離脱防止)。公開はKYC承認後のみ。需要側はゲスト問い合わせ可(KYC不要)。

**個人情報保護**: 収集最小化(生年月日・住所・銀行情報は集めない)。公開プロフィール=表示名・種別・対応エリア・バッジのみ。電話/メール/ID/事業者番号は非公開テーブルに分離。登録解除時PII削除(リスティングは匿名化して履歴保持可)。

### 4.5 S5 物件投稿フォーム — gate必須スキーマ（PLAYBOOK §1-§5を5ステップに変換）
各ステップ完了でチェックが緑、全緑までS6の「公開申請」がdisabled。

**Step1 基本(プロト2軸踏襲)**: 取引種別(賃貸/売買・排他)/用途(住居/事業用/土地)/物件タイプ10種/専有面積(㎡)/入居可能日/位置(地図ピン+行政区域)。

**Step2 商業条件gate(§1) — negotiable禁止の主戦場**:
| フィールド | 必須 | バリデーション |
|-----------|----|----------------|
| 月額賃料 or 売買価格 | ● | テキスト不可・「交渉可」入力欄が存在しない。数値のみ |
| 賃料 全期間固定か | ●(賃貸) | Noなら更新時値上げ条件を別途必須化 |
| 更新時賃料据置 契約明記 | ●(賃貸) | 口約束無効をUIで明示 |
| デポジット月数 | ●(賃貸) | — |
| デポ返金条件 | ●(賃貸) | 満額/控除条件あり |
| 税インボイス(VAT)発行可否 | ● | 事業テナントのVAT控除可否=事業用フィルタの肝 |
| 家賃源泉税(10%)負担 | ●(事業用) | 大家/借主 |

**Step3 物件適合gate(§2)+業種別モジュール(§2b)**: 商業利用可否/インターネット回線/駐車台数/修繕清掃オーナー負担/業種タグ選択(選ぶと業種別必須項目が動的追加 — F&B→ガス/換気/グリストラップ、食品→40ft進入/床荷重/冷蔵)。

**Step4 光熱費gate(§3) — 測定権の開示**: 電気専用メーター(Yes/No)/水専用メーター/電気単価+(政府~$0.15-0.20/大家マークアップ)/水単価/メーターなし時の代替(月額固定/サブメーター設置)。

**Step5 法務gate(§4)+売買用title**: 契約年数/中途解約条項/売却時賃借権存続条項/(売買時)title種別(ハード/ソフト/ストラタ・外国人所有可否に直結)/Verify.gov.kh照合(未/照合済)/外国人取得可否(可/ノミニー要/不可)/写真3枚以上(釣り広告防止)。

### 4.6 公開ボタンのバリデーションUX（design-protectionの実装）
- 常時表示の進捗バー「公開まで あと3項目」。各gateグループにチェック/赤バッジ。
- 「公開申請」ボタンは未充足の間`disabled`+ツールチップ「未入力: 税インボイス可否, 電気メーター, 契約年数」をクリックで該当Stepへジャンプ。
- 「後で交渉」系の逃げ道をUIから削除(価格・デポ・単価は必須数値。空欄/「相談」テキスト不可)。
- 保存(下書き)は常時可だが`listing_draft`では検索に一切出ない。
- 全緑→活性化→押下で`in_review`(公開審査キュー)。審査観点: 写真と開示値の整合、明らかな虚偽(政府料金の3倍単価等)、禁止カテゴリ。AI一次自動判定→グレーのみ人手。

### 4.7 エージェントの自由文の隔離（物件詳細を3層分離）
1. **gate値ブロック(最上部・改変不可の事実)**: 価格/契約年数/デポ/メーター/VAT/用途をバッジ表形式。エージェントは作文できない。
2. **検証バッジブロック**: title照合/メーター公式性/VAT可を「事実開示」表示(P1=賠償でなく事実)。
3. **エージェント・コメント枠(最下部・「出品者コメント」)**: ここだけ自由文OK。ただし①gate値と矛盾する記述はAI検知で公開前にブロック/警告 ②「交渉応相談」等の価格曖昧表現は禁止ワードでブロック ③周辺環境/内装/入居柔軟さの推奨アピールは歓迎。
= 自由文は「事実を覆さず、事実に色を付ける」範囲に物理制限。

### 4.8 S8a 問い合わせ応答（開示を壊さない返信）
| 仕組み | 内容 |
|--------|------|
| 定型クイック返信 | 「内見可能日」「追加写真」「契約書サンプル」をボタン化 |
| 価格再交渉はスキーマ更新のみ | チャットで「$50引く」を書いても掲載価格は変わらない。値下げは価格フィールド編集=全員同じ最終価格 |
| 自由文は補足のみ | gate値と矛盾する記述をAIが検知し警告「掲載と異なる条件を提示しています。掲載を修正しますか?」 |
| Telegram誘導の抑止 | 「外部連絡先の早期交換は反応スコアに反映されない」+場外移動に注意フラグ(leakage耐性) |

各問い合わせに「どの物件・どのgate値を見て来たか」を表示(例「事業用・VAT可・専用メーターで絞り込み」)。

### 4.9 S4 ダッシュボード（継続利用の動機装置）& 在庫鮮度ループ
| カード | 内容 |
|--------|------|
| マイ物件 | live/in_review/draft/paused/rented 件数バッジ+サムネ |
| 問い合わせ | 未読数(`new`)赤バッジ・SLA「24h以内返信で反応率↑」 |
| 反響(今週) | 物件別の表示数/保存数/問い合わせ数/問い合わせ率 |
| アクション喚起 | 「写真が少ない物件」「30日更新なしで鮮度低下」 |

**在庫鮮度ループ**: 成約したら`rented`にマークするとダッシュボードの信頼バッジが上がる→死んだ広告が自然に消える(Khmer24の最大の穴を直す)。

### 4.10 S0 オンボーディング（headhunter co-opt）
| セクション | メッセージ |
|-----------|----------|
| ヒーロー | 「あなたの物件を、地図で指でなぞって探す客に届ける」+プロトの囲い検索デモ流用 |
| 価値1 | 「Khmer24より上に出る・釣り広告と並ばない=本気の客だけ来る」 |
| 価値2 | 「問い合わせ・反響が数字で見える」 |
| 価値3 | 「成約マークで信頼バッジ=次の客が増える」 |
| 安心 | 「掲載は無料(P1設計保護)。エスクロー等は任意の後付け」=参入摩擦ゼロ |
| CTA | 「3分で登録→最初の物件を載せる」→S1 |

### 4.11 需要側（MVP最小・供給の反響を成立させるだけ）
| 画面 | 出すもの | 出さないもの |
|------|---------|-------------|
| D1 検索/地図 | 囲い検索+2軸フィルタ(取引×用途)+gate値フィルタ(VAT可/専用メーター/外国人可/契約年数) | アカウント不要で閲覧可 |
| D2 物件詳細 | §4.7の3層(gate値→バッジ→出品者コメント) | エスクロー/決済導線なし |
| D3 問い合わせ | gate値を見た上での問い合わせフォーム→S8へ | アカウント任意(電話で可) |

需要側でgate値フィルタを出すことが供給側に「だから全項目埋めろ」の圧を生む(構造で供給を躾ける)。

---

## 5. 規制整合（Prakas 064 / KYC / 問い合わせ動線 / title / 規約）

**基本方針**: 真正クメール法人がPrakas 064免許を取得して運営する広告/リスティングサービスとして建てる(免許は回避でなく取得側へ・研究正本エンティティ方針)。MVPは情報の構造化開示＋オフライン成約への送客だけ。資金保持・成約保証・title判定は一切自前でやらない(設計保護P1に限定、金銭保護P3は将来)。

### 5.1 Prakas 064（広告ライセンス）— MVPで何をして何を避けるか
最重要論点: 「物件の広告掲載行為がPrakas 064のトリガーか」=収益モデルの生死(G0-a第1問★・弁護士未確認)。確認に依存せず最安全側で設計。

| 項目 | 分類 | 内容 |
|---|---|---|
| 運営主体=真正クメール法人 | 将来(公開ブロッカー) | 一般公開前にクメール法人設立+Prakas 064免許取得が必須。それまで招待制クローズドβ(`noindex`・検索インデックス禁止)で公開リスティングを出さない |
| MVP開発・デモ段階 | MVPで対応 | 開発・投資家デモ・勧誘は「広告サービス提供」に当たらない設計(ダミー or 同意済みパイロット物件のみ)。現行HTMLデモは営業道具として安全 |
| リスティングの法的位置づけ | MVPで対応 | 「不動産仲介業者」でなく「オーナー/登録エージェントが自ら掲載する情報掲示+検索ツール(classifieds/intermediary)」。価格交渉・成約・金銭授受に関与しない |
| 全リスティング共通フッター免責 | MVPで対応 | ①情報掲示PFであり仲介・評価・法的助言をしない ②情報は投稿者の自己申告で正確性を保証しない ③契約・支払い・物件確認は利用者責任 ④title/メーター/税務は専門家に確認 |
| 免許取得前の収益化 | やらない | 掲載課金・featured・サブスクを取らない(対価収受がPrakas 064トリガーになる確率が最も高い)。無料βで供給を貯める |
| 自前の不動産評価・査定 | やらない | 査定額・「適正賃料」表示はしない(別免許リスク)。出すのは投稿者申告値と公的基準値(政府電気$0.15-0.20/水$0.18-0.26)の比較表示のみ |

MVPで避けること: (a)免許前の公開リスティング (b)掲載・送客への課金 (c)仲介・査定を名乗ること (d)資金保持 (e)titleの正否を当社が判定すること。

### 5.2 KYC（最小・出品者本人確認）
§4.4の通り「出品者本人確認」レベルに最小化。資金を扱わないため重いPSI型KYC不要。身分証は検証バッジ申請時のみ必須・暗号化非公開・OCR後破棄も検討。需要側はゲスト問い合わせ可。
- 将来: カンボジアのデータ保護法制に合わせた本格DPA・域内保存要件確認、検証バッジ申請時の第三者eKYC(決済PSI連携時に再利用)。
- やらない: 金融機関レベルのeKYC(顔認証・liveness)、借主(需要側)のKYC。

### 5.3 決済なし前提の「問い合わせ→オフライン成約」動線
決済/エスクローは入れない(PSI資本要件$2M・5%供託が重い)。接触を生むが金銭は一切通さない。Airbnbの教訓(場外流出leakage対策)を最小コストで取り込む。

**MVPで対応**:
1. 問い合わせフォーム(PF内): 物件詳細「問い合わせ」→ログイン/ゲスト名+連絡先→送信。**この時点でオーナー連絡先を出さない**(Airbnb型)。
2. 段階的開示: 送信後にオーナー/エージェントへ通知→双方応答でPF内スレッド/Telegram deeplink発行。問い合わせログを当社が保持(成約トラッキングの足場)。
3. gate条件が問い合わせ前に全部見える(価格最終・デポ・メーター・VAT・契約年数が確定表示)→オフライン成約が速い。
4. 成約は完全オフライン。当社は「成約しましたか?」フォローアップ通知だけ送り任意申告で回収(後の手数料モデルの布石)。
5. 免責: 「当社は契約当事者でなく、内見・契約・支払いの安全は利用者責任」を問い合わせ確定画面に明示。
- 将来: 連絡先開示を成約手数料/保護feeにフック(HousingAnywhere型・第三者PSI経由)、KHQR/Bakong連携でデポ・初月家賃エスクロー(P3)。
- やらない: 自前の資金保持・エスクロー・家賃集金、PF内決済UI。
- **leakageへの現実スタンス**: MVPでは流出を完全には防げない。防がず「成約後の保護/契約書生成」という場外で失う価値を将来レイヤーで作る(研究正本7f: P1設計保護→P3金銭保護)。

### 5.4 title/物件真正性の検証フラグ（Verify.gov.kh参照）
Verify.gov.kh=MLMUPC運営の国家デジタル土地権原検証(QR/アプリで Original/Fake 即判定)。当社は「事実開示」に留め保証(賠償)はしない(7f)。

| フラグ | MVPの意味 | 当社の責任範囲 |
|---|---|---|
| Title種別の自己申告 | Hard/Soft/Strata/Lease を宣言値として入力必須(gate条件) | 申告の転記のみ。正否保証なし |
| Verify.gov.kh 確認推奨リンク | 物件詳細に「公的検証はVerify.gov.khで」と外部リンク+手順 | 利用者が自分で検証する導線提供のみ |
| 検証ステータス(self-declared/pending/—) | バッジは「当社判定」でなく「オーナーが検証書類を提出済み」の事実フラグ | 事実開示。Original/Fakeを断定しない |
| 外国人所有の構造警告 | コンド=ストラタ70%上限・地上階上のみ・国境30km等を自動注記(研究§6) | 一般情報の提示(法的助言でない旨明記) |

- 将来: Verify.gov.khとのAPI/公式連携(MLMUPC確認が前提)、検証済みtitleの「verifiedバッジ」昇格(書類審査+免責設計を弁護士確認後)。
- やらない: 当社がtitleの真贋を判定・断定・保証すること(P3領域)、title書類の代行取得・登記facilitation。

### 5.5 利用規約・プライバシーポリシー（3言語・最低限）
クメール法人がdata controller。

**ToS最低限**: サービスの性質(情報掲示PF・仲介/評価/助言/当事者でない)/掲載情報の責任(投稿者が保証・当社無保証)/禁止事項(虚偽・negotiable等の曖昧表記=gate違反・スパム・無断掲載・なりすまし)/投稿者の表明保証/免責・責任制限(取引上の損害について免責・賠償上限)/連絡先取扱い/準拠法・管轄(カンボジア法・プノンペン)/規約変更・バージョン管理。

**PP最低限**: 管理者(運営クメール法人)/収集情報(氏名/連絡先/メール/任意で身分証・事業者番号/問い合わせ履歴/Cookie・ログ)/利用目的/第三者提供(問い合わせ相手への連絡先開示・将来の決済PSIへの提供は同意ベース)/保存・セキュリティ(身分証=暗号化非公開・PIIとリスティング分離)/利用者の権利(アクセス・訂正・削除・登録解除)/Cookie・解析(Consent banner)/国際移転(Supabase/Vercelの域外保存明示)。

- MVPで対応: 上記全条項のドラフト掲載+登録時同意取得+バージョン記録、3言語、Consent banner。
- やらない: GDPR完全準拠の重厚なDPO設置(過剰。ただしConsent bannerは入れる)。

### 5.6 規制対応サマリー（MVPで対応 / 将来 / やらない）
| 領域 | MVPで対応 | 将来 | やらない |
|---|---|---|---|
| Prakas 064 | クローズドβ(noindex)・免責表示・掲示板位置づけ・無料 | 公開前にクメール法人+免許・課金開始 | 免許前の公開・掲載課金・仲介/査定を名乗る |
| KYC | 最小出品者確認・身分証は暗号化任意 | 検証バッジ用eKYC・域内保存対応 | 金融eKYC・需要側KYC |
| 決済/成約 | PF内問い合わせ→段階的連絡先開示→オフライン成約・成約フォロー | 連絡先開示への手数料・KHQR/Bakongエスクロー(PSI提携) | 自前資金保持・エスクロー・決済UI |
| title検証 | 自己申告必須・Verify.gov.kh導線・self-declaredフラグ・外資警告 | Verify.gov.kh公式連携・verifiedバッジ昇格 | 当社による真贋判定/保証・書類代行 |
| データ保護/規約 | 3言語ToS/PP・同意記録・PII分離・Consent banner | カンボジア法制準拠監査・DPA | 過剰なGDPR重装備 |

---

## 6. スコープ & 段階計画

固定前提: 順序は供給(エージェント登録→gate投稿→問い合わせ応答)を需要UIより先。P1は決済ライセンス不要で立つ(顧客資金を自社BSに1秒も乗せない)。MVP v1は決済・エスクロー・接触課金徴収を全て外す＝法務不確実性下でも供給資産(gate充足物件DB)だけは積めるヘッジ。

### 6.1 MVP v1 スコープ表
判定軸: ①供給サイドに必須か ②法務リスク(課金=Prakas 064)を生まないか ③AI 1名が2-4週で組めるか。

**出す機能(MVP v1 / 4週)**:
| # | 機能 | 実装メモ |
|---|---|---|
| S1 | エージェント/オーナー登録+KYC最小 | Supabase Auth(email/OTP)。身分証アップは手動承認(自動審査しない) |
| S2 | 物件投稿フォーム(gate必須スキーマ) | 公開最低要件4項目=写真・賃料最終値(negotiable不可)・面積・外国人契約可否。上位表示誘導=VAT可否/メーター帰属/更新据置/源泉負担。2軸+物件タイプ10種 |
| S3 | 業種別モジュール3種(F&B/小売/オフィス) | gate §B選択式。残り業種はP2 |
| S4 | 物件一覧+詳細ページ | 一覧=フィルタ+ソート。詳細=gate値構造化表示+gate充足バッジ(事実開示) |
| S5 | 地図表示+ピン(用途別色) | **指なぞり囲い検索はv1では四角/円のboundsクエリに簡略化**(turf自由図形はv2) |
| S6 | 問い合わせ→応答(PF内スレッド) | Supabase Realtime or polling。連絡先伏せ。leakage計測データ取得が目的 |
| S7 | 3言語i18n(EN/JA/KH) | next-intl。UI文字列のみ。物件本文の自動翻訳はv2 |
| S8 | gate充足バッジ(事実開示・無賠償) | gate値の有無を機械判定。規約で「保証でない」明示 |
| S9 | 借主/オーナー gate診断ツール | gate質問→危険スコア+確認漏れ→「条件満たす物件を見る」導線。SNSシェアでcold-startリード獲得(バイラルエンジン) |
| S10 | PWA shell | manifest+SW。インストール可・オフラインshellのみ |
| S11 | 管理画面(投稿承認・KYC承認) | 白石/運営が目視承認してから公開(初期品質担保) |

**出さない機能(明示的に切る)**:
| # | 機能 | いつ | 理由 |
|---|---|---|---|
| X1 | 決済・接触課金の徴収 | G0-a回答後 | 課金=Prakas 064 "advertisement activities"罰金リスク。無料で供給を積む |
| X2 | エスクロー/デポ保護/家賃集金 | P3(PSI提携後) | PSIライセンス($0.5-2M資本) |
| X3 | Verify.gov.kh title自動連携 | P2 | 公開API未確認。v1は売買主体でなくtitle検証不要 |
| X4 | 本人確認の自動審査・本格KYC | P2 | v1は手動目視承認で足りる |
| X5 | AI gate自動抽出 | P2 | 書いてない値は抽出不能。v1は手入力で確定値を取る方が速い |
| X6 | バイリンガル契約書自動生成 | P1後半(leakage防御主力) | v1構築工数を圧迫 |
| X7 | 指なぞり自由図形検索(turf) | v2 | bounds(矩形/円)でv1探索体験は成立 |
| X8 | 中国語・微信対応 | P3 or 非対応判断 | v1はJA→EN→KHに集中 |
| X9 | エージェントtool(CRM/サブスク) | P2 | 課金機能=X1と同じ法務待ち |
| X10 | レコメンド/マッチングAI | P2以降 | コピー可能・堀でない |
| X11 | ネイティブアプリ | 後期 | PWAで足りる |

### 6.2 週割りマイルストーン（W1-W4 / AI実装1名）
各週末に動くものをVercelにデプロイし白石が触れる状態に(インクリメンタル)。

| 週 | テーマ | 作るもの | 週末の検証可能状態 |
|---|---|---|---|
| W1 | 基盤+データモデル | Next.js初期化(App Router/TS/Tailwind)・Supabaseスキーマ(profiles, agents, listings, listing_translations, inquiries, documents)+RLS・next-intl 3言語スキャフォールド・Auth(OTP)+登録(S1)・Vercelデプロイ+ドメイン仮設定 | ログイン→空ダッシュボードが3言語表示。DBにテーブル存在 |
| W2 | 供給の心臓=投稿 | gate必須スキーマ投稿フォーム(S2)・業種別モジュール3種(S3)・Storage画像アップ・管理画面の投稿承認(S11)・KYC手動承認(S1) | エージェントが物件をgate必須で投稿→運営承認→DBに乗る。**供給サイドの根幹が動く** |
| W3 | 需要側の最小UI | 物件一覧+フィルタ+ソート(S4)・物件詳細+gate充足バッジ(S4/S8)・地図+ピン+bounds検索(S5)・問い合わせスレッド(S6) | 投稿物件が一覧/地図/詳細で見え、問い合わせスレッド開通。**供給→需要の一気通貫が成立** |
| W4 | 仕上げ+獲得装置+PWA | gate診断ツール(S9)・PWA shell(S10)・3言語の文言精査(KHネイティブレビュー依頼)・gate充足バッジ/利用規約の無賠償文言・seed投入(BFC実物件+手動キュレ20-30件)・デモ録画 | 全機能がスマホでインストール可・3言語・実物件入り。**営業デモ+βテスト可能** |

調整弁: 2週版ならW1+W2圧縮しW3地図/W4 PWA・診断を削る(供給投稿+一覧+問い合わせだけ)。**4週推奨**(供給3要件+獲得装置まで入る)。

### 6.3 インフラ/API 月額コスト
全て無料枠で開始可能。MVP〜βは実質**$0-2/月(ドメインのみ)**。

| サービス | 無料枠 | 有料移行ライン | 課金後目安 |
|---|---|---|---|
| Supabase | Free: DB 500MB/Storage 1GB/月5万MAU | Pro=$25/月(DB 8GB/Storage 100GB/日次バックアップ) | $0→$25 |
| Vercel | Hobby: 100GB帯域(商用は要Pro) | 商用Pro必須=$20/月/席 | $0→$20 |
| 地図API | 選択肢で大差(下記) | — | $0〜25 |
| ドメイン | — | .com=年$12-15 / bellfoods.com.khサブドメインなら$0 | 約$1-2/月 |
| メール(OTP/通知) | Supabase内蔵 or Resend Free(3千通/月) | Resend $20/月(5万通) | $0 |
| **合計** | — | — | **MVP: $0-2/月 → β: $45-90/月 → P1スケール: $150-300/月** |

地図API選択(コスト最重要分岐・§3.5の対立点):
| 候補 | コスト | 適合 |
|---|---|---|
| MapLibre+MapTiler/Protomaps+OSM | 無料枠大・OSSレンダラ・Leaflet資産と親和 | コスト最小・カンボジアOSM実用十分(事業計画側の推奨) |
| Mapbox | 月5万ロード無料→従量 | turf囲い検索移植が軽い・大量ピン軽量(テックリード側の推奨) |
| Google Maps | $200クレジット→従量(高単価) | 精度最高だが課金リスク大・MVPでは避ける |

判断: **MVP v1 = MapLibre+MapTiler/OSM($0)で開始 → 要件次第でMapboxへ差替**。Google Maps化は営業デモで必要なら差替(プロト同方針)。

### 6.4 成功指標KPI（初期目標値・仮置き）
北極星=供給の厚みと問い合わせ発生。leakage率はP1後半まで測らない。

| 区分 | KPI | W4(β開始) | P1(3ヶ月) |
|---|---|---|---|
| 供給 | 登録エージェント/オーナー数 | 10-15 | **40-60** |
| 供給 | gate充足の掲載物件数 | 20-30(seed込) | **150-250**(事業用≥40%) |
| 供給品質 | gate上位要件(VAT/メーター/更新)充足率 | — | ≥50%が3項目以上 |
| 需要 | 月間ユニークビジター | — | 500-1,500 |
| 需要 | 問い合わせ数(スレッド開通) | 5-10 | **50-100/月** |
| 獲得装置 | gate診断ツール完了数 | 30+ | 300+ |
| 転換(参考) | 問い合わせ→内見/成約報告(自己申告) | — | 把握開始 |
| wedge検証 | WTPヒアリング完了(P-1層) | — | **10-20名**(財務を組む前提・最重要) |

判定: P1終了時に**掲載150件 AND 問い合わせ50/月 AND 登録40**の3つが揃えば供給サイドPMF初期シグナル→課金導入(P2)へ。

### 6.5 鶏卵問題（供給ゼロ）の初期ブートストラップ
エージェント依存をやめ、BFCローカル関係資本を初期供給の本体に置く。順序=自社→手動キュレ→人脈→エージェント従属化。

| # | 策 | 内容 | 見込み件数 |
|---|---|---|---|
| B1 | BFC自社物件をseed | Chamkarmon E0移転(2026-06)のworked example。新旧2物件をgate完全充足のショーケースに | 1-2件(見本) |
| B2 | 手動キュレ(運営が代理投稿) | FB Groups/Khmer24の優良物件をオーナー許可取りgate化(supply-side concierge) | 20-30件 |
| B3 | 日本人会/F&B網/商工会に診断ツール配布 | wedge層の横connで「自分の物件診断して」→オーナー自発投稿 | 10-20件(自発) |
| B4 | wedge餌でエージェント従属化 | 「gate充足物件には支払い能力高い外国人/法人の本気借主が来る」=良質リードへのアクセスを餌に | P1後半 10-20件 |
| B5 | 片面集中の順序設計 | 需要側マーケ(診断ツール拡散)を供給20件超えてから開始。空の箱に客を呼ばない | — |

cold-startの肝: B1+B2で「最初から30件gate充足物件が見える状態」を作ってからβ公開(W4のseed投入)。エージェントを供給計画の前提に置かない。

### 6.6 AI実装の工数（AI 1名・白石レビューのみ）
| フェーズ | 私の実働 | 白石の実働 |
|---|---|---|
| MVP v1構築(W1-W4) | 実働15-25時間相当(4週カレンダー) | レビュー+判断のみ(各週末デモ確認・KHネイティブ手配) |
| seed投入 | 5-8時間(手動キュレ含む) | オーナー許可取り(対外) |
| 法務並行(G0-a) | 0(AI不可) | AK手動(弁護士friend一次見解) |
| WTPヒアリング | 2-3時間(設計) | AK手動(10-20名は対外) |

難所=Supabase RLS設計・i18nの物件本文扱い・地図bounds検索。**律速はコードでなく白石の対外3行為(法務一次見解・WTPヒアリング・オーナー許可取り)**。法務(G0-a)が「掲載=広告でアウト」なら無料供給積みは続行可だが課金=P2が全面ストップ→Plan B。

### 6.7 撤退ライン / 見直しライン（Go/Pivot/Kill）
| ゲート | 時点 | Go | Pivot | Kill |
|---|---|---|---|---|
| **G-法務(親ゲート)** | W2-W4(G0-a回答) | 掲載=広告がPrakas 064非該当 or RPRライセンス外国法人取得可 | 該当だがB-1(自社RPR取得)/B-2(JV)で課金可 | 該当+ライセンス外国法人発給不可+JVも又貸しリスク=**marketplace本体Kill→B-3 SaaS縮退**(gate/検証toolをエージェントにSaaS販売) |
| G-供給 | P1(3ヶ月) | 掲載150+登録40+問い合わせ50/月 | 1-2指標未達だが伸び率良好→seed策強化し延長 | 3指標とも未達+横ばい→縮退 |
| G-WTP | P1並行 | wedge10-20名が検証バッジ/サブスクに具体額提示 | 額が想定下限割れ→収益モデル組み直し | 「金は払わない・FBで十分」支配的→ピボット(B2C相場ツール等)or Kill |
| G-leakage | P2(防御実装後) | leakage率≤30% | 30-50%→接触課金ゲート/契約生成を強化 | >50%が防御後も継続→「FBの劣化版」確定→Kill |
| 撤退コスト | 全期間 | — | — | SPC清算+預り金返還+退職金=**$20-50k**を常時バッファ確保(割ったら無条件縮退) |

最重要: **G-法務(G0-a)が全ての親ゲート**。黒なら無料供給積みまでは作るが課金は封印しPlan B(SaaS縮退)へ即移行。

---

## 7. 白石の判断が要る未決事項

### A. 法務・エンティティ（最優先・親ゲート）
- **G0-a★(弁護士一次確認・未実施)**: 物件の「広告掲載」行為がPrakas 064免許のトリガーか=収益モデルの生死。クローズドβ(無料・noindex・ダミー/同意済みパイロット物件のみ)なら免許前に運用してよいか、掲載自体が即トリガーで法人設立まで一切公開不可か。**この回答がβ運用範囲・課金可否(P2)・全撤退ゲートの親**。AK着手判断要。
- クローズドβの境界線が「招待制・ダミー/同意済み物件のみ・noindex・無料」ならPrakas 064非該当という安全側仮説を弁護士に当てる必要(確度未検証)。
- 本番repoのGitHub org/名義: Kangi-PLC か真正クメール法人の新org か。Prakas 064(運営主体=真正クメール法人)確定によりホスティング/ドメイン/repo名義を法人に寄せる必要があるが法人設立がG0法務確認待ち。
- ToS/PPの準拠法・管轄をカンボジア法・プノンペン管轄とする方針でよいか(クメール法人運営に整合・弁護士確定版で確認)。

### B. ドメイン・インフラ
- ドメイン確定: bellfoods.com.kh サブパス vs Kangi独自ドメイン vs 不動産専用新ドメイン。運営法人が決まらないと契約名義が決められない(新設SPC方針との整合)。
- Supabase有料プラン移行ライン: PostGIS有効化・Storage(KYC画像)・preview用別DBで無料枠を超える時期。本番トラフィック前提が固まり次第コスト試算が必要。

### C. 地図API（セクション間で推奨が対立 — 1本化判断）
- **Mapbox GL JS(テックリード推奨・確度80%・移植軽い/大量ピン軽量) vs MapLibre+MapTiler/OSM(事業計画推奨・$0)**。本書は「MapLibre+MapTilerで$0開始→要件次第でMapboxへ差替」で両立させたが、最終1本化は白石判断。プノンペンのPOI/店舗名表示をどれだけ重視するか=将来「近隣の店/施設」表示を主要機能にするならGoogle寄せの再検討余地。住所補完にGoogle Places併用=Google課金が一部発生する点の承認も要。

### D. データモデル運用ポリシー
- **必須gateの集合をどこまで強制するか**: データモデル(§2)は全gateをpublish必須にできる設計だが、事業計画(§6)はW2で公開最低要件4項目に絞る。「強制する必須gate集合」を起動時パラメータで絞れる設計にしたが、供給が積まれるに従ってどの順で必須範囲を広げるかの閾値設計が未決。
- 業種別モジュールの必須キー集合をコードにハードコードするか`module_required_keys`マスタで運用可変にするか(後者を仮採用。誰が業種要件を編集するか未定)。
- `title_deed_type`を sale だけでなく rent でも必須にするか(sale では必須化が望ましいが rent では任意でよいか)。
- `expires_at`(鮮度管理)を何日に設定するか(30/60/90日? カンボジア市場の物件回転速度の実データ待ち)。
- 中国語(zh)を`listing_translations`の locale に最初から許可するか、EN/JA/KH確定後にAI翻訳で追加する運用フラグだけ用意するか。
- decimal精度: USD単一か、リエル建て(KHR)併記の`currency`列を持つか(Bakong/KHQR将来連携を見据え)。

### E. 供給・運営フロー
- エージェントKYCの`verified`判定を誰が行うか(admin手動 vs Verify.gov.kh/事業者登録APIの自動照合)。MVP初期はadmin手動を仮置きしたが運用負荷の見積り未了。
- 公開審査(in_review→live)を全件人手にするか、AI一次判定+グレーのみ人手にするか(MVP初期の物件流入量と運営工数の想定値が未確定)。
- KYCで仲介会社のライセンス番号を必須にするか任意にするか(Prakas064上どこまで掲載主体に免許確認義務が及ぶかはG0-a弁護士確認の回答待ち=KYCフォームの必須/任意が変わる)。
- 成約 rented マークはエージェント自己申告に依存。虚偽放置(成約済を載せ続け)を罰する仕組み(反響スコア減衰・期限切れ自動paused)の閾値設計が未決。
- 身分証画像の保存可否・暗号化要件・保存期間がカンボジアのデータ保護法制でどう規定されるか(OCR後破棄方針でリスク最小化したが法令確認待ち)。Supabase/Vercelの域外保存が将来の域内保存要件(data localization)に抵触しないか。

### F. leakage・検証
- エージェントのTelegram外部誘導(leakage)をどこまでUIで抑止するか。完全ブロックは登録忌避を招くため初回はソフト(スコア非加点+注意フラグ)で十分か。
- Verify.gov.kh のtitle照合を投稿フローに組み込むタイミング(MVPは手動フラグのみ、自動連携は次フェーズで良いか)。Verify.gov.kh(MLMUPC)が外部PFからのAPI/公式リンク利用を許容するか公式確認が必要。

### G. スコープ・スケジュール
- KPI目標値(掲載150件/登録40/問い合わせ50月)は仮置き。WTPヒアリング(P-1層10-20名)の結果で上書きする前提だが、ヒアリング着手時期=白石の対外スケジュール待ち。
- MVP v1を2週版(供給+一覧+問い合わせのみ・投資家デモ優先)にするか4週版(診断ツール/PWA/地図までβテスト品質)にするか。推奨は4週だが松崎/投資家デモの時期次第。
- βテスト最初の供給オーナー(B2手動キュレ/B3人脈)へのアプローチ=対外行為のため白石着手が必要(日本人会/F&B網/商工会の優先順、誰から声をかけるか)。
- KYC自動化のタイミング(初期は人手審査で確定したが、エージェント登録数増加時に連携する自動KYCプロバイダ(カンボジア対応)の選定は将来課題)。

---

## 付録A. 確定した設計判断（全42件）
- gate強制はアプリ層でなくDB層: listing_status ENUM の published 遷移時にトリガー fn_listing_gate_complete が全Universal gate+業種モジュール必須項目+agent KYC verified を検査しNULL/未充足なら RAISE EXCEPTION。直API/管理画面もバイパス不可＝design-protection の最深層
- Universal必須gate(賃料/デポ/契約年数/専用メーター/VATインボイス可否/面積/入居日 等)は物理カラムで保持(JSONBに逃がさない)。理由=published時のNOT NULL/CHECK制約は物理列にしか刺さらない。可変な業種別モジュールだけ JSONB(listing_business_modules)に分離
- price_negotiable カラムをスキーマに存在させない。rent_usd_month は published時 NOT NULL かつ >0。白石の『negotiable禁止＝最終条件が最初から出る』をデータ型レベルで物理的に実現
- 多言語は別テーブル listing_translations を採用(JSONBでなく)。per-locale で translation_source(human/ai)・is_reviewed を行で持てネイティブレビュー必須要件に直結、中国語AI後付け・部分翻訳・locale別FTSが自然。構造化gate値(数値/bool/enum)は言語非依存なので listings 本体に1回だけ持ち翻訳不要
- 地図検索は PostGIS geography + GiST を採用。ST_Within でプロトの turf.js 指なぞり囲い検索をサーバ側で正確再現、bbox一次絞り→ポリゴン精査の2段。search_listings_in_polygon RPC でフロント資産をそのまま移植
- Storageは2バケット分離: listing-media=public read(公開写真)、secure-docs=private(title deed/身分証/契約書/税書類)で署名URL経由のみ。匿名のtitle deed閲覧を物理的に不可に
- 検証バッジ/title検証は『事実開示』であって賠償保証でないとDB上も分離(白石7f原則)。verification_badges は gate値/title_verifications から導出、金銭保証(エスクロー)はPSI免許が必要なため今回スキーマに入れない(将来 escrow_holds で追加)
- leakage(場外取引)対策: エージェント実連絡先は agents に持つが v_public_listings View からは除外、問い合わせ成立後のプラットフォーム内 messages で会話、contact_shared フラグで場外共有を監視
- KYC強制を供給サイドの入口に: agents.kyc_status='verified' でないと物件を published にできない＝headhunter co-opt(エージェント登録)が在庫公開の必須条件になる構造
- 監査テーブル audit_log で全 status遷移/KYC承認/title検証/publish拒否を記録。Prakas 064 監督下の説明責任 + design-protection 証跡
- 本番リポジトリは新規 kh-property を作成し、プロト kh-property-proto は営業道具として温存(本番化しない確定方針どおり)
- route group で (public)/(agent)/(auth)/(admin) を分離。エージェント供給サイド(物件投稿gateフォーム)を最優先実装
- 画面内の変更は全て Server Actions、外部システムが叩く口(Webhook/OGP/cron/将来public API)だけ Route Handler に限定してAPIサーフェスを最小化
- ロールは app_metadata.role(JWTクレーム)に保持しuser_metadataは使わない。防御の本丸は全テーブルRLS、layoutガードはUX用の二層構成
- 地図API=Mapbox GL JS を推奨(確度80%)。理由=turf囲い検索がWebGL+GeoJSONでほぼ無改造移植・大量ピンが軽い・コスト3割安。住所補完だけGoogle Places APIを部分併用するハイブリッド
- turfの booleanPointInPolygon ロジックは地図ライブラリ非依存なので1行も変えず温存。データ増大に備えクライアントturf(即時プレビュー)+PostGIS ST_Within(本番絞り込み)の二段構え。複数選択SetフィルタはURL params+nuqsに載せ替え
- i18n=next-intl v4。locales=[en,ja,km]・defaultLocale=en・localePrefix=as-needed。proxy.ts で next-intl と Supabaseセッションrefreshを統合
- AI翻訳パイプライン=en.json正本→Claude haikuで欠損キーのみ埋めるマージ方式。クメール語はネイティブレビュー必須(reviewed:falseフラグ運用)。中国語はlocale追加だけで拡張可能な設計
- PWA=app/manifest.ts(native)+Serwist(next-pwa後継)。ネイティブアプリは後期の確定方針どおり、まずホーム画面追加体験まで
- gate条件必須スキーマをDB NOT NULL制約とzod requiredの二重で強制し、'negotiable禁止・設計保護'をコードレベルで担保
- 公開ボタンは全gate充足までDB制約+UI二重ガードでdisabled。'交渉可/negotiable'は入力欄自体を存在させない=構造上入力不能にして情報アービトラージを消す(design-protection)
- KYC審査(人・1回)と公開審査(物件1件ごと・開示完全性)を別ゲートに分離。KYC審査中も物件下書きは可にし離脱を防止、公開のみKYC承認後
- エージェントの自由文は物件詳細を3層分離した最下層『出品者コメント枠』にのみ隔離。gate値と矛盾する記述はAIが公開前に検知・ブロック。価格再交渉はチャット不可・価格フィールド編集のみ(全員同一最終価格を維持)
- 投稿フォームはPLAYBOOK §1-§5を5ステップウィザードに変換。プロトの2軸(取引×用途)+物件タイプ10種を踏襲し移植コストを最小化。業種タグ選択で§2b業種別必須項目を動的追加
- 成約時の rented マークで信頼バッジが上がる在庫鮮度ループを実装し、Khmer24の『死んだ広告』問題を構造で解消
- 需要側はMVPで検索/詳細/問い合わせの3画面のみ。gate値フィルタ(VAT可/専用メーター/外国人可/契約年数)を需要側に出すことで供給側へ全項目入力の圧を生む。決済/エスクロー/ユーザーDBはスコープ外(確定方針通り)
- オンボーディングはheadhunter co-opt=エージェントを排除でなく『集客が増える道具』として口説く。掲載無料(P1設計保護・liability無)を参入摩擦ゼロの訴求にする
- 免許は回避でなく取得側に回す（研究正本§エンティティ方針準拠）。公開リスティングを出す前=真正クメール法人設立＋Prakas 064免許取得を公開ブロッカーに設定。それまでは招待制クローズドβ(noindex・ダミー/同意済みパイロット物件のみ)で供給サイド(エージェント在庫)を貯める
- 免許取得前は掲載課金・featured・サブスクを一切取らない。『広告サービスの対価収受』がPrakas 064トリガーになる確率が最も高いため、無料β期間に限定し収益化は免許後
- プラットフォームを『不動産仲介業者/評価業者』ではなく『投稿者が自ら掲載する情報掲示＋検索ツール(classifieds/intermediary)』と位置づけ、価格交渉・成約・金銭授受に一切関与しない。これでPrakas/決済/評価の3つの免許リスクを同時に下げる
- KYCは金融eKYCではなく『出品者本人確認』レベルに最小化。資金を扱わないため重いPSI型KYCは不要。身分証は検証バッジ申請時のみ必須＝暗号化非公開・表示しない・OCR後破棄も検討。需要側は摩擦低減のためゲスト問い合わせ可(KYC不要)
- 決済は入れず、Airbnb型の段階的連絡先開示(問い合わせ前はオーナー連絡先非開示)で問い合わせログを当社が保持→将来の成約手数料の足場にする。成約自体は完全オフライン
- title/メーター/税務は『当社が判定・保証』せず『投稿者の自己申告＋Verify.gov.kh外部検証導線＋self-declaredフラグ』に限定。研究正本7f『バッジ=事実開示であって賠償保証でない(設計保護P1)』を全フラグに徹底適用し、賠償責任を負わない
- ToS/PP/Consent bannerを3言語(EN/JA/Khmer)で最初から。管理者=クメール法人。PIIとリスティングを分離テーブルで保持し収集最小化
- MVP v1から決済・接触課金徴収・エスクローを全て外す＝Prakas 064(掲載=広告)の法務リスクを回避したまま供給資産(gate充足物件DB)だけを無料で積むヘッジ設計にする
- 供給サイド3要件(エージェント登録→gate必須投稿→問い合わせ応答)をW1-W4の心臓に据え、需要側UIはW3まで後回し。各週末に動くものをVercelデプロイしてインクリメンタル検証
- 地図APIはMapLibre+MapTiler/OSM(無料)を採用。Google Maps Platformは課金リスク大でMVPでは避け、営業デモで必要なら差替(プロト同方針)
- 指なぞり自由図形検索(turf)はv2送り、MVP v1はbounds(矩形/円)検索に簡略化して工数圧縮
- 初期供給はエージェント依存をやめBFCローカル関係資本(自社物件seed+運営の手動キュレ20-30件+人脈)を本体にし、エージェントはP1後半に従属チャネル化
- インフラは無料枠開始でMVP実質$0-2/月(ドメインのみ)、β公開で$45-90/月、P1スケールで$150-300/月
- AI実装1名で4週カレンダー内に構築完了可能。律速はコードでなく白石の対外3行為(法務一次見解・WTPヒアリング・オーナー許可取り)
- G-法務(G0-a)を全ゲートの親に設定。黒なら課金封印しB-3 SaaS縮退へ即移行

## 付録B. 白石の判断が要る未決事項（全27件）
- [ ] 業種別モジュールの必須キー集合をコードにハードコードするか module_required_keys マスタテーブルで運用可変にするか。後者を仮採用したが運用者(誰が業種要件を編集するか)が未定
- [ ] title_deed_type を sale だけでなく rent でも必須にするか。外国人の地上階所有制限を踏まえると sale では必須化が望ましいが、rent では任意でよいか白石判断要
- [ ] listing の expires_at(鮮度管理)を何日に設定するか。Zillow式で classifieds に勝つ要素だが、カンボジア市場の物件回転速度に合わせた日数(30/60/90日?)は実データ待ち
- [ ] 中国語(zh)を listing_translations の locale に最初から許可するか、i18n基盤(EN/JA/KH)確定後にAI翻訳で追加する運用フラグだけ用意するか。スキーマは locale text で任意言語可だが運用ポリシー未定
- [ ] エージェントKYCの verified 判定を誰が行うか(admin手動 vs Verify.gov.kh/事業者登録APIの自動照合)。MVP初期は admin手動を仮置きしたが運用負荷の見積り未了
- [ ] decimal精度: rent_usd_month を numeric(12,2)・単価を numeric(6,4) としたが、リエル建て併記(KHR)を持つか(USD単一か)未確定。Bakong/KHQR将来連携を見据えると currency 列の追加余地あり
- [ ] 本番リポジトリの GitHub org/名義: Kangi-PLC か、真正クメール法人の新orgか。Prakas 064(運営主体=真正クメール法人)の確定によりホスティング/ドメイン/repo名義を法人に寄せる必要があるが、法人設立はG0法務確認待ち=白石判断
- [ ] ドメイン確定: bellfoods.com.kh サブパス vs Kangi独自ドメイン vs 不動産専用新ドメイン。運営法人が決まらないと契約名義が決められない
- [ ] Mapbox vs Google の最終確定(現状Mapbox推奨・確度80%)。Phnom PenhのPOI/店舗名表示をどれだけ重視するか=もし将来『近隣の店/施設』表示を主要機能にするならGoogle寄せの再検討余地。住所補完にGoogle Places併用=Google課金が一部発生する点の承認
- [ ] KYC自動化のタイミング: 初期は人手審査(adminが身分書画像確認)で確定したが、エージェント登録数が増えた時に連携する自動KYCプロバイダ(カンボジア対応)の選定は将来課題
- [ ] Supabase の有料プラン移行ライン: PostGIS有効化・Storage(KYC画像)・preview用別DBで無料枠を超える時期。本番トラフィック前提が固まり次第コスト試算が必要
- [ ] 公開審査(in_review→live)を全件人手にするか、AI一次判定+グレーのみ人手にするか。MVP初期の物件流入量と運営工数の想定値が未確定
- [ ] KYCで仲介会社のライセンス番号を必須にするか任意にするか。Prakas064上どこまで掲載主体に免許確認義務が及ぶかはG0-a弁護士確認(research §8)の回答待ち=その結論でKYCフォームの必須/任意が変わる
- [ ] エージェントのTelegram外部誘導(leakage)をどこまでUIで抑止するか。完全ブロックは登録忌避を招くため、初回はソフト(スコア非加点+注意フラグ)で十分か要判断
- [ ] Verify.gov.kh のtitle照合を投稿フローに組み込むタイミング。MVPは手動フラグ(未/照合済)のみで、自動連携は次フェーズで良いか確認したい
- [ ] 成約 rented マークはエージェント自己申告に依存する。虚偽放置(成約済を載せ続け)を罰する仕組み(反響スコア減衰・期限切れ自動paused)の閾値設計が未決
- [ ] G0-a★(弁護士一次確認・未実施): 物件の『広告掲載』行為がPrakas 064免許のトリガーか=収益モデルの生死。クローズドβ(無料・noindex)なら免許前に運用してよいか、それとも掲載自体が即トリガーで法人設立まで一切公開不可か。この回答でβ運用範囲が決まる
- [ ] クローズドβの境界線が未確定: 『招待制・ダミー/同意済みパイロット物件のみ・検索インデックス禁止・無料』ならPrakas 064非該当という当方の安全側仮説を弁護士に当てる必要がある（確度未検証）
- [ ] Verify.gov.kh(MLMUPC)が外部プラットフォームからのAPI/公式リンク利用を許容するか、利用規約・連携可否の公式確認が必要（MVPは外部リンク導線のみで非依存にしてあるが将来連携の前提）
- [ ] 身分証画像の保存可否・暗号化要件・保存期間がカンボジアのデータ保護法制でどう規定されるか（OCR後破棄方針でリスク最小化したが法令確認待ち）
- [ ] Supabase/Vercelの域外データ保存がカンボジアの将来の域内保存要件(data localization)に抵触しないか。MVPは域外でよいが法人運営フェーズで要確認
- [ ] ToS/PPの準拠法・管轄をカンボジア法・プノンペン管轄とする方針でよいか（クメール法人運営に整合させる前提）＝弁護士確定版で確認
- [ ] KPI目標値(掲載150件/登録40/問い合わせ50月)は仮置き。WTPヒアリング(P-1層10-20名)の結果で上書きする前提だが、ヒアリング着手時期=白石の対外スケジュール待ち
- [ ] G0-a(弁護士friendへの一次見解=掲載課金がPrakas 064トリガーか)が未回答。MVP構築は法務回答を待たず先行できるが、課金導入(P2)の可否はこの回答が親ゲート。白石の着手判断要
- [ ] ドメインをbellfoods.com.khサブドメインにするかKangi独自ドメイン新規取得するか(エンティティ=新設SPC方針との整合)。白石判断
- [ ] MVP v1を2週版(供給+一覧+問い合わせのみ)で投資家デモ優先にするか、4週版(診断ツール/PWA/地図まで)でβテスト品質にするか。推奨は4週だが松崎/投資家デモの時期次第
- [ ] βテスト最初の供給オーナー(B2手動キュレ/B3人脈)へのアプローチ=対外行為のため白石着手が必要。誰から声をかけるか(日本人会/F&B網/商工会の優先順)
