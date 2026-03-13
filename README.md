# Legal-TH Plugin

AI-powered legal productivity plugin สำหรับกฎหมายไทย — รีวิวสัญญา, ตรวจ PDPA compliance, คัดกรอง NDA, ค้นหากฎหมาย

รองรับ **Claude Code** และ **OpenCode** โดยเชื่อมต่อกับ MCP Server สำหรับฐานข้อมูลกฎหมายไทย

> **Disclaimer:** Plugin นี้ช่วยงาน workflow ทางกฎหมาย ไม่ใช่คำปรึกษาทางกฎหมาย ผลวิเคราะห์ต้องได้รับการตรวจสอบจากทนายความที่มีคุณสมบัติ

---

## Features

### Slash Commands

| Command | คำอธิบาย | ต้องมี Server |
|---------|---------|:------------:|
| `/review-contract` | รีวิวสัญญาตาม playbook ขององค์กร (GREEN/YELLOW/RED) | ไม่จำเป็น |
| `/compliance-check` | ตรวจ compliance ตามกฎหมายไทย (PDPA, แรงงาน, แพ่ง ฯลฯ) | ไม่จำเป็น |
| `/triage-nda` | คัดกรอง NDA อย่างรวดเร็ว (GREEN/YELLOW/RED) | ไม่จำเป็น |
| `/search-laws` | ค้นหากฎหมายไทยจากฐานข้อมูล | ต้องมี |
| `/glossary` | ค้นหาศัพท์กฎหมายไทย-อังกฤษ | ต้องมี |
| `/brief` | สรุปรายวัน / ค้นคว้าหัวข้อ / สรุปเหตุการณ์เร่งด่วน | บางส่วน |

### Skills (ความรู้ที่ AI ใช้)

| Skill | เนื้อหา |
|-------|---------|
| `thai-contract-review` | กฎหมายสัญญาไทย, ข้อสัญญาไม่เป็นธรรม, อากรแสตมป์, ภาษีหัก ณ ที่จ่าย, ระบบ GREEN/YELLOW/RED |
| `pdpa-compliance` | PDPA เต็มรูปแบบ — ฐานทางกฎหมาย, สิทธิ์เจ้าของข้อมูล, DPA checklist, แจ้งเหตุละเมิด, บทลงโทษ |
| `thai-labor-law` | แรงงานไทย — สัญญาจ้าง, ค่าล่วงเวลา, เลิกจ้าง, ค่าชดเชย 6 ขั้น, ประกันสังคม |
| `thai-legal-risk-assessment` | Risk matrix 5x5, ระดับ GREEN/YELLOW/ORANGE/RED, แนวทางดำเนินการ, เกณฑ์ escalation |

---

## Architecture

```
                ผู้ใช้ (ทีมกฎหมาย / นักพัฒนา)
                         │
                         ▼
              ┌─────────────────────┐
              │  AI Coding Agent    │
              │  Claude / OpenCode  │
              │                     │
              │  ┌───────────────┐  │
              │  │ Legal-TH      │  │
              │  │ Client Plugin │  │
              │  └───────┬───────┘  │
              └──────────┼──────────┘
                         │ MCP HTTP
                         ▼
              ┌─────────────────────┐
              │  Legal-TH           │
              │  MCP Server (Cloud) │
              │  ฐานข้อมูลกฎหมายไทย  │
              └─────────────────────┘
```

**Client plugin** (repo นี้) = commands + skills + playbook config + MCP config

**MCP Server** (แยก repo) = ฐานข้อมูลกฎหมาย + search + compliance engine

---

## Project Structure

```
legal-th-plugin/
│
├── clients/
│   ├── legal-th-client-spec.md            # Spec สำหรับทีม client
│   │
│   ├── legal-th-plugin-claude/            # Plugin สำหรับ Claude Code
│   │   ├── .claude-plugin/plugin.json
│   │   ├── .mcp.json
│   │   ├── legal.local.md                 # Playbook config
│   │   ├── CONNECTORS.md
│   │   ├── commands/                      # 6 slash commands
│   │   └── skills/                        # 4 skills
│   │
│   └── legal-th-plugin-opencode/          # Plugin สำหรับ OpenCode
│       ├── opencode.json
│       ├── AGENTS.md
│       ├── legal.local.md
│       ├── CONNECTORS.md
│       └── .opencode/
│           ├── commands/                  # 6 slash commands
│           ├── skills/                    # 4 skills
│           ├── plugins/                   # JS/TS hooks (พร้อมเพิ่ม)
│           └── tools/                     # Custom tools (พร้อมเพิ่ม)
│
├── server/
│   └── legal-th-server-spec.md            # Spec สำหรับทีม backend
│
└── legal-th-plugin-concept-v2.md          # Concept document
```

---

## Quick Start

### Claude Code

```bash
# Clone repo
git clone https://github.com/monthop-gmail/legal-th-plugin.git

# Copy plugin ไปยัง project ที่ต้องการ
cp -r legal-th-plugin/clients/legal-th-plugin-claude/ your-project/
```

จากนั้นใน Claude Code:
```
/review-contract สัญญาจ้างงาน.pdf
/compliance-check เราจะเก็บข้อมูลชีวมิติพนักงาน
/triage-nda nda-draft.docx
```

### OpenCode

```bash
# Clone repo
git clone https://github.com/monthop-gmail/legal-th-plugin.git

# Copy plugin ไปยัง project ที่ต้องการ
cp -r legal-th-plugin/clients/legal-th-plugin-opencode/ your-project/
```

จากนั้นใน OpenCode:
```
/review-contract สัญญาบริการ.pdf
/compliance-check ต้องการส่งข้อมูลลูกค้าไปประมวลผลที่ต่างประเทศ
/triage-nda nda-mutual.pdf
```

---

## Playbook Configuration

แก้ไข `legal.local.md` เพื่อกำหนดจุดยืนขององค์กร:

```markdown
## Contract Review Positions

### Limitation of Liability (ข้อจำกัดความรับผิด)
- Standard position: จำกัดร่วมกันทั้งสองฝ่าย ไม่เกิน 12 เดือนของค่าบริการ
- Acceptable range: 6-24 เดือน
- Escalation trigger: ความรับผิดไม่จำกัด

### Governing Law (กฎหมายที่ใช้บังคับ)
- Preferred: กฎหมายไทย ศาลไทย
- Acceptable: อนุญาโตตุลาการในไทย
- Escalation trigger: กฎหมายต่างประเทศ
```

ถ้าไม่มี playbook → plugin จะใช้มาตรฐานกฎหมายไทยทั่วไปเป็น baseline

---

## Offline Mode

Commands 3 ตัวหลักใช้ได้ **ทันทีโดยไม่ต้องมี MCP Server**:

| Command | ไม่มี Server | มี Server |
|---------|:-----------:|:---------:|
| `/review-contract` | ใช้ได้เต็มที่ | + ค้นหากฎหมายเพิ่มได้ |
| `/compliance-check` | ใช้ได้ | + ค้นหากฎหมายเพิ่มได้ |
| `/triage-nda` | ใช้ได้เต็มที่ | เหมือนกัน |
| `/search-laws` | ใช้ไม่ได้ | ใช้ได้ |
| `/glossary` | บางส่วน | ใช้ได้เต็มที่ |
| `/brief` | บางส่วน | + ข่าวกฎหมายล่าสุด |

---

## Supported Platforms

| Platform | Status | หมายเหตุ |
|----------|:------:|---------|
| Claude Code / Cowork | ✅ | พร้อมใช้ |
| OpenCode | ✅ | พร้อมใช้ |
| GitHub Copilot | 🔲 | รอพัฒนา |
| Cursor | 🔲 | รอพัฒนา |
| ChatGPT / Codex | 🔲 | รอพัฒนา |

เนื้อหา commands + skills เหมือนกันทุกค่าย ต่างกันแค่ config format — ประมาณ **2-3 วัน** ต่อ platform ใหม่

---

## Thai Legal Coverage

### กฎหมายที่ครอบคลุม

- **PDPA** (พ.ร.บ.คุ้มครองข้อมูลส่วนบุคคล พ.ศ. 2562) — ฐานทางกฎหมาย, สิทธิ์เจ้าของข้อมูล, DPA, แจ้งเหตุละเมิด
- **พ.ร.บ.คุ้มครองแรงงาน พ.ศ. 2541** — สัญญาจ้าง, ค่าชดเชย, เลิกจ้าง, ประกันสังคม
- **ประมวลกฎหมายแพ่งและพาณิชย์** — สัญญา, นิติกรรม, อายุความ
- **พ.ร.บ.ว่าด้วยข้อสัญญาที่ไม่เป็นธรรม พ.ศ. 2540** — ข้อสัญญาที่ศาลอาจปรับ
- **ประมวลรัษฎากร** — อากรแสตมป์, ภาษีหัก ณ ที่จ่าย

### กฎหมายที่รองรับเพิ่มเติม (ผ่าน MCP Server)

- พ.ร.บ.คุ้มครองผู้บริโภค, พ.ร.บ.ลิขสิทธิ์, พ.ร.บ.สิทธิบัตร, พ.ร.บ.เครื่องหมายการค้า
- พ.ร.บ.ธุรกรรมอิเล็กทรอนิกส์, พ.ร.บ.คอมพิวเตอร์, กฎหมายสิ่งแวดล้อม, กฎหมายภาษี

---

## Documentation

| เอกสาร | คำอธิบาย |
|--------|---------|
| [Concept Document](legal-th-plugin-concept-v2.md) | ภาพรวม project ทั้งหมด |
| [Client Spec](clients/legal-th-client-spec.md) | Spec สำหรับทีมพัฒนา client plugin |
| [Server Spec](server/legal-th-server-spec.md) | Spec สำหรับทีมพัฒนา MCP server |

---

## Roadmap

### Phase 1 — MVP (ปัจจุบัน)
- [x] Client plugin สำหรับ Claude Code
- [x] Client plugin สำหรับ OpenCode
- [x] 6 slash commands
- [x] 4 skills (PDPA, แรงงาน, สัญญา, risk assessment)
- [x] Playbook configuration
- [x] Client spec + Server spec
- [ ] ทดสอบบน Claude Code / OpenCode จริง
- [ ] MCP Server MVP (3 endpoints)

### Phase 2 — Feature Expansion
- [ ] MCP Server: compliance engine + contract generator
- [ ] เพิ่ม skills: ทรัพย์สินทางปัญญา, ภาษี, คุ้มครองผู้บริโภค
- [ ] Port ไป Copilot / Cursor
- [ ] Shared source + build script

### Phase 3 — Production
- [ ] MCP Server: regulatory updates + admin dashboard
- [ ] ขยายฐานข้อมูล 500+ กฎหมาย
- [ ] Security audit
- [ ] SLA 99.5%

---

## Contributing

ยินดีรับ contributions! โดยเฉพาะ:

- **ผู้เชี่ยวชาญกฎหมาย** — ตรวจสอบความถูกต้องของเนื้อหา skills
- **นักพัฒนา** — port plugin ไปค่ายอื่น, พัฒนา MCP server
- **ผู้ใช้** — ทดสอบ commands และแจ้ง feedback

### วิธี Contribute

1. Fork repo
2. สร้าง branch (`git checkout -b feature/my-feature`)
3. Commit (`git commit -m 'Add some feature'`)
4. Push (`git push origin feature/my-feature`)
5. สร้าง Pull Request

---

## License

[MIT](LICENSE)

---

## Team

Built by Legal-TH Plugin Team

**Co-authored with** Claude Opus 4.6
