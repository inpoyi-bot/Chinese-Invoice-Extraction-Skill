# Chinese Invoice Extraction — Codex & Claude

A portable Agent Skill for extracting structured data from Chinese invoices (image, PDF, or text description). The same `SKILL.md` can be installed in Codex or Claude; it does not rely on provider-specific tool names.

> **Status**: schema v1.2 — Historical testing was performed on Claude Sonnet; extraction quality remains model-dependent. See [Known Limitations](#known-limitations) before use.

---

## What It Does

Extracts the following structured information from Chinese invoices:

- **Invoice metadata**: type, region, invoice number, invoice date
- **Parties**: seller name & tax ID, buyer name & tax ID
- **Amounts**: pre-tax amount, tax amount, total (preserves negative signs for red invoices)
- **Items**: itemized list with `name` (e.g., `*服装*短袖 T 恤`) and `spec` (e.g., `XL`)
- **Special states**: red invoice (冲销) flag, voided invoice flag
- **Quality annotations**: each field tagged with status (`clear` / `partial` / `uncertain` / `missing`) and cross-field warnings

The Skill outputs **strict JSON** following a defined schema (see `SKILL.md` for the full schema definition).

---

## What It Does NOT Do

- **Does not verify invoice authenticity** — extraction only, no fraud detection
- **Does not perform OCR by itself** — relies on the current platform's available vision and document-reading capability
- **Does not handle non-Chinese invoices** — designed specifically for the Chinese 发票 (fapiao) format
- **Does not de-duplicate or match invoices** — this is a single-invoice extraction Skill
- **Does not store or persist data** — stateless extraction; downstream storage is the caller's responsibility

---

## Installation

### Claude

1. Download this repository as a ZIP, or clone it
2. Go to Claude.ai → Settings → Skills
3. Click "Add skill" and upload the `chinese-invoice-extraction` folder (or ZIP)
4. Enable the Skill

### Codex

Copy or install the `chinese-invoice-extraction` folder into your Codex skills directory (commonly `~/.codex/skills/`), then start a new Codex task. Codex can load the same `SKILL.md` when the task involves extracting Chinese invoice data.

---

## Quick Example

**Input**: Upload a Chinese invoice image

**Output**:

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
      "region": { "value": "上海市", "status": "clear" },
      "invoice_number": { "value": "26317000001426205777", "status": "clear" },
      "invoice_date": { "value": "2026-05-03", "status": "clear" },
      "seller_name": { "value": "上海茵赫实业有限公司", "status": "clear" },
      "seller_tax_id": { "value": "91310115MA1H70PK5R", "status": "clear" },
      "buyer_name": { "value": "林悦", "status": "clear" },
      "buyer_tax_id": { "value": null, "status": "missing" },
      "amount_pre_tax": { "value": "94.34", "status": "clear" },
      "tax_amount": { "value": "5.66", "status": "clear" },
      "amount_total": { "value": "100.00", "status": "clear" },
      "items": {
        "value": [
          { "name": "*餐饮服务*餐饮服务", "spec": null, "status": "clear" }
        ],
        "count": 1,
        "status": "clear"
      },
      "warnings": []
    }
  ]
}
```

For more examples (multi-item invoices, red invoices, redacted/partial visibility cases), see [`references/examples.md`](references/examples.md).

---

## Repository Structure

```
chinese-invoice-extraction/
├── README.md              # This file
├── LICENSE                # MIT License
├── SKILL.md               # Skill definition (read by the agent)
└── references/
    └── examples.md        # Detailed output examples for various scenarios
```

---

## Known Limitations

### Platform and model dependency

**The quality of extraction depends heavily on the vision and document-reading capability available in the current platform. The Skill itself does not perform OCR — it relies on the model's ability to read invoice images.**

#### Historical test configuration

This Skill has been validated **only on Claude Sonnet** (via Claude.ai web client, May 2026).

| Model | Status | Notes |
|---|---|---|
| Sonnet | ✅ Tested | Passes standard test cases (see [Test Results](#test-results)) |
| Haiku | ⚠️ Known to fail | In a single-invoice spot check, produced multiple OCR errors (misread digits, character substitutions). The status system **did not catch these errors** — fields were labeled `clear` despite being wrong. **Not recommended.** |
| Opus | ❓ Untested | May perform similarly to or better than Sonnet, but no validation has been performed |

This testing scope reflects the project's nature as a personal learning artifact, not an industrial-grade product. Independent multi-model validation has not been conducted.

#### Choosing a capable model

Use a vision-capable model in your chosen platform. Claude.ai's default model varies by subscription tier and conversation type; Codex model selection depends on your configured surface and workspace.

- **Claude.ai web/app**: The model in use is visible at the bottom of the message input area or in conversation settings
- **Claude**: Check the model shown in the conversation UI or settings before uploading an invoice.
- **Codex**: Use the model configured for your task or workspace, and verify that it can inspect the supplied image or PDF.

#### For Production / Critical Use

If you intend to use this Skill for critical workflows (tax filing, reimbursement, accounting), **do not rely solely on this Skill's testing**. Independent validation in your specific environment and use cases is strongly recommended. See also the [Status System Caveat](#status-system-caveat) below.

### OCR-Inherent Limitations

Even on Sonnet, the following remain challenging:

- Long product names with mixed Chinese/English (e.g., brand model codes)
- Watermarked / stamped areas overlapping with text
- Very low-resolution or heavily compressed images

These are limits of the underlying vision model, **not** limits of this Skill's design.

### Status System Caveat

The `status` field reflects the model's *self-assessment* of recognition quality. In practice:

- A field tagged `clear` is **not guaranteed correct** — the model may not know it misread a character
- A field tagged `uncertain` uses `?` placeholders at unclear positions (per v1.2 spec)
- A field tagged `partial` uses `*` placeholders for obscured characters (per v1.2 spec)

**Downstream callers should treat `clear` as "high confidence", not "verified correct"**. For critical use (tax filing, reimbursement), human verification of key fields (invoice number, amounts, seller name) is recommended.

### Other Known Issues

(To be filled in after round-2 testing)

---

## Test Results

> **__TODO: Fill in after round-2 testing__**
>
> Planned content:
> - Phase 1 trigger test results (6 cases)
> - Phase 2 execution test results (E-1 to E-5)
> - Per-improvement-point verification (CRITICAL OUTPUT / uncertain `?` / partial `*` / items split / warnings ordering)
> - Pass rate summary
> - Test date, model used

---

## Contributing

This is currently a personal learning project. Issues and observations are welcome via GitHub Issues.

PRs are not actively solicited at this stage — if you'd like to contribute, please open an issue first to discuss.

---

## Versioning

Current version: **1.1**

Schema versioning follows the rule in `SKILL.md` § Schema Versioning:
- Major (e.g., 1.x → 2.0): incompatible schema changes
- Minor (e.g., 1.0 → 1.1): backward-compatible additions or refinements

See `SKILL.md` for full version history.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

## Acknowledgments

This Skill was designed and tested through iterative collaboration with Claude (Anthropic).
The design process — including problem identification, schema design, edge case mapping, and round-2 testing methodology — is itself a learning artifact in AI product design.
