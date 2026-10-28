# ระบบจองห้องประชุมภายในองค์กร (Meeting Room Booking System)

เว็บแอปพลิเคชันระดับองค์กรสำหรับบริหารจัดการและจองห้องประชุม พัฒนาด้วย **React + TypeScript + Vite + Tailwind CSS + shadcn/ui + FullCalendar + Recharts + Supabase (Auth, Postgres, Realtime, RLS)**

รองรับภาษาไทยเป็นหลัก แสดงผลตามเขตเวลาไทย (`Asia/Bangkok` UTC+7) พร้อมปีพุทธศักราช (พ.ศ.) มีระบบป้องกันการจองทับซ้อนในระดับฐานข้อมูล ปฏิทินแสดงผล Real-time ศูนย์การแจ้งเตือน และระบบรายงานที่สามารถส่งออกข้อมูลเป็น **CSV (Excel รองรับภาษาไทยสมบูรณ์), XLSX และ PDF**

---

## 🌟 ฟังก์ชันและจุดเด่นของระบบ

### 1. ระบบควบคุมสิทธิ์และความปลอดภัย (Role-Based Access Control & RLS)
- แบ่งผู้ใช้ออกเป็น 3 บทบาทหลัก:
  - **Admin (ผู้ดูแลระบบ):** จัดการห้องประชุมทั้งหมด, เพิ่ม/แก้ไข/ปิดปรับปรุงห้อง, จัดการอุปกรณ์, ดูรายงานและ Audit Logs
  - **Approver (ผู้อนุมัติ):** ตรวจสอบคำขอจองห้อง, อนุมัติ หรือปฏิเสธพร้อมระบุเหตุผล, เรียกดูสถิติ
  - **User (ผู้ใช้ทั่วไป):** ค้นหาห้องว่าง, ยื่นคำขอจอง, ดูปฏิทิน, ตรวจสอบสถานะคำขอ และยกเลิกการจองของตนเอง
- ควบคุมสิทธิ์ทั้งฝั่ง Frontend และ Backend ด้วย **Supabase Row Level Security (RLS)** และ PostgreSQL Stored Procedures
- มีแถบสลับบทบาท (Role Switcher) ด้านบนของหน้าจอ ช่วยให้สามารถทดสอบมุมมองทั้ง 3 บทบาทได้ทันที

### 2. แดชบอร์ดแบบ Real-time (Enterprise Dashboard)
- แสดงข้อมูล KPI สรุปสถานะ:
  - จำนวนห้องประชุมทั้งหมด
  - จำนวนห้องว่างในขณะนี้
  - จำนวนห้องที่กำลังใช้งาน
  - จำนวนการจองวันนี้
  - จำนวนคำขอรออนุมัติ
  - จำนวนการจองที่ยกเลิก
- กราฟแนวโน้มการจอง: สลับมุมมอง รายวัน, รายสัปดาห์ และรายเดือน (Recharts)
- กราฟอัตราการใช้งานของแต่ละห้อง (Room Utilization Rate)
- ตารางรายการจองล่าสุด พร้อมปุ่มอนุมัติ/ปฏิเสธด่วนสำหรับ Approver
- บันทึกกิจกรรมระบบล่าสุด (Audit Logs)

### 3. ระบบจองห้องประชุมและป้องกันการจองทับซ้อน (Anti-Overlap Engine)
- ตรวจสอบความถูกต้องด้วย **React Hook Form + Zod**
- ระบุวัน, เวลาเริ่มต้น, เวลาสิ้นสุด, จำนวนผู้เข้าร่วม, รายชื่อผู้เข้าร่วม, เบอร์ติดต่อ, อุปกรณ์ที่ต้องการ
- **กลไกป้องกันการชนกัน (Concurrency & Race Condition Guard):**
  - ฝั่งฐานข้อมูล: ใช้ **PostgreSQL Exclusion Constraint** ผ่านส่วนขยาย `btree_gist`
    ```sql
    CONSTRAINT no_overlapping_bookings
    EXCLUDE USING gist (
        room_id WITH =,
        tstzrange(start_time, end_time) WITH &&
    ) WHERE (status IN ('pending', 'approved'))
    ```
  - ทั้งสถานะ **Pending (รออนุมัติ)** และ **Approved (อนุมัติแล้ว)** จะถูกกันเวลาไว้ทันที ทำให้ผู้ใช้อื่นไม่สามารถจองช่วงเวลาเดียวกันได้
  - สโตว์โพรซีเยอร์ `create_booking_atomic` ใช้ `SELECT ... FOR UPDATE` เพื่อล็อกแถวป้องกันคลิกพร้อมกันในระดับมิลลิวินาที
  - ตรวจสอบความจุห้อง (ไม่อนุญาตให้ผู้เข้าร่วมเกินความจุ)
  - ไม่อนุญาตให้จองเวลาย้อนหลัง หรือเวลาสิ้นสุดน้อยกว่าเวลาเริ่มต้น

### 4. ปฏิทินการจอง (FullCalendar Integration)
- มุมมอง รายเดือน (Month), รายสัปดาห์ (Week), และรายวัน (Day)
- ตัวกรองแยกตามห้องประชุม และสถานะการจอง
- แยกสีตามสถานะอย่างชัดเจน (เขียว = อนุมัติแล้ว, ส้ม = รออนุมัติ, น้ำเงิน = เสร็จสิ้น, แดง = ไม่อนุมัติ, เทา = ยกเลิก)
- คลิกที่รายการเพื่อดูรายละเอียดแบบ Modal
- คลิกที่ช่วงเวลาว่างเพื่อเริ่มเปิดฟอร์มจองโดยระบุวันเวลาให้อัตโนมัติ

### 5. ศูนย์การแจ้งเตือน Real-time (Notification Center)
- ไอคอนกระดิ่งพร้อม Badge แสดงจำนวนการแจ้งเตือนที่ยังไม่อ่าน
- แจ้งเตือนเมื่อ: มีคำขอจองใหม่, ได้รับการอนุมัติ, ถูกปฏิเสธ (พร้อมแสดงเหตุผล), ยกเลิกการจอง, ใกล้ถึงเวลาเริ่มประชุม (ก่อน 30 นาที)
- ปุ่มทำเครื่องหมายว่าอ่านแล้วรายรายการ และปุ่มอ่านทั้งหมด
- ซิงก์ข้อมูลแบบ Real-time ผ่าน WebSocket Channel ของ Supabase

### 6. ระบบรายงานและส่งออกข้อมูล (Reports & Export)
- กรองตามช่วงวันที่, ห้องประชุม, ผู้จอง, และสถานะการจอง
- สรุปชั่วโมงการใช้งานรวม และอัตราการจัดสรรพื้นที่ (Utilization %)
- ส่งออกข้อมูลได้ 3 รูปแบบ:
  - **CSV (Excel):** มีการแทรก UTF-8 BOM ทำให้เปิดใน Microsoft Excel แล้วภาษาไทยไม่เป็นภาษาต่างดาว
  - **XLSX:** รูปแบบไฟล์ Excel มาตรฐาน
  - **PDF:** จัดรูปแบบเป็นตารางสรุปรายงานพร้อมหัวเอกสาร

---

## 🛠️ เทคโนโลยีที่ใช้ (Tech Stack)

- **Frontend Framework:** React 19 + TypeScript + Vite 8
- **Styling & UI:** Tailwind CSS v4 + ออกแบบตามมาตรฐาน shadcn/ui
- **Icons:** Lucide React
- **Calendar Engine:** FullCalendar (@fullcalendar/react, @fullcalendar/daygrid, @fullcalendar/timegrid, @fullcalendar/interaction)
- **Charts:** Recharts
- **Form & Validation:** React Hook Form + Zod
- **Backend & Database:** Supabase (PostgreSQL 15+, Supabase Auth, Realtime)
- **Exporting Tools:** SheetJS (xlsx), jsPDF, jsPDF-AutoTable
- **Testing:** Vitest

---

## 📁 โครงสร้างโปรเจกต์ (Project Structure)

```
meeting-room-booking/
├── src/
│   ├── types/
│   │   └── database.types.ts       # Types สำหรับ Profiles, Rooms, Bookings, Notifications, Audit
│   ├── lib/
│   │   ├── supabase.ts             # การเชื่อมต่อ Supabase Client & Health Check
│   │   ├── mockData.ts             # ข้อมูล Seed เริ่มต้นสำหรับโหมด Standalone / Offline
│   │   ├── storage.ts              # Local Database & Overlap Prevention Logic
│   │   └── utils.ts                # การจัดรูปแบบวันเวลาไทย (พ.ศ.) และฟังก์ชัน Export (CSV/XLSX/PDF)
│   ├── context/
│   │   ├── AuthContext.tsx         # Supabase Auth, Session, และ Role Switcher
│   │   └── BookingContext.tsx      # Realtime State, CRUD การจอง, ห้อง, แจ้งเตือน, Audit
│   ├── components/
│   │   ├── common/
│   │   │   ├── Navbar.tsx          # แถบเมนูด้านบน, นาฬิกาไทย, สถานะ DB, โปรไฟล์
│   │   │   ├── Sidebar.tsx         # เมนูด้านข้างพร้อม Badge นับจำนวนรออนุมัติ
│   │   │   ├── NotificationDropdown.tsx # ศูนย์แจ้งเตือน Real-time
│   │   │   ├── StatusBadge.tsx     # ป้ายสถานะภาษาไทยมาตรฐาน
│   │   │   ├── Modal.tsx           # หน้าต่าง Modal มาตรฐาน
│   │   │   └── SupabaseSetupModal.tsx # หน้าต่างแนะนำการตั้งค่าฐานข้อมูล
│   │   ├── dashboard/
│   │   │   ├── DashboardView.tsx   # รวมวิดเจ็ตแดชบอร์ด
│   │   │   ├── StatCard.tsx        # การ์ด KPI ตัวเลขสถิติ
│   │   │   ├── BookingChart.tsx    # กราฟแท่งเปรียบเทียบแนวโน้ม (Recharts)
│   │   │   ├── RoomUtilizationChart.tsx # กราฟอัตราใช้งานของแต่ละห้อง
│   │   │   ├── RecentBookingsTable.tsx  # ตารางการจองล่าสุดพร้อมปุ่มอนุมัติด่วน
│   │   │   └── RecentActivityList.tsx   # บันทึกกิจกรรม Audit Log
│   │   ├── calendar/
│   │   │   ├── CalendarView.tsx    # ปฏิทิน FullCalendar ภาษาไทย
│   │   │   └── BookingDetailModal.tsx # รายละเอียดการจอง, ปุ่มอนุมัติ/ปฏิเสธ/ยกเลิก
│   │   ├── booking/
│   │   │   ├── BookingFormModal.tsx # ฟอร์มจองห้อง พร้อมเช็กการชนกันและอุปกรณ์
│   │   │   └── MyBookingsList.tsx  # หน้าประวัติการจองของผู้ใช้
│   │   ├── approvals/
│   │   │   └── ApprovalsList.tsx   # หน้าสำหรับ Approver พิจารณาคำขอ
│   │   ├── rooms/
│   │   │   ├── RoomList.tsx        # จัดการห้องประชุม, กรองความจุ, เปิด/ปิดปรับปรุง
│   │   │   └── RoomFormModal.tsx   # เพิ่ม/แก้ไขห้องประชุมและอุปกรณ์
│   │   ├── reports/
│   │   │   └── ReportsView.tsx     # หน้ารายงาน, กรองข้อมูล, ส่งออก CSV/Excel/PDF
│   │   └── audit/
│   │       └── AuditLogsView.tsx   # ตารางประวัติกิจกรรมและความปลอดภัย
│   ├── test/
│   │   └── bookingValidation.test.ts # ชุดทดสอบ Automated Test ด้วย Vitest
│   ├── App.tsx                     # Main Layout & Tab Routing
│   └── main.tsx                    # Entry Point
├── supabase/
│   ├── migrations/
│   │   └── 20261028_meeting_room_schema.sql # SQL Migration สมบูรณ์ (ตาราง, RLS, Trigger, Lock)
│   └── seed.sql                    # ข้อมูลห้องประชุมและอุปกรณ์ตัวอย่าง
├── .env.example                    # ตัวอย่างการตั้งค่า Environment Variables
└── package.json
```

---

## 🚀 วิธีติดตั้งและรันโปรเจกต์ (Installation & Quick Start)

### 1. ความต้องการของระบบ (Prerequisites)
- Node.js version 18.0 ขึ้นไป (แนะนำ LTS)
- npm หรือ pnpm

### 2. ขั้นตอนการติดตั้ง
```bash
# 1. เข้าสู่โฟลเดอร์โปรเจกต์
cd meeting-room-booking

# 2. ติดตั้ง Dependencies
npm install --legacy-peer-deps

# 3. รัน Development Server
npm run dev
```

เปิดเบราว์เซอร์ไปที่: `http://localhost:5173`

> 💡 **หมายเหตุ:** ระบบมีฐานข้อมูลจำลองและข้อมูล Seed ตั้งต้นมาให้พร้อมใช้งานทันที ทำให้สามารถเปิดทดลองฟังก์ชันได้ครบ 100% แม้ยังไม่ได้ใส่ Supabase API Keys และเมื่อใส่ Key ในไฟล์ `.env.local` ระบบจะเชื่อมต่อไปยัง Live PostgreSQL อัตโนมัติ

---

## 🗄️ การตั้งค่า Supabase และการสร้างฐานข้อมูล (Supabase Setup Guide)

### ขั้นตอนที่ 1: สร้างโปรเจกต์ Supabase
1. เข้าไปที่ [supabase.com](https://supabase.com) แล้วล็อกอินเข้าสู่ระบบ
2. กดปุ่ม **"New Project"** ตั้งชื่อโปรเจกต์และรหัสผ่านฐานข้อมูล
3. เลือก Region ที่ใกล้ประเทศไทย เช่น **Singapore (ap-southeast-1)**

### ขั้นตอนที่ 2: รัน SQL Migration
1. ในหน้า Supabase Dashboard ไปที่เมนูด้านซ้ายเลือก **"SQL Editor"**
2. กด **"New query"**
3. คัดลอกเนื้อหาทั้งหมดจากไฟล์:
   ```
   supabase/migrations/20261028_meeting_room_schema.sql
   ```
   ไปวางใน SQL Editor แล้วกดปุ่ม **"Run"**
   *(สคริปต์นี้จะสร้างตาราง `profiles`, `rooms`, `room_equipment`, `bookings`, `booking_attendees`, `notifications`, `audit_logs` พร้อมทั้ง Index, RLS Policies, Triggers และ Exclusion Constraints ให้ครบถ้วน)*

### ขั้นตอนที่ 3: ลงข้อมูลเริ่มต้น (Seed Data)
1. ใน SQL Editor สร้าง Query ใหม่อีกอัน
2. คัดลอกเนื้อหาจากไฟล์:
   ```
   supabase/seed.sql
   ```
   ไปวางแล้วกดปุ่ม **"Run"** เพื่อสร้างห้องประชุมและอุปกรณ์ตัวอย่าง

### ขั้นตอนที่ 4: ตั้งค่า Environment Variables
1. ใน Supabase Dashboard ไปที่ **Project Settings -> API**
2. คัดลอกค่า **Project URL** และ **Project API Keys (anon public)**
3. สร้างไฟล์ `.env.local` ในโฟลเดอร์รากของโปรเจกต์:
   ```env
   VITE_SUPABASE_URL=https://your-project-id.supabase.co
   VITE_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
   ```
4. รีสตาร์ต dev server:
   ```bash
   npm run dev
   ```
5. แถบสถานะด้านบนจะเปลี่ยนเป็น **🟢 "Supabase Connected"**

---

## 🧪 การทดสอบระบบ (Running Tests)

โปรเจกต์มีชุดทดสอบ Unit & Validation Test ครอบคลุมฟังก์ชันการป้องกันการจองทับซ้อน, การตรวจสอบความถูกต้องของเวลา, และการจัดรูปแบบภาษาไทย

รันการทดสอบด้วยคำสั่ง:
```bash
npx vitest run
```

---

## 📦 การ Build และ Deploy สู่ Production

### Build โปรเจกต์
```bash
npm run build
```
ผลลัพธ์ของไฟล์ Production Bundle จะอยู่ในโฟลเดอร์ `dist/`

### การ Deploy บน Vercel หรือ Netlify
1. เชื่อมต่อ Git Repository กับ Vercel หรือ Netlify
2. กำหนด Build Command: `npm run build`
3. กำหนด Output Directory: `dist`
4. ตั้งค่า Environment Variables ในหน้า Dashboard ของผู้ให้บริการ:
   - `VITE_SUPABASE_URL`
   - `VITE_SUPABASE_ANON_KEY`
5. กด Deploy ระบบจะพร้อมใช้งานแบบสาธารณะทันที
