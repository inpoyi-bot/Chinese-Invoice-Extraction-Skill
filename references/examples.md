## Output Examples

### Example 1: Standard Invoice(标准正常发票,单项目)

**输入场景**:一张厦门味友鸭肉面线馆的电子普通发票,所有字段清晰可读。规格型号栏为空。

**输出 schema**:

```json
{
  "schema_version": "1.2",
  "registration_date": "2026-05-22",
  "invoice_count": 1,
  "invoices": [
    {
      "invoice_type": { "value": "electronic_general", "status": "clear" },
      "is_red_invoice": false,
      "is_voided": false,
      "region": { "value": "厦门市", "status": "clear" },
      "invoice_number": { "value": "26942000000599520736", "status": "clear" },
      "invoice_date": { "value": "2026-05-06", "status": "clear" },
      "seller_name": { "value": "厦门味友鸭肉面线馆", "status": "clear" },
      "seller_tax_id": { "value": "91350203302812477Y", "status": "clear" },
      "buyer_name": { "value": "林悦", "status": "clear" },
      "buyer_tax_id": { "value": null, "status": "missing" },
      "amount_pre_tax": { "value": "133.96", "status": "clear" },
      "tax_amount": { "value": "8.04", "status": "clear" },
      "amount_total": { "value": "142.00", "status": "clear" },
      "items": {
        "value": [
          { "name": "*餐饮服务*餐费", "spec": null, "status": "clear" }
        ],
        "count": 1,
        "status": "clear"
      },
      "warnings": []
    }
  ]
}
```

**这个例子教 Agent**:
- 标准 schema 形态
- 所有字段 `clear` 状态
- 个人 buyer 的 `buyer_tax_id` 为 `missing`(正常情况)
- region 走兜底路径从销售方名称提取
- items 单项 + spec 为 null 的标准形态

---

### Example 2: Multi-Item Invoice with Specifications(多项目发票,带规格型号)

**输入场景**:一张无印良品的电子普通发票,有 2 个项目,均有规格型号。双城市名销售方(注册地上海,经营地厦门)。购买方为个人。

**输出 schema**:

```json
{
  "schema_version": "1.2",
  "registration_date": "2026-05-22",
  "invoice_count": 1,
  "invoices": [
    {
      "invoice_type": { "value": "electronic_general", "status": "clear" },
      "is_red_invoice": false,
      "is_voided": false,
      "region": { "value": "厦门市", "status": "clear" },
      "invoice_number": { "value": "26942000000594314176", "status": "clear" },
      "invoice_date": { "value": "2026-05-03", "status": "clear" },
      "seller_name": {
        "value": "无印良品(上海)商业有限公司厦门分公司",
        "status": "clear"
      },
      "seller_tax_id": { "value": "91350200562849225R", "status": "clear" },
      "buyer_name": { "value": "林岗(个人)", "status": "clear" },
      "buyer_tax_id": { "value": null, "status": "missing" },
      "amount_pre_tax": { "value": "222.12", "status": "clear" },
      "tax_amount": { "value": "28.88", "status": "clear" },
      "amount_total": { "value": "251.00", "status": "clear" },
      "items": {
        "value": [
          {
            "name": "*服装*天竺编织圆领短袖 T 恤",
            "spec": "XL",
            "status": "clear"
          },
          {
            "name": "*服装*【中】凉感天竺编织短袖家居睡衣套装男士",
            "spec": "L~XL",
            "status": "clear"
          }
        ],
        "count": 2,
        "status": "clear"
      },
      "warnings": []
    }
  ]
}
```

**这个例子教 Agent**:
- 多项目 items 数组(2 项)
- 每项的 `name` 与 `spec` 严格分离 —— 不要把规格型号拼接到 name 中
- 长项目名称的处理(保留完整名称,包括"【中】"等装饰符)
- 双城市销售方名称的 region 提取(取实际经营地"厦门",不取注册地"上海")
- buyer 含"(个人)"标识为发票原文,不是 AI 推测的补充
- 个人 buyer 的 `buyer_tax_id` 为 `missing`(正常)

---

### Example 3: Red Invoice with Mixed Status(红字发票,字段状态多样)

**输入场景**:一张红字发票(冲销发票),金额均为负数;部分字段因图片裁切或图像质量出现质量打折。

**输出 schema**:

```json
{
  "schema_version": "1.2",
  "registration_date": "2026-05-22",
  "invoice_count": 1,
  "invoices": [
    {
      "invoice_type": { "value": "electronic_general", "status": "clear" },
      "is_red_invoice": true,
      "is_voided": false,
      "region": { "value": "北京市", "status": "clear" },
      "invoice_number": { "value": null, "status": "partial" },
      "invoice_date": { "value": null, "status": "partial" },
      "seller_name": {
        "value": "北京洺月锦堂餐饮管理有限责任公司望京分公司",
        "status": "clear"
      },
      "seller_tax_id": { "value": "91110105348275531R", "status": "clear" },
      "buyer_name": { "value": "北京三快在线科技有限公司", "status": "clear" },
      "buyer_tax_id": {
        "value": "911101085621441?0X",
        "status": "uncertain"
      },
      "amount_pre_tax": { "value": "-9839.62", "status": "clear" },
      "tax_amount": { "value": "-590.38", "status": "clear" },
      "amount_total": { "value": "-10430.00", "status": "clear" },
      "items": {
        "value": [
          { "name": "*餐饮服务*餐费", "spec": null, "status": "clear" }
        ],
        "count": 1,
        "status": "clear"
      },
      "warnings": [
        {
          "warning_code": "FIELD_FORMAT_INVALID",
          "warning_level": "LOW",
          "field_names": ["invoice_number"],
          "description": "发票号码因截图裁切仅显示前 7 位,完整 20 位号码不可见"
        },
        {
          "warning_code": "FIELD_FORMAT_INVALID",
          "warning_level": "LOW",
          "field_names": ["invoice_date"],
          "description": "开票日期因截图裁切仅显示'2026 年',月日不可见"
        },
        {
          "warning_code": "FIELD_FORMAT_INVALID",
          "warning_level": "LOW",
          "field_names": ["buyer_tax_id"],
          "description": "购买方税号倒数第 3 位字符存在歧义,可能为 '1' 或 'l',已用 `?` 标注于该位置"
        }
      ]
    }
  ]
}
```

**这个例子教 Agent**:
- 红字发票:`is_red_invoice: true` + 所有金额带负号
- `partial` 状态:`invoice_number` / `invoice_date` 因图像截断导致结构性不完整,`value: null`
- `uncertain` 状态:`buyer_tax_id` 用 `?` 标注**具体位置**,而不是给一个完整的猜测值
- warnings 排序:多个 LOW 级别,按字段在 schema 中出现顺序排列(invoice_number → invoice_date → buyer_tax_id)
- 复杂多部分销售方名(含"分公司")完整保留

---

### Example 4: Redacted Invoice (Privacy Protection)(打码发票)

**输入场景**:一张红字发票,销售方和购买方名称、税号均被打码遮挡(隐私保护)。

**输出 schema**:

```json
{
  "schema_version": "1.2",
  "registration_date": "2026-05-22",
  "invoice_count": 1,
  "invoices": [
    {
      "invoice_type": { "value": "electronic_general", "status": "clear" },
      "is_red_invoice": true,
      "is_voided": false,
      "region": { "value": null, "status": "missing" },
      "invoice_number": { "value": "2631****48086", "status": "partial" },
      "invoice_date": { "value": "2026-03-27", "status": "clear" },
      "seller_name": { "value": "广**科技有限公司", "status": "partial" },
      "seller_tax_id": { "value": "91441900M***GG26", "status": "partial" },
      "buyer_name": { "value": "**业发展有限公司", "status": "partial" },
      "buyer_tax_id": { "value": "9*************G36", "status": "partial" },
      "amount_pre_tax": { "value": "-701.77", "status": "clear" },
      "tax_amount": { "value": "-91.23", "status": "clear" },
      "amount_total": { "value": "-793.00", "status": "clear" },
      "items": {
        "value": [
          {
            "name": "*服装*【优惠价】北面女短款防风防水硬壳冲锋衣户外新款外套The NorthFace",
            "spec": "NF0A8GKJLK6100S",
            "status": "clear"
          }
        ],
        "count": 1,
        "status": "clear"
      },
      "warnings": [
        {
          "warning_code": "FIELD_FORMAT_INVALID",
          "warning_level": "MEDIUM",
          "field_names": ["seller_name", "seller_tax_id", "buyer_name", "buyer_tax_id"],
          "description": "销售方、购买方名称及税号因隐私打码部分遮挡,使用 * 标注遮挡位置"
        },
        {
          "warning_code": "FIELD_FORMAT_INVALID",
          "warning_level": "LOW",
          "field_names": ["invoice_number"],
          "description": "发票号码中间位被遮挡,仅前 4 位和后 5 位可见"
        }
      ]
    }
  ]
}
```

**这个例子教 Agent**:
- **打码处理标准**:每个不可识别的字符用一个 `*` 占位,**保留位置信息**
- 打码字段的 `status` 标为 `partial`(不是 `uncertain` —— 字符不是"有歧义",而是"看不见")
- 单项目带规格型号(spec)的形态:服装类的型号编码
- warnings 排序示例:MEDIUM 在 LOW 之前
- **不要做的事**:不要尝试猜测打码内容(如不要写"广州 XX 科技有限公司")
- `region` 在此例标 `missing`,展示**保守处理**(销售方名称完全打码 + 不主动推断);另见 Example 5 展示**同场景下使用 `INFERENCE_USED` 的诚实推断处理**

---

### Example 5: Honest Inference(诚实推断 — 从税号推断地区)

**输入场景**:发票上销售方地址栏为空,销售方名称完全打码无法提取城市,但销售方税号 `914419...` 可见。此时按 region 优先级 3(诚实推断)从税号前 6 位推断省份。

**输出 schema(关键片段)**:

```json
{
  "schema_version": "1.2",
  "registration_date": "2026-05-23",
  "invoice_count": 1,
  "invoices": [
    {
      "invoice_type": { "value": "electronic_general", "status": "clear" },
      "is_red_invoice": true,
      "is_voided": false,
      "region": {
        "value": "广东省",
        "status": "uncertain"
      },
      "seller_name": { "value": "广**科技有限公司", "status": "partial" },
      "seller_tax_id": { "value": "91441900M***GG26", "status": "partial" },
      ... 其他字段 ...
      "warnings": [
        {
          "warning_code": "INFERENCE_USED",
          "warning_level": "LOW",
          "field_names": ["region"],
          "description": "Region inferred from seller_tax_id prefix '914419' (44 = Guangdong province code per GB/T 2260). Not directly visible on invoice."
        }
      ]
    }
  ]
}
```

**这个例子教 Agent**:
- **何时使用 INFERENCE_USED**:当字段无法从主要来源(地址栏/名称)识别,但可从同发票上其他可见证据推断
- `status` **必须**标 `uncertain`(不可标 clear)
- warnings 必须**明确说明推断依据**(具体是从哪个字段、什么前缀推断的)
- **关键区别**:这是"诚实推断"(基于可见证据 + 标 uncertain + warning 说明),不是"幻觉补全"
