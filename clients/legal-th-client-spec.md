# Legal-TH Client Plugin Specification

**สำหรับทีมพัฒนา Client Plugin**

---

## 1. ภาพรวม

Legal-TH Client Plugin เป็น plugin ฝั่ง client ที่ติดตั้งบน AI coding agent เพื่อเชื่อมต่อกับ Legal-TH MCP Server ให้ผู้ใช้สามารถใช้งานฟีเจอร์กฎหมายไทยผ่าน slash commands ได้โดยตรง

```
┌──────────────────────────────────────┐
│  ผู้ใช้ (ทีมกฎหมาย / นักพัฒนา)        │
│  พิมพ์: /review-contract สัญญา.pdf   │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│  AI Coding Agent                     │
│  (Claude Code / OpenCode / อื่นๆ)    │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  Legal-TH Client Plugin       │  │
│  │  ├─ Commands (slash commands)  │  │
│  │  ├─ Skills (knowledge)        │  │
│  │  ├─ Playbook (config)         │  │
│  │  └─ MCP Config (→ server)     │  │
│  └────────────────────────────────┘  │
└──────────────┬───────────────────────┘
               │ MCP HTTP
               ▼
┌──────────────────────────────────────┐
│  Legal-TH MCP Server (Cloud)        │
│  (ดู server/legal-th-server-spec.md) │
└──────────────────────────────────────┘
```

**หน้าที่ของ client plugin:**
- ให้ slash commands แก่ผู้ใช้ (workflow ทีละขั้นตอน)
- ให้ skills (ความรู้กฎหมายไทย) แก่ AI
- ให้ playbook config สำหรับปรับจุดยืนขององค์กร
- เชื่อมต่อ MCP server สำหรับข้อมูลที่ต้อง remote

**หน้าที่ที่ไม่ใช่ของ client plugin:**
- ไม่มี business logic ซับซ้อน (อยู่บน server)
- ไม่เก็บข้อมูลกฎหมาย (ดึงจาก server)
- ไม่มี database

---

## 2. Supported Platforms

| Platform | Status | Directory | หมายเหตุ |
|----------|--------|-----------|---------|
| **Claude Code / Cowork** | ✅ พร้อม | `legal-th-plugin-claude/` | ใช้ .claude-plugin format |
| **OpenCode** | ✅ พร้อม | `legal-th-plugin-opencode/` | ใช้ .opencode format |
| **GitHub Copilot** | 🔲 ยังไม่ทำ | `legal-th-plugin-copilot/` | รอศึกษา plugin format |
| **Cursor** | 🔲 ยังไม่ทำ | `legal-th-plugin-cursor/` | รอศึกษา plugin format |
| **ChatGPT / Codex** | 🔲 ยังไม่ทำ | `legal-th-plugin-chatgpt/` | รอศึกษา plugin format |

### ทำไมต้องแยก directory แต่ละค่าย

แต่ละ AI coding agent มี plugin format ต่างกัน:

| รายการ | Claude Code | OpenCode |
|--------|------------|---------|
| Config file | `.claude-plugin/plugin.json` | `opencode.json` |
| MCP config | `.mcp.json` | `opencode.json > "mcp"` |
| Commands path | `commands/` | `.opencode/commands/` |
| Skills path | `skills/` | `.opencode/skills/` |
| Project context | ไม่มี | `AGENTS.md` |
| JS/TS plugin | ไม่มี | `.opencode/plugins/` |
| Custom tools | ไม่มี | `.opencode/tools/` |

**แต่เนื้อหา commands + skills + playbook เหมือนกันทุกค่าย** — ต่างกันแค่โครงสร้าง/config

---

## 3. Plugin Components

### 3.1 Commands (Slash Commands)

Commands คือ workflow ที่ผู้ใช้เรียกผ่าน `/command-name` แต่ละ command เป็นไฟล์ markdown ที่บอก AI ว่าต้องทำอะไร ขั้นตอนไหน output เป็นยังไง

| Command | ไฟล์ | คำอธิบาย | ใช้ MCP Server |
|---------|------|---------|---------------|
| `/review-contract` | `review-contract.md` | รีวิวสัญญาตาม playbook (GREEN/YELLOW/RED) | ไม่จำเป็น (ใช้ knowledge ของ AI + playbook) |
| `/compliance-check` | `compliance-check.md` | ตรวจ compliance ตามกฎหมายไทย | ใช้ `legal_th/search_laws` เพื่อหากฎหมายที่เกี่ยวข้อง |
| `/triage-nda` | `triage-nda.md` | คัดกรอง NDA (GREEN/YELLOW/RED) | ไม่จำเป็น |
| `/search-laws` | `search-laws.md` | ค้นหากฎหมายจากฐานข้อมูล | ใช้ `legal_th/search_laws` **(ต้องมี server)** |
| `/glossary` | `glossary.md` | ค้นหาศัพท์กฎหมายไทย-อังกฤษ | ใช้ `legal_th/glossary_lookup` **(ต้องมี server)** |
| `/brief` | `brief.md` | สรุปรายวัน / หัวข้อ / เหตุการณ์ | ใช้ `legal_th/regulatory_updates` (ถ้ามี) |

**สำคัญ:** commands 3 ตัว (`/review-contract`, `/compliance-check`, `/triage-nda`) **ใช้ได้ทันทีโดยไม่ต้องมี MCP server** เพราะใช้ knowledge ของ AI + playbook + skills เป็นหลัก

commands อีก 3 ตัว (`/search-laws`, `/glossary`, `/brief`) **ต้องมี MCP server** จึงจะทำงานเต็มที่

### 3.2 Skills (Knowledge)

Skills คือ knowledge base ที่ AI โหลดเมื่อต้องใช้ความรู้เฉพาะทาง เป็นไฟล์ markdown ที่ให้ข้อมูลเชิงลึก

| Skill | ไฟล์ | เนื้อหา |
|-------|------|---------|
| `thai-contract-review` | `SKILL.md` | กฎหมายสัญญาไทย (ป.พ.พ.), ข้อสัญญาไม่เป็นธรรม, เบี้ยปรับ, อายุความ, อากรแสตมป์, ภาษีหัก ณ ที่จ่าย, ระบบ GREEN/YELLOW/RED, แนวทาง redline |
| `pdpa-compliance` | `SKILL.md` | PDPA เต็มรูปแบบ — ฐานทางกฎหมาย 6 ฐาน, ข้อมูลอ่อนไหว, สิทธิ์เจ้าของข้อมูล 7 สิทธิ์, DPA checklist, แจ้งเหตุละเมิด, บทลงโทษ, เปรียบเทียบ GDPR/CCPA |
| `thai-labor-law` | `SKILL.md` | แรงงานไทย — สัญญาจ้าง, ชั่วโมงทำงาน, วันหยุด, ค่าล่วงเวลา, เลิกจ้าง, ค่าชดเชย 6 ขั้น, ม.119, ประกันสังคม, checklist จ้างงาน |
| `thai-legal-risk-assessment` | `SKILL.md` | Risk matrix 5x5 (severity x likelihood), GREEN/YELLOW/ORANGE/RED, แนวทางดำเนินการแต่ละระดับ, รูปแบบบันทึกประเมิน, เกณฑ์จ้างที่ปรึกษาภายนอก |

### 3.3 Playbook Configuration

`legal.local.md` — ไฟล์ที่ผู้ใช้/องค์กรปรับแต่งจุดยืน

**เนื้อหา:**
- Contract Review Positions (ข้อจำกัดความรับผิด, การชดใช้, IP, คุ้มครองข้อมูล, ระยะเวลา, กฎหมายที่ใช้บังคับ)
- NDA Defaults
- PDPA Compliance Defaults
- Response Templates path

**วิธีทำงาน:** เมื่อผู้ใช้เรียก `/review-contract` หรือ `/triage-nda` AI จะอ่าน playbook นี้เป็น baseline ในการรีวิว ถ้าไม่มี playbook จะใช้มาตรฐานกฎหมายไทยทั่วไป

### 3.4 MCP Config

Config ที่ชี้ไป remote MCP server

**Claude Code** (`.mcp.json`):
```json
{
  "mcpServers": {
    "legal-th": {
      "type": "http",
      "url": "https://api.legal-th.example.com/mcp"
    }
  }
}
```

**OpenCode** (`opencode.json`):
```json
{
  "mcp": {
    "legal-th": {
      "type": "remote",
      "url": "https://api.legal-th.example.com/mcp"
    }
  }
}
```

### 3.5 Connectors

`CONNECTORS.md` — แผนผัง tool categories ที่ plugin ใช้

ใช้ `~~category` placeholder ใน commands เพื่อให้ tool-agnostic:
- `~~legal-th` → Legal-TH MCP Server
- `~~chat` → Slack / Teams / LINE
- `~~cloud storage` → Box / Google Drive / SharePoint
- `~~email` → Gmail / Microsoft 365
- `~~calendar` → Google Calendar / Microsoft 365
- `~~e-signature` → DocuSign / Adobe Sign

---

## 4. Command Format Specification

### 4.1 Frontmatter

ทุก command file ต้องมี YAML frontmatter:

```yaml
---
description: คำอธิบายสั้นๆ ของ command
argument-hint: "<คำแนะนำ argument>"
---
```

### 4.2 โครงสร้างเนื้อหา

```markdown
# /command-name -- ชื่อภาษาไทย

> หากเห็น placeholder ที่ไม่คุ้นเคย ดู [CONNECTORS.md](../CONNECTORS.md)

[คำอธิบาย]

**สำคัญ**: [disclaimer]

## Invocation
[วิธีเรียกใช้]

## Workflow
### ขั้นตอน 1: ...
### ขั้นตอน 2: ...

## Output Format
[template ของ output]

## หมายเหตุ
[ข้อควรระวัง]
```

### 4.3 หลักการออกแบบ Command

1. **Step-by-step workflow** — AI ทำตามทีละขั้นตอน ไม่กระโดดข้าม
2. **ถามก่อนทำ** — ถ้าข้อมูลไม่พอ ให้ถามผู้ใช้ ไม่เดา
3. **Output ชัดเจน** — มี template ที่เป็นมาตรฐาน
4. **GREEN/YELLOW/RED** — ใช้ระบบจัดระดับเดียวกันทุก command
5. **Graceful degradation** — ถ้า MCP server ไม่ได้เชื่อม ก็ใช้ knowledge ของ AI แทน พร้อมแจ้งข้อจำกัด
6. **Disclaimer ทุกครั้ง** — ไม่ใช่คำปรึกษาทางกฎหมาย

---

## 5. Skill Format Specification

### 5.1 Frontmatter

```yaml
---
name: skill-name
description: คำอธิบายที่ AI ใช้ตัดสินใจว่าจะโหลด skill นี้เมื่อไหร่
---
```

### 5.2 หลักการออกแบบ Skill

1. **เนื้อหาถูกต้อง** — อ้างอิงมาตรา/กฎหมายที่ถูกต้อง
2. **ใช้ตาราง** — เปรียบเทียบ ตัวเลข อัตรา ให้ใช้ตาราง
3. **มี checklist** — ให้ AI ใช้ตรวจสอบได้
4. **มีตัวอย่าง** — ตัวอย่างสำหรับแต่ละระดับ GREEN/YELLOW/RED
5. **ไม่ยาวเกินไป** — AI มี context window จำกัด ให้กระชับแต่ครบ
6. **Update ได้** — เมื่อกฎหมายเปลี่ยน ต้อง update skill ด้วย

---

## 6. การทำงานร่วมกับ MCP Server

### 6.1 Dependency Map

```
Command              Skill ที่ใช้                MCP Endpoint ที่เรียก
─────────────────────────────────────────────────────────────────────
/review-contract  →  thai-contract-review      →  (ไม่จำเป็น)
                     thai-legal-risk-assessment
                     pdpa-compliance

/compliance-check →  pdpa-compliance           →  legal_th/search_laws (optional)
                     thai-labor-law

/triage-nda       →  thai-contract-review      →  (ไม่จำเป็น)

/search-laws      →  (ไม่ใช้ skill)             →  legal_th/search_laws (จำเป็น)

/glossary         →  (ไม่ใช้ skill)             →  legal_th/glossary_lookup (จำเป็น)

/brief            →  (ตาม context)              →  legal_th/regulatory_updates (optional)
```

### 6.2 ทำงานได้โดยไม่มี Server (Offline Mode)

| Command | ไม่มี Server | มี Server |
|---------|-------------|-----------|
| `/review-contract` | ✅ ใช้ได้เต็มที่ (AI + playbook + skills) | ✅ + ค้นหากฎหมายเพิ่มเติมได้ |
| `/compliance-check` | ✅ ใช้ได้ (AI + skills) | ✅ + ค้นหากฎหมายที่เกี่ยวข้องได้ |
| `/triage-nda` | ✅ ใช้ได้เต็มที่ | ✅ เหมือนกัน |
| `/search-laws` | ❌ ใช้ไม่ได้ | ✅ ค้นหาจากฐานข้อมูล |
| `/glossary` | ⚠️ ใช้ได้บางส่วน (AI knowledge) | ✅ ค้นหาจาก glossary DB |
| `/brief daily` | ⚠️ ได้เฉพาะ email/calendar | ✅ + ข่าวกฎหมายล่าสุด |

**Plugin ออกแบบให้ใช้ได้ทันทีโดยไม่ต้องรอ server** — commands หลัก 3 ตัวทำงานได้ด้วย AI + skills + playbook

---

## 7. การเพิ่ม Platform ใหม่

เมื่อต้องการ port ไปค่ายใหม่ ทำตามขั้นตอนนี้:

### 7.1 Checklist

```
1. [ ] ศึกษา plugin format ของค่ายนั้น
   - Config file format อะไร?
   - Commands อยู่ที่ไหน?
   - Skills/Knowledge อยู่ที่ไหน?
   - MCP config format อะไร?

2. [ ] สร้าง directory structure
   clients/legal-th-plugin-{platform}/

3. [ ] ปรับ config files
   - Plugin metadata
   - MCP server config
   - Platform-specific settings

4. [ ] คัดลอก commands
   - ย้ายไป path ที่ถูกต้องของค่ายนั้น
   - ปรับ frontmatter ถ้า format ต่างกัน
   - เนื้อหา markdown ไม่ต้องแก้

5. [ ] คัดลอก skills
   - ย้ายไป path ที่ถูกต้อง
   - ปรับ frontmatter ถ้าจำเป็น
   - เนื้อหาไม่ต้องแก้

6. [ ] คัดลอก playbook + connectors
   - legal.local.md (ไม่ต้องแก้)
   - CONNECTORS.md (ไม่ต้องแก้)

7. [ ] ทดสอบ
   - ติดตั้ง plugin บน platform จริง
   - ทดสอบทุก command
   - ทดสอบเชื่อมต่อ MCP server
```

### 7.2 ระยะเวลาประมาณ

| งาน | เวลา |
|-----|------|
| ศึกษา platform format | 1-2 วัน |
| สร้าง config + โครงสร้าง | ครึ่งวัน |
| คัดลอก + ปรับ commands/skills | ครึ่งวัน |
| ทดสอบ | 1 วัน |
| **รวม** | **2-3 วัน ต่อ platform** |

---

## 8. การ Maintain เนื้อหา

### 8.1 เมื่อกฎหมายเปลี่ยน

```
กฎหมายเปลี่ยน
    │
    ├─ update skill ที่เกี่ยวข้อง (เช่น pdpa-compliance/SKILL.md)
    ├─ update playbook defaults ถ้าจำเป็น (legal.local.md)
    ├─ update command ถ้า workflow เปลี่ยน
    │
    └─ deploy ทุก platform พร้อมกัน
       ├─ legal-th-plugin-claude/
       ├─ legal-th-plugin-opencode/
       └─ ... (ค่ายอื่นๆ)
```

### 8.2 Single Source of Truth

**ปัญหา:** เนื้อหาเดียวกันอยู่หลาย directory → ต้อง sync ให้ตรงกัน

**แนวทาง (เลือก 1):**

**Option A: Copy & Sync (ปัจจุบัน)**
- แต่ละ platform มี copy ของตัวเอง
- ใช้ script sync เนื้อหาเมื่อ update
- ง่าย แต่เสี่ยง out-of-sync

**Option B: Shared Source + Build Script (แนะนำ)**
```
clients/
├── shared/                    ← เนื้อหาเขียนที่เดียว
│   ├── commands/*.md
│   ├── skills/*/SKILL.md
│   ├── legal.local.md
│   └── CONNECTORS.md
├── legal-th-plugin-claude/    ← generated จาก shared
├── legal-th-plugin-opencode/  ← generated จาก shared
└── build.sh                   ← script สร้าง plugin แต่ละค่าย
```

**Option C: Symlinks**
- ใช้ symlink ชี้ไปที่ shared
- ง่ายที่สุด แต่บาง platform อาจไม่รองรับ

### 8.3 Versioning

- ใช้ semantic versioning: `MAJOR.MINOR.PATCH`
- **MAJOR**: เปลี่ยน command interface / ลบ command
- **MINOR**: เพิ่ม command ใหม่ / เพิ่มเนื้อหา skill
- **PATCH**: แก้ไขเนื้อหา / update กฎหมาย
- ทุก platform ใช้ version เดียวกัน

---

## 9. ทดสอบ

### 9.1 Test Cases สำหรับทุก Command

#### /review-contract
```
[ ] รีวิวสัญญาจ้างแรงงาน (ภาษาไทย)
[ ] รีวิวสัญญาบริการ (ภาษาอังกฤษ)
[ ] รีวิวสัญญาที่มี playbook config
[ ] รีวิวสัญญาโดยไม่มี playbook (ใช้มาตรฐานทั่วไป)
[ ] สัญญายาว (50+ หน้า)
[ ] ตรวจว่ามี GREEN/YELLOW/RED classification
[ ] ตรวจว่ามี redline suggestions สำหรับ YELLOW/RED
[ ] ตรวจว่าเตือนเรื่องอากรแสตมป์ (ถ้าเกี่ยวข้อง)
```

#### /compliance-check
```
[ ] ตรวจ PDPA — เก็บข้อมูลลูกค้า
[ ] ตรวจแรงงาน — จ้างพนักงานใหม่
[ ] ตรวจผู้บริโภค — ขายสินค้าออนไลน์
[ ] ตรวจว่าระบุกฎหมายที่เกี่ยวข้องได้ถูกต้อง
[ ] ตรวจว่ามีระดับความเสี่ยง
[ ] ตรวจว่ามี action items
```

#### /triage-nda
```
[ ] NDA Mutual มาตรฐาน → GREEN
[ ] NDA ที่มีข้อเบี่ยงเบนเล็กน้อย → YELLOW
[ ] NDA ที่มี non-compete แฝง → RED
[ ] NDA ภาษาอังกฤษ
[ ] NDA ที่ไม่ใช่ NDA จริง → RED + แจ้งเตือน
```

#### /search-laws (ต้องมี MCP server)
```
[ ] ค้นหา "PDPA" → พบ พ.ร.บ.คุ้มครองข้อมูลส่วนบุคคล
[ ] ค้นหา "ค่าชดเชย เลิกจ้าง" → พบ ม.118
[ ] ค้นหาด้วยภาษาอังกฤษ
[ ] ค้นหาแบบมี filter (category, year)
[ ] ไม่พบผลลัพธ์ → แจ้งอย่างเหมาะสม
```

#### /glossary (ต้องมี MCP server)
```
[ ] ค้นหา "นิติกรรม" → ได้คำแปล + คำอธิบาย
[ ] ค้นหา "force majeure" → ได้ "เหตุสุดวิสัย"
[ ] คำที่ไม่พบ → แจ้งอย่างเหมาะสม
```

#### /brief
```
[ ] /brief daily → สรุปรายวัน
[ ] /brief topic PDPA → ค้นคว้าหัวข้อ
[ ] /brief incident ข้อมูลรั่ว → สรุปเหตุการณ์เร่งด่วน
[ ] ทำงานได้แม้ไม่มี connectors ทั้งหมด
```

### 9.2 Cross-Platform Testing

```
สำหรับแต่ละ platform:
[ ] ติดตั้ง plugin สำเร็จ
[ ] ทุก command เรียกได้
[ ] Skills ถูก load เมื่อต้องใช้
[ ] MCP server เชื่อมต่อได้
[ ] Playbook ถูกอ่านเมื่อรีวิวสัญญา
[ ] ทำงานได้เมื่อไม่มี MCP server (offline mode)
```

---

## 10. ไฟล์ที่ส่งมอบ (Deliverables)

### สถานะปัจจุบัน

```
clients/
├── legal-th-client-spec.md              ← เอกสารนี้
│
├── legal-th-plugin-claude/              ← ✅ พร้อม
│   ├── .claude-plugin/plugin.json
│   ├── .mcp.json
│   ├── CONNECTORS.md
│   ├── legal.local.md
│   ├── commands/ (6 commands)
│   └── skills/ (4 skills)
│
└── legal-th-plugin-opencode/            ← ✅ พร้อม
    ├── opencode.json
    ├── AGENTS.md
    ├── CONNECTORS.md
    ├── legal.local.md
    └── .opencode/
        ├── commands/ (6 commands)
        ├── skills/ (4 skills)
        ├── plugins/ (ว่าง — พร้อมเพิ่ม)
        └── tools/ (ว่าง — พร้อมเพิ่ม)
```

### สิ่งที่ต้องทำต่อ

| ลำดับ | งาน | ใครทำ | เมื่อไหร่ |
|-------|-----|-------|---------|
| 1 | ทดสอบ commands ทั้ง 6 บน Claude Code จริง | Client team | สัปดาห์ 1 |
| 2 | ทดสอบ commands ทั้ง 6 บน OpenCode จริง | Client team | สัปดาห์ 1 |
| 3 | ปรับเนื้อหาตาม feedback | Client team | สัปดาห์ 2 |
| 4 | ตั้ง shared source + build script (Option B) | Client team | สัปดาห์ 2 |
| 5 | เชื่อมต่อ MCP server เมื่อ server พร้อม (Phase 1) | Client + Server team | สัปดาห์ 4+ |
| 6 | ศึกษาและ port ไป Copilot / Cursor | Client team | เมื่อพร้อม |

---

## 11. คำถามที่ต้องตัดสินใจ

1. **Shared source strategy** — ใช้ Option A (copy & sync), B (build script), หรือ C (symlinks)?
2. **ลำดับ platform ถัดไป** — Copilot, Cursor, หรือ ChatGPT ก่อน?
3. **Playbook defaults** — ค่า default ใน `legal.local.md` เหมาะสมหรือต้องปรับ? (ต้องให้ทนายตรวจ)
4. **Skills เพิ่มเติม** — ต้องเพิ่ม skill อื่นอีกไหม? (เช่น ทรัพย์สินทางปัญญา, ภาษี, คุ้มครองผู้บริโภค)
5. **ภาษา** — commands/skills เป็นภาษาไทยหมด จะทำ English version ด้วยไหม?

---

## 12. เอกสารที่เกี่ยวข้อง

| เอกสาร | ที่อยู่ | คำอธิบาย |
|--------|--------|---------|
| Concept Document | `../legal-th-plugin-concept-v2.md` | ภาพรวมทั้ง project |
| Server Spec | `../server/legal-th-server-spec.md` | Spec สำหรับทีม backend |
| Claude Plugin | `./legal-th-plugin-claude/` | Plugin สำหรับ Claude Code |
| OpenCode Plugin | `./legal-th-plugin-opencode/` | Plugin สำหรับ OpenCode |
| Anthropic Legal Plugin (ref) | `../knowledge-work-plugins/legal/` | Reference จาก Anthropic |

---

**จัดทำ:** Legal-TH Plugin Team
**วันที่:** 2026-03-13
**Version:** 1.0
**สถานะ:** Draft — รอทีมรับทราบและทดสอบ
