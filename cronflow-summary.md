## สรุปผลการสกัดไฟล์ CronFlow

ไฟล์ที่อัปโหลดเป็นเอกสารโครงการ **CronFlow — ชุดเครื่องมือจัดตารางงานแบบ Full Stack TS/TSX** ซึ่งเป็นระบบจัดตารางงาน (Job Scheduler) แบบครบชุด เขียนด้วย TypeScript ทั้ง Frontend และ Backend สรุปส่วนหลักดังนี้:

### 🏗 โครงสร้างระบบ
| ส่วน | เทคโนโลยี |
|------|-----------|
| **Backend** | Node.js + Fastify + SQLite (รองรับ PostgreSQL) |
| **Frontend** | React 18 + TSX + Vite |
| **Scheduler** | node-cron + cron-parser + distributed lock |

### 🔑 ส่วนประกอบหลัก
1. **ชนิดข้อมูลกลาง** — กำหนด type `Job`, `JobRun`, `AlertRule` และ enum ต่างๆ (JobStatus, RunStatus, ScheduleType, HandlerType)
2. **ฐานข้อมูล SQLite** — ตาราง `jobs`, `job_runs`, `job_locks`, `settings`, `alerts` พร้อม CRUD ครบ
3. **CRON Parser** — ตรวจสอบและคำนวณเวลารันครั้งต่อไป
4. **Scheduler Engine** — ลูปตรวจงานที่ถึงเวลา, รองรับ handler แบบ shell และ API, มีระบบ lock ป้องกันรันซ้ำ
5. **ระบบแจ้งเตือน** — ส่ง Webhook พร้อม HMAC-SHA256 signature เมื่องานผิดพลาดตามเงื่อนไข
6. **REST API** — `/api/health`, `/api/jobs`, `/api/jobs/:id/toggle`, `/api/runs/recent`
7. **Dashboard** — หน้าเว็บ Dark Theme แสดงสถิติ, รายการงาน, ประวัติการรัน พร้อมปุ่ม toggle สถานะงาน

### 🚀 การใช้งาน
```bash
npm install
npm run dev:server   # Backend (port 8002)
npm run dev:client   # Frontend (port 5173, proxy → backend)
```

สรุปเต็มได้ถูกบันทึกลงไฟล์ `cronflow-summary.md` ให้แล้วครับ 🎯