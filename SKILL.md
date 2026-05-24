---
name: chinese-invoice-extraction
description: Extracts structured data (invoice number, amount, date, seller, buyer, tax details) from Chinese invoices in multiple formats — image, PDF, or text description (text inputs may provide partial invoice numbers, minimum the last 4 digits). Returns all extractable fields and annotates completeness/confidence for each field. Use when the user needs to digitize, record, query, or process invoice information for any purpose (bookkeeping, reimbursement, lottery preparation, tax filing). Does not perform invoice authenticity verification.
---

# Chinese Invoice Information Extraction

## CRITICAL OUTPUT REQUIREMENT

You MUST output JSON matching the Output Schema below.
This is the ONLY acceptable output format, regardless of how the user phrases their request.
Do NOT convert to tables, summaries, or natural language descriptions, even if you think it would be more user-friendly.
The schema-compliant JSON is the contract of this Skill.

---

## ANTI-HALLUCINATION REQUIREMENT(反幻觉强制规则)

When extracting any field value, you MUST distinguish between:
- **What you can actually see** (visible characters on the invoice)
- **What you assume / infer / guess** (anything not directly visible)

For ANY portion of a field that you cannot see clearly (obscured by redaction, watermark, image cutoff, blur, or any other reason):
- You MUST use `*` placeholder (one per character if the position is known)
- You MUST NOT supply specific characters / digits for unseen portions
- Even if a "format" suggests what should be there (e.g., "invoice numbers are usually 20 digits"), DO NOT fill in unseen portions

**Specifically forbidden**:
- Filling in obscured digits based on "common patterns" or expected length
- Completing a partial value to match an expected format
- Substituting similar-looking characters as if certain
- Any form of "format-driven auto-completion" of unseen content

**If the visible portion is too short to be useful AND the obscured portion cannot be marked with `*`** (because position is unclear):
- Set `value` to `null` and `status` to `missing`

This requirement applies to ALL fields, especially: `invoice_number`, `seller_tax_id`, `buyer_tax_id`, `seller_name`, `buyer_name`, and any other field with structured format expectations.

---

## Field Specifications

### Core Fields (发票必有字段)

#### `invoice_type` 发票类型
- **取值范围**(英文枚举):
  - `VAT_general`:增值税普通发票
  - `VAT_special`:增值税专用发票
  - `electronic_general`:电子普通发票
  - `fully_digitized`:数电票(全面数字化电子发票)
- **判断依据**:发票抬头(如"电子发票(普通发票)"对应 `electronic_general`)
- **注**:当抬头显示为"电子发票(普通发票)"但号码为 20 位时,仍归入 `electronic_general` 类型

#### `region` 地区
- **优先级 1**:从销售方地址栏提取省/市信息(若发票有此栏位)
- **优先级 2**:从销售方名称中提取城市信息
- **优先级 3(诚实推断)**:从销售方税号前 6 位推断省份
  - 中国统一社会信用代码前 6 位为行政区划代码(GB/T 2260)
  - 例:`914419` → 44 = 广东省
  - 触发此优先级时,**必须遵守诚实推断规则**:`status: uncertain` + warnings 加 `INFERENCE_USED`
- **多城市名判断规则**:
  - **"XX(城市 A)集团/公司 + 城市 B 分公司"形态**:取**城市 B**(实际经营地/开票地),不取城市 A(注册地)
    - 示例:"无印良品(上海)商业有限公司厦门分公司" → 取 **厦门**
  - **"XX 集团 + 子公司"形态(无明确城市标识)**:取首个识别到的城市
- **完全无法判断时**(三个优先级都不适用):`value: null` + `status: missing`

#### `invoice_number` 发票号码
- **长度**:20 位或 8 位
  - **8 位**:2015 年起的传统增值税普票/电子普票
  - **20 位**:2021 年起的数字化电子发票(数电票)
- **注**:`electronic_general` 类型在不同时期对应不同号码长度,8 位或 20 位都是合法形态,**不应基于长度差异判定为格式异常**
- **OCR 易错点**:注意区分字母 O 和数字 0
- **位置**:通常在发票右上角,以"No."或"号码:"开头
- **文字输入容忍**:用户文字描述时,**最少提供后 4 位**即可接受;输出时标注完整度

#### `invoice_date` 开票日期
- **识别格式**:`YYYY 年 MM 月 DD 日` / `YYYY-MM-DD` / `YYYY.MM.DD`
- **输出格式**:统一为 ISO `YYYY-MM-DD`(如 `2026-05-15`)
- **不可推断**:必须从发票上识别,识别失败标 `missing`,**绝不默认填入**

#### `seller_name` 销售方名称
- **位置**:发票上"销售方"区域的"名称"栏
- **形态**:通常是全称(含"有限公司""股份"等)
- **OCR 注意点**:长名称在物理发票上可能换行,OCR 时注意拼接;关键的品牌/字号部分模糊会严重影响有效性

#### `seller_tax_id` 销售方税号
- **位置**:发票上"销售方"区域的"统一社会信用代码/纳税人识别号"栏
- **形态**:通常 18 位字母数字混合(如 `91350200562849225R`)
- **发票上几乎都有此字段**

#### `buyer_name` 购买方名称
- **位置**:发票上"购买方"区域的"名称"栏
- **形态**:可能是个人姓名(如"林悦")、企业全称、或简单标识"个人"
- **可能为空**:部分零售小额发票购买方栏空白

#### `buyer_tax_id` 购买方税号
- **位置**:发票上"购买方"区域的"统一社会信用代码/纳税人识别号"栏
- **个人购买方通常为空**:标 `missing` 是正常情况,**非异常**

#### `amount_pre_tax` 税前金额
- **位置**:发票明细行的"金额"栏(数字小写)
- **输出**:字符串形式,保留原始小数位(如 `"150.00"`)
- **多项目时**:取明细行的合计金额

#### `tax_amount` 税额
- **位置**:发票明细行的"税额"栏(数字小写)
- **输出**:字符串形式
- **多项目时**:取明细行的合计税额

#### `amount_total` 价税合计
- **位置**:发票"价税合计(小写)"栏
- **输出**:字符串形式
- **大写形式**(如"壹佰伍拾玖元整")不需单独提取,仅作为校验参考

#### `items` 项目明细
- **位置**:"货物或应税劳务、服务名称"栏(name)+ "规格型号"栏(spec)
- **name 格式**:常见 `*类别*具体项目`(如 `*餐饮服务*餐费`、`*货物*办公用品`)
- **保留原始格式提取,不剥离星号**
- **spec 格式**:发票"规格型号"列的原文(如 `XL`、`L~XL`、`NF0A8GKJLK6100S`)
- **如果规格型号列为空**:`spec` 设为 `null`
- **数量**:一张发票可能有多项,提取所有项目,输出为 list
- **特殊情况**:多项被合并描述(如"办公用品一批")时,作为单个 item 处理
- **不要做的事**:不要把规格型号拼接到 name 中,也不要把 name 拆分塞进 spec

### Special State Flags(特殊状态标识)

#### `is_red_invoice` 是否红字发票
- **判断依据**:金额为负数 / 发票上明确标识"红字"
- **默认 `false`**

#### `is_voided` 是否作废发票
- **判断依据**:发票上明确标记"作废"字样或红色作废印章
- **默认 `false`**
- **处理方式**:完整提取所有字段 + 标注 `is_voided: true`,**不替调用方判断作废发票的可用性**

### Metadata Field(元数据字段)

#### `registration_date` 登记日期
- **系统元数据**,默认为今天(用户录入时间)
- **不带 status**(不源自发票识别,无识别质量概念)
- **与 `invoice_date` 严格区分**:登记日期 = 用户操作时间;开票日期 = 发票上印的日期

---

## Inference Rules(可推断 vs 不可推断)

### 可推断字段
- `registration_date`:默认填入今天

### 不可推断字段
**所有其他字段**(地区、金额、发票号码、开票日期、商家、消费者、项目等)
- 若识别失败,**必须标 `missing`**
- **绝不默认填入或推测值**

---

## Output Schema

### Output Format
输出为 **JSON** 格式。所有发票内容值保留中文原文;系统标签(`invoice_type`、`status`、`warning_code`、`warning_level`)使用英文枚举。数值字段以字符串形式输出,以保留原始格式并避免浮点精度问题。

### Schema Structure

```json
{
  "schema_version": "1.2",
  "registration_date": "<ISO 格式日期,系统默认为今天>",
  "invoice_count": 1,
  "invoices": [
    {
      "invoice_type": {
        "value": "VAT_general | VAT_special | electronic_general | fully_digitized | null",
        "status": "clear | partial | uncertain | missing"
      },
      "is_red_invoice": false,
      "is_voided": false,
      "region": { "value": "<省/市,如 '厦门市'> | null", "status": "..." },
      "invoice_number": { "value": "<8 位或 20 位字符串> | null", "status": "..." },
      "invoice_date": { "value": "<ISO 格式日期> | null", "status": "..." },
      "seller_name": { "value": "<销售方全称> | null", "status": "..." },
      "seller_tax_id": { "value": "<18 位字母数字> | null", "status": "..." },
      "buyer_name": { "value": "<购买方名称> | null", "status": "..." },
      "buyer_tax_id": { "value": "<18 位字母数字> | null", "status": "..." },
      "amount_pre_tax": { "value": "<金额字符串,如 '150.00'> | null", "status": "..." },
      "tax_amount": { "value": "<税额字符串> | null", "status": "..." },
      "amount_total": { "value": "<价税合计字符串> | null", "status": "..." },
      "items": {
        "value": [
          {
            "name": "<*类别*具体项目>",
            "spec": "<规格型号> | null",
            "status": "clear | partial | uncertain"
          }
        ],
        "count": 1,
        "status": "clear | missing"
      },
      "warnings": []
    }
  ]
}
```

### Status Definitions(字段状态定义)

每个字段的 status 反映该字段的识别质量。

| Status | 含义 | 示例 |
|---|---|---|
| `clear` | 字段所有字符清晰可辨,且符合预期格式 | 发票号码完整 8 位且每位都清楚 |
| `partial` | 字段被识别到,但**结构性不完整** | 金额只识别到整数,小数部分被遮挡 |
| `uncertain` | 字段所有部分都识别到了,但**某些字符存在歧义** | 发票号码 `1234567?`,某位可能是 8 也可能是 6 |
| `missing` | 字段完全未识别到,或发票上根本没有 | 购买方栏完全空白 |

**关键区分**:
- `partial` vs `uncertain`:`partial` 是**位置/范围**不完整;`uncertain` 是**字符**不确定
- `partial` / `uncertain` vs `missing`:前两者 `value` 不为 null,仅质量打折;`missing` 时 `value` 必为 `null`

#### `uncertain` 状态输出规则(强制)

When marking a field as `uncertain`, the `value` MUST indicate the uncertain position(s) using `?` placeholder, not a best-guess value:
- Single uncertain character: `"913502033028124?7Y"`
- Multiple uncertain characters: `"913502033??812477Y"`

The `warnings` entry MUST specify which position(s) are uncertain, not point to a wrong location.

**Do NOT** output a single best-guess value labeled as `uncertain` — that defeats the purpose of the status system. If you must commit to a specific value, label it `clear`. If you genuinely don't know specific characters, mark them with `?` and explain in warnings.

#### `partial` 状态:打码 / 遮挡 / 模糊 / 截断处理规则

`partial` 状态适用于以下**所有**场景 —— 任何"字段部分不可见"的情况:

1. **图像截断(Image truncation)**:字段被图像边缘截断
   - 已知截断位置:用 `*` 标缺失位置(如 `2631****` 表示后 4 位被截断)
   - 截断位置不明:value 设为 `null`,status 设为 `missing`

2. **隐私打码 / 马赛克(Privacy redaction / mosaic)**:字段被刻意遮挡
   - 每个被遮挡字符用一个 `*` 占位(保留长度信息)

3. **水印 / 印章覆盖(Watermark / stamp overlap)**:文字被水印或印章覆盖
   - 被覆盖的每个字符用一个 `*` 占位

4. **严重模糊(Severe blur)**:字符存在但完全不可读
   - 不可读的每个字符用一个 `*` 占位
   - 注意:**严重模糊(看不出来是什么字符)用 `*`;轻微模糊(有歧义但能猜)用 `?` + uncertain 状态**

**在以上所有场景**:
- `value`:已识别的字符按原样保留,不可识别的位置用 `*` 占位
- `status`:`partial`
- `warnings`:必须描述遮挡类型(privacy redaction / watermark / image truncation / severe blur)

Examples:
- 中间被打码的发票号:`"2631****8086"` (status: partial)
- 部分遮挡的销售方:`"广**科技有限公司"` (status: partial)
- 水印遮挡的税号:`"91441900M***GG26"` (status: partial)
- 边缘截断的发票号:`"2631200000****"` (status: partial)

**关键禁止**:
- **绝不**在被遮挡的位置填入具体字符 / 数字(即使该字段"通常"是 8 位或 20 位)
- **绝不**为了让 value 看起来"完整"而补全不可见部分
- **绝不**把"打码区域"当作"模糊但可识别"来对待

这是 ANTI-HALLUCINATION REQUIREMENT 在 partial 状态下的具体执行规则。

### Items Field Notes(items 字段特殊处理)

- 内部每项自带 `status`(`clear` / `partial` / `uncertain`)
- 外层 `status` 永远存在:
  - **正常情况**:`"clear"` + `count` 反映实际项目数
  - **完全 missing**(发票上根本没识别到项目):`"missing"` + `value: null` + `count: 0`
- 内部字段:`name`(项目名,必有)+ `spec`(规格型号,可为 null)
- `spec` 为 null 时,不影响整体 status 判断

### Warnings Field(警告字段)

`warnings` 字段记录**字段层面的关联性异常**或**字段值的推断来源**,与 `status`(单字段质量)互为补充。

```json
"warnings": [
  {
    "warning_code": "FIELD_FORMAT_INVALID | FIELDS_INCONSISTENT | IDENTICAL_PARTY_NAMES | INFERENCE_USED",
    "warning_level": "LOW | MEDIUM | HIGH",
    "field_names": ["<相关字段名,可多个>"],
    "description": "<人类可读说明>"
  }
]
```

**Warning Codes**:
- `FIELD_FORMAT_INVALID`:单字段格式不符合预期(如发票号码长度异常)
- `FIELDS_INCONSISTENT`:多字段间存在矛盾(如 amount_pre_tax + tax_amount ≠ amount_total)
- `IDENTICAL_PARTY_NAMES`:销售方与购买方名称相同(可能开票错误)
- `INFERENCE_USED`:字段值通过推断得到,而非直接从发票视觉证据识别(详见下面"诚实推断规则")

**Warning Levels**:
- `LOW`:字段质量打折,但数据基本可用
- `MEDIUM`:数据存在明显问题,可用性下降
- `HIGH`:数据存在严重异常,可能不可靠

**当无 warning 时**:`warnings: []`(永远是数组,空时为空数组)

#### 诚实推断规则(Honest Inference Rule)

ANTI-HALLUCINATION REQUIREMENT 明确禁止"幻觉补全"。但在**某些场景下**,基于**可见证据的辅助信息**推断字段值是允许的(例:从销售方税号前 6 位推断地区)。

**触发条件**:
- 字段无法从其主要来源直接识别(如 region 无法从地址栏或销售方名称识别)
- 但从**同一发票上的其他可见、可靠字段**(如税号前缀)可以**推断**出一个**有依据的值**

**强制要求**:
- 字段 `status` **必须**标为 `uncertain`(不可标 clear)
- **必须**在 `warnings` 中加 `INFERENCE_USED` 条目
- `warning_level` 通常为 `LOW` 或 `MEDIUM`
- `description` **必须**说明:**推断出什么 + 基于什么证据**

Example:

```json
"region": {
  "value": "广东省",
  "status": "uncertain"
}
...
"warnings": [
  {
    "warning_code": "INFERENCE_USED",
    "warning_level": "LOW",
    "field_names": ["region"],
    "description": "Region inferred from seller_tax_id prefix '914419' (44 = Guangdong province code). Not directly visible on invoice."
  }
]
```

**注意区分**:
- **诚实推断**(允许):基于发票上**可见、可靠的其他字段**推断一个有具体依据的值,标 uncertain + INFERENCE_USED
- **幻觉补全**(禁止,见 ANTI-HALLUCINATION REQUIREMENT):为不可见的字段位置填入猜测的字符

**Warnings 排序规则**:

When multiple warnings exist, sort them by:
1. `warning_level` priority: HIGH → MEDIUM → LOW
2. Within the same level, sort by field appearance order in the schema (e.g., warnings about `invoice_number` come before warnings about `amount_total`)

This ensures downstream consumers address most critical issues first.

### Schema Versioning

`schema_version` 字段标识 schema 版本。字段定义、状态枚举或结构发生不兼容变更时,主版本号(`1.0` → `2.0`)递增;新增可选字段且向后兼容时,次版本号(`1.0` → `1.1`)递增。

**v1.2 相对 v1.1 的变更**:
- 新增 ANTI-HALLUCINATION REQUIREMENT 顶层强制规则(禁止幻觉补全)
- `partial` 状态明确扩展到 4 个具体场景(图像截断 / 隐私打码 / 水印覆盖 / 严重模糊),并明确该状态下"绝不补全不可见字符"
- 新增 `INFERENCE_USED` warning_code,用于标记"诚实推断"(如从税号前缀推断地区)。这是向后兼容的新增枚举值(老的下游解析器看到陌生 warning_code 应忽略而非报错)
- region 字段新增优先级 3:税号前缀推断(必须配合 INFERENCE_USED warning)

**v1.1 相对 v1.0 的变更(历史记录)**:
- `items[].text` 字段拆分为 `items[].name` + `items[].spec`
- 新增 `uncertain` 状态的位置标注规则(`?` 占位符)
- 新增 `partial` 状态的打码占位符规则(`*` 占位符)
- 新增 warnings 排序规则
- 顶部加 CRITICAL OUTPUT REQUIREMENT

---

## Extraction Process

### Step 1: Input Validation

Before extraction, verify that the input contains Chinese invoice characteristics (e.g., invoice header, tax ID section, seller/buyer fields).

- If the input is **not an invoice**(receipt, photo of unrelated content, non-invoice document)→ Return an error indicating non-invoice input
- If the input is a **non-Chinese invoice** → Return an error indicating the Skill is designed for Chinese invoices only

### Step 2: Invoice Type Identification

Identify the invoice type from:`VAT_general`、`VAT_special`、`electronic_general`、`fully_digitized`。If type cannot be determined, set `invoice_type` status to `uncertain` or `missing` accordingly。

### Step 3: Field Extraction

Extract all fields as defined in **Field Specifications**. For each field:
- Assess recognition quality and assign `status`(`clear` / `partial` / `uncertain` / `missing`)
- For `missing` fields, set `value: null`
- For `uncertain` fields, mark uncertain character positions with `?`
- For `partial` fields due to obstruction/redaction, mark obscured characters with `*`

Apply field-specific extraction rules — e.g., multi-page PDFs of the same invoice should be merged before extraction; multiple separate invoices should be output as separate items in the `invoices` array。

### Step 4: Cross-Field Validation

After extraction, check for cross-field anomalies:
- Whether `amount_pre_tax + tax_amount` matches `amount_total`
- Whether `seller_name` and `buyer_name` are identical
- Whether field formats match expected patterns(e.g., invoice_number length)

Record any anomalies in the `warnings` array (sorted by HIGH → MEDIUM → LOW)。

### Step 5: Special Status Marking

Check and mark special invoice states:
- `is_red_invoice: true` if the invoice is a red(reversing)invoice
- `is_voided: true` if the invoice is marked as voided

### Step 6: Output Assembly

Assemble the final output following the Output Schema:
- Top-level fields:`schema_version`、`registration_date`、`invoice_count`
- For each invoice, include all extracted fields, warnings, and special status flags
- If `invoice_count > 1`, ensure all invoices are in the `invoices` array

**Escalation rule**:If no field can be extracted with status `clear` or `uncertain`, return an error rather than an all-`missing` schema。

---

## Edge Case Handling(边界处理)

边界处理分 3 个等级:
- **等级 1**:Skill 内部处理,无需告知调用方
- **等级 2**:Skill 处理 + 在输出中标注
- **等级 3**:Skill 拒绝处理,返回错误

### A. 输入合法性(Input Validity)

| 边界情况 | 等级 | 处理 |
|---|---|---|
| 上传非发票文件(收据、小票、其他文档) | 3 | 返回错误,前置检查拒绝 |
| 上传境外发票(不适用中国发票规范) | 3 | 返回错误,理由:字段规范不适配 |
| 图片严重损坏 | 3 | 返回错误 |
| 图片倒置 | 1 | 内部纠正方向 |
| 完全模糊(无任何字段可清晰识别) | 3 | 升级阈值:无任何 `clear` 字段时升级为错误 |

### B. 输入数量(Input Quantity)

| 边界情况 | 等级 | 处理 |
|---|---|---|
| 上传多张发票 | 1.5 | 统一用 `invoices` 数组结构,每张独立输出 |
| 跨页 PDF(同一张发票分多页) | 1 | 内部识别+合并;需区分"跨页同张" vs "多张独立" |

**跨页 vs 多张判断逻辑**:
- 如果每页都有完整的发票号、销售方、金额 → 多张独立发票
- 如果只有部分页有完整字段,其他页是延续 → 一张跨页发票

### C. 发票本身状态(Invoice State)

| 边界情况 | 等级 | 处理 |
|---|---|---|
| 红字发票(冲销/负数金额) | 2 | `is_red_invoice: true` + 完整提取 |
| 作废发票 | 2 | `is_voided: true` + 完整提取(方案 A:不替调用方判断作废发票可用性) |
| 手开发票(OCR 难度大) | 2 | 尽力识别,质量打折用 `status` 反映 |
| 多联次(记账联/抵扣联) | — | 非边界,正常处理 |

### D. 字段层面异常(Field-Level Anomalies)

| 边界情况 | 等级 | 处理 |
|---|---|---|
| 字段缺失 | 2 | `status: missing`,`value: null` |
| 字段轻微模糊(字符存歧义) | 2 | `status: uncertain` + 在不确定位置用 `?` 占位 |
| 字段被打码 / 水印 / 截断 / 严重模糊 | 2 | `status: partial` + 在不可见位置用 `*` 占位(每字符一个 `*`)。**绝不补全不可见部分** |
| 字段无法直接识别但可从其他字段推断 | 2 | `status: uncertain` + value 给推断值 + `warnings: INFERENCE_USED` 说明推断依据 |
| 字段格式异常(如发票号码长度异常) | 2 | `warnings: FIELD_FORMAT_INVALID` |
| 字段间矛盾(金额加起来对不上) | 2 | `warnings: FIELDS_INCONSISTENT` |
| 同字段多值(销售方与购买方名称相同) | 2 | `warnings: IDENTICAL_PARTY_NAMES` |

### E. 元数据(Metadata)

- **用户明确指定开票日期(如"这张是上周的发票")**:不是冲突。`invoice_date` 从发票识别,`registration_date` 仍为今天,两个字段独立。

---

For complete examples of expected output, see `references/examples.md`
