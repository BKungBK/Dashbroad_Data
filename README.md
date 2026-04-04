# Shared Room — Database Schema

## Identity & Auth

### `tbusers`
บัญชีผู้ใช้ที่สมัครสมาชิก

| Field | Key | คำอธิบาย |
|---|---|---|
| user_id | PK | ID ผู้ใช้ ไม่ซ้ำ auto increment |
| username | | ชื่อผู้ใช้ ไม่ซ้ำกัน |
| email | | อีเมล ไม่ซ้ำกัน |
| email_verified | | ยืนยันอีเมลแล้วหรือยัง |
| password_hash | | รหัสผ่านที่เข้ารหัสด้วย bcrypt |
| is_active | | บัญชีเปิดใช้งานอยู่หรือไม่ |
| created_at | | วันที่สร้างบัญชี |
| updated_at | | วันที่แก้ไขข้อมูลล่าสุด |

---

### `tbidentities`
ตาราง identity กลาง รวม registered user และ guest ไว้ด้วยกัน ทุกตารางอื่นอ้างอิงที่นี่

| Field | Key | คำอธิบาย |
|---|---|---|
| identity_id | PK | ID identity ไม่ซ้ำ |
| identity_type | | ประเภท: `registered` หรือ `guest` |
| user_id | FK → tbusers | มีค่าเฉพาะถ้าเป็น registered |
| guest_session_id | | UUID session สำหรับ guest — มีค่าเฉพาะถ้าเป็น guest |
| display_name | | ชื่อที่แสดงในห้อง |
| email | | อีเมล (NULL ถ้าเป็น guest) |
| ip_address | | IP address ของผู้ใช้ |
| device_fingerprint | | ตัวระบุ browser/อุปกรณ์ |
| resolved_user_id | FK → tbusers | กรณี guest สมัครบัญชีภายหลัง |
| resolved_at | | วันที่ guest แปลงเป็น registered user |
| expires_at | | วันหมดอายุ guest session |
| is_active | | identity นี้ยังใช้งานอยู่หรือไม่ |
| created_at | | วันที่สร้าง identity |
| last_seen_at | | ครั้งล่าสุดที่ใช้งาน |

---

### `tbrefresh_tokens`
เก็บ refresh token สำหรับต่ออายุ session โดยไม่ต้อง login ใหม่

| Field | Key | คำอธิบาย |
|---|---|---|
| token_id | PK | ID token |
| user_id | FK → tbusers | เจ้าของ token |
| token_hash | | ค่า hash ของ token ไม่เก็บ token จริง |
| device_info | | ข้อมูล browser/อุปกรณ์ที่ใช้ |
| ip_address | | IP ที่สร้าง token |
| created_at | | วันที่สร้าง token |
| expires_at | | วันที่ token หมดอายุ |
| revoked_at | | วันที่ยกเลิก token ก่อนหมดอายุ (NULL = ยังใช้ได้) |
| last_used_at | | ครั้งล่าสุดที่ใช้ token นี้ |

---

### `tbverifications`
เก็บรหัสยืนยันอีเมล ใช้ได้ทั้งสมัคร รีเซ็ตรหัสผ่าน และเปลี่ยนอีเมล

| Field | Key | คำอธิบาย |
|---|---|---|
| verification_id | PK | ID รายการยืนยัน |
| identity_id | FK → tbidentities | identity ที่ขอยืนยัน |
| email | | อีเมลที่ส่งรหัสไป |
| purpose | | จุดประสงค์: `registration` / `password_reset` / `email_change` |
| code_hash | | รหัสยืนยันที่เข้ารหัสแล้ว ไม่เก็บ OTP จริง |
| attempt_count | | จำนวนครั้งที่กรอกรหัสผิด |
| max_attempts | | จำนวนครั้งสูงสุดที่อนุญาต ค่าปกติ 5 |
| resend_count | | จำนวนครั้งที่ขอส่งรหัสใหม่ |
| last_sent_at | | ส่งอีเมลล่าสุดเมื่อไหร่ |
| consumed_at | | วันที่ใช้รหัสสำเร็จ (NULL = ยังไม่ได้ใช้) |
| expires_at | | วันที่รหัสหมดอายุ |
| created_at | | วันที่สร้างรายการนี้ |

---

## Rooms & Members

### `tbrooms`
ข้อมูลห้องทำงานร่วมกัน

| Field | Key | คำอธิบาย |
|---|---|---|
| room_id | PK | ID ห้อง |
| room_name | | ชื่อห้อง |
| room_code | | รหัส 8 หลักสำหรับเชิญเข้าห้อง ไม่ซ้ำกัน |
| owner_identity_id | FK → tbidentities | เจ้าของห้อง |
| description | | คำอธิบายห้อง |
| room_password_hash | | รหัสผ่านห้อง (NULL = ห้องสาธารณะ) |
| max_members | | จำนวนสมาชิกสูงสุดในห้อง |
| allow_anonymous | | อนุญาตให้ guest เข้าห้องได้หรือไม่ |
| is_active | | ห้องเปิดใช้งานอยู่หรือไม่ |
| is_deleted | | ห้องถูกลบแล้วหรือไม่ (soft delete) |
| deleted_at | | วันที่ลบห้อง |
| deleted_by_identity_id | FK → tbidentities | คนที่ลบห้อง |
| created_at | | วันที่สร้างห้อง |
| updated_at | | วันที่อัปเดตห้องล่าสุด |

---

### `tbroom_members`
บันทึกว่าใครอยู่ในห้องไหน และมีสิทธิ์อะไร

| Field | Key | คำอธิบาย |
|---|---|---|
| member_id | PK | ID รายการสมาชิก |
| room_id | FK → tbrooms | ห้องที่เป็นสมาชิก |
| identity_id | FK → tbidentities | สมาชิก |
| role | | บทบาทในห้อง: `owner` / `editor` / `viewer` |
| joined_at | | วันที่เข้าร่วมห้อง |
| last_active_at | | ใช้งานในห้องล่าสุดเมื่อไหร่ |

---

## Room Content

### `tbfiles`
เก็บ metadata ของไฟล์ที่อัปโหลดเข้าห้อง ตัวไฟล์จริงอยู่ใน storage

| Field | Key | คำอธิบาย |
|---|---|---|
| file_id | PK | ID ไฟล์ |
| room_id | FK → tbrooms | ห้องที่ไฟล์อยู่ |
| file_name | | ชื่อไฟล์ต้นฉบับ |
| stored_name | | ชื่อที่ใช้เก็บใน storage เป็น UUID ป้องกันชื่อชน |
| file_extension | | นามสกุลไฟล์ |
| file_size | | ขนาดไฟล์ในหน่วย bytes |
| storage_path | | path จริงใน storage |
| mime_type | | ประเภทไฟล์ เช่น image/png, application/pdf |
| file_hash | | SHA-256 hash ของไฟล์ ใช้ตรวจสอบความสมบูรณ์ |
| uploader_identity_id | FK → tbidentities | คนที่อัปโหลด |
| uploader_ip | | IP ของคนที่อัปโหลด |
| is_deleted | | ไฟล์ถูกลบแล้วหรือไม่ (soft delete) |
| deleted_at | | วันที่ลบไฟล์ |
| deleted_by_identity_id | FK → tbidentities | คนที่ลบไฟล์ |
| uploaded_at | | วันที่อัปโหลด |

---

### `tbdownload_logs`
บันทึกทุกครั้งที่มีการดาวน์โหลดไฟล์

| Field | Key | คำอธิบาย |
|---|---|---|
| download_id | PK | ID log การดาวน์โหลด |
| file_id | FK → tbfiles | ไฟล์ที่ถูกดาวน์โหลด |
| identity_id | FK → tbidentities | คนที่ดาวน์โหลด (NULL ถ้าระบุไม่ได้) |
| session_id | | session ID สำรอง กรณีระบุ identity ไม่ได้ |
| ip_address | | IP ของผู้ดาวน์โหลด |
| user_agent | | ข้อมูล browser ที่ใช้ดาวน์โหลด |
| downloaded_at | | วันเวลาที่ดาวน์โหลด |

---

### `tbnotes`
โน้ตประจำห้อง แต่ละห้องมีโน้ตได้หนึ่งชิ้น

| Field | Key | คำอธิบาย |
|---|---|---|
| note_id | PK | ID โน้ต |
| room_id | FK → tbrooms | ห้องที่โน้ตอยู่ (unique ต่อห้อง) |
| content | | เนื้อหาโน้ตปัจจุบัน |
| version | | ตัวนับ version เพิ่มทุกครั้งที่แก้ไข ใช้ป้องกัน conflict |
| last_editor_identity_id | FK → tbidentities | คนที่แก้ไขล่าสุด |
| last_edited_at | | วันเวลาที่แก้ไขล่าสุด |
| created_at | | วันที่สร้างโน้ต |

---

### `tbnote_history`
เก็บ snapshot โน้ตทุกครั้งที่มีการแก้ไข สำหรับดูประวัติหรือ revert

| Field | Key | คำอธิบาย |
|---|---|---|
| history_id | PK | ID ประวัติการแก้ไข |
| note_id | FK → tbnotes | โน้ตที่ถูกแก้ไข |
| content_snapshot | | เนื้อหาโน้ต ณ เวลาที่แก้ไข |
| version_at | | version ของโน้ต ณ เวลาที่บันทึก snapshot นี้ |
| editor_identity_id | FK → tbidentities | คนที่แก้ไข |
| editor_display_name | | ชื่อผู้แก้ไข snapshot ณ เวลานั้น ไม่เปลี่ยนตามชื่อปัจจุบัน |
| editor_ip | | IP ของผู้แก้ไข |
| edited_at | | วันเวลาที่แก้ไข |

---

## Audit & Logging

### `tbactivity_logs`
บันทึกทุก action ที่เกิดในห้อง เช่น อัปโหลด ดาวน์โหลด แก้โน้ต

| Field | Key | คำอธิบาย |
|---|---|---|
| log_id | PK | ID log |
| room_id | FK → tbrooms | ห้องที่เกิด action (NULL ถ้านอกห้อง) |
| identity_id | FK → tbidentities | คนที่ทำ action |
| session_id | | session ID สำรอง กรณีระบุ identity ไม่ได้ |
| ip_address | | IP ของคนที่ทำ action |
| action_type | | ประเภท action เช่น upload, download, edit_note |
| target_type | | ประเภทเป้าหมาย: file, note หรือ room |
| target_id | | ID ของเป้าหมาย |
| target_name | | ชื่อเป้าหมาย snapshot ณ เวลานั้น อ่านได้แม้ลบไปแล้ว |
| metadata | | ข้อมูลเพิ่มเติมในรูป JSON ไม่มี schema ตายตัว |
| created_at | | วันเวลาที่เกิด action |

---

### `tbsystem_logs`
บันทึก event ระดับระบบ เช่น login fail, revoke token, error

| Field | Key | คำอธิบาย |
|---|---|---|
| log_id | PK | ID log |
| identity_id | FK → tbidentities | คนที่เกี่ยวข้อง |
| session_id | | session ID สำรอง |
| action_type | | ประเภท event ระดับระบบ |
| detail | | รายละเอียดเพิ่มเติมในรูป JSON |
| target_file_id | FK → tbfiles | ถ้า event เกี่ยวกับไฟล์ |
| target_room_id | FK → tbrooms | ถ้า event เกี่ยวกับห้อง |
| ip_address | | IP ที่เกิด event |
| created_at | | วันเวลาที่เกิด event |
