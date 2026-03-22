# Lab 0 (Warmup): Linux Basics — Essential Commands, File Descriptor, I/O Redirection & Pipe
---

## วัตถุประสงค์

1. ใช้คำสั่ง Linux พื้นฐานสำหรับจัดการไฟล์ ดู process และตรวจสอบระบบได้
2. อธิบายแนวคิด File Descriptor (fd) และระบุ fd มาตรฐานทั้ง 3 ตัวได้
3. ใช้ Input/Output Redirection (`<`, `>`, `>>`, `2>`) ได้อย่างถูกต้อง
4. ใช้ Pipe (`|`) เพื่อเชื่อมต่อ process ได้ และอธิบายหลักการทำงานได้

---

## Part 1 — คำสั่ง Linux ที่ควรรู้

### 1A: การจัดการไฟล์และ Directory

**1A.1) นำทางใน filesystem**

```bash
pwd                     # แสดง directory ปัจจุบัน
ls                      # แสดงรายการไฟล์
ls -la                  # แสดงทั้งหมด รวม hidden files + รายละเอียด
cd /tmp                 # เปลี่ยน directory
cd ~                    # กลับ home directory
cd -                    # กลับไป directory ก่อนหน้า
```

📝 **สังเกต:** `-la` รวม 2 option → `-l` (long format) + `-a` (all, รวม hidden files ที่ขึ้นต้นด้วย `.`)

**1A.2) สร้าง, คัดลอก, ย้าย, ลบ**

```bash
mkdir -p /tmp/lab0/sub1/sub2     # สร้าง directory ซ้อนกัน (-p = สร้าง parent ให้ด้วย)
touch /tmp/lab0/hello.txt        # สร้างไฟล์เปล่า (หรืออัปเดต timestamp)
cp /tmp/lab0/hello.txt /tmp/lab0/hello2.txt    # คัดลอกไฟล์
mv /tmp/lab0/hello2.txt /tmp/lab0/renamed.txt  # ย้าย/เปลี่ยนชื่อ
rm /tmp/lab0/renamed.txt         # ลบไฟล์
rm -r /tmp/lab0/sub1             # ลบ directory ทั้ง tree
```

📝 **สังเกต:** `rm -r` ลบแบบ recursive — **ระวังให้ดี** เพราะ Linux ไม่มีถังขยะ ลบแล้วกู้คืนยาก

**1A.3) ดูเนื้อหาไฟล์**

```bash
cat /etc/hostname             # แสดงทั้งไฟล์
head -5 /etc/passwd           # แสดง 5 บรรทัดแรก
tail -5 /etc/passwd           # แสดง 5 บรรทัดสุดท้าย
less /etc/passwd              # เปิดดูแบบ scroll ได้ (กด q เพื่อออก)
wc -l /etc/passwd             # นับจำนวนบรรทัด
```

### 1B: ค้นหาไฟล์และข้อความ

**1B.1) ค้นหาไฟล์ด้วย `find`**

```bash
find /etc -name "*.conf" 2>/dev/null | head -10    # หาไฟล์ .conf ใน /etc
find /tmp -type d                                   # หาเฉพาะ directory
find /tmp -name "*.txt" -type f                     # หาเฉพาะไฟล์ .txt
```

📝 **สังเกต:** `2>/dev/null` ซ่อน error (permission denied) — จะเข้าใจลึกขึ้นใน Part ถัดไป

**1B.2) ค้นหาข้อความในไฟล์ด้วย `grep`**

```bash
grep "root" /etc/passwd                   # หาบรรทัดที่มีคำว่า root
grep -n "bash" /etc/passwd                # แสดงเลขบรรทัดด้วย (-n)
grep -r "error" /var/log/ 2>/dev/null | head -5   # ค้นหา recursive ใน directory
grep -i "hello" /tmp/lab0/hello.txt       # ค้นหาแบบ case-insensitive (-i)
grep -c "nologin" /etc/passwd             # นับจำนวนบรรทัดที่ match (-c)
```

📝 **สังเกต:** `grep` เป็นคำสั่งที่ใช้บ่อยมากเมื่อรวมกับ pipe

### 1C: สิทธิ์ไฟล์ (File Permissions)

**1C.1) ดูสิทธิ์ไฟล์**

```bash
ls -l /etc/passwd
ls -l /etc/shadow
```

📝 **สังเกต:** output จะมีรูปแบบ `-rw-r--r--` แบ่งเป็น 3 กลุ่ม:

```
-  rw-  r--  r--
│  │    │    │
│  │    │    └── others (ทุกคน)
│  │    └─────── group
│  └──────────── owner
└─────────────── type (- = file, d = directory)

r = read    (4)
w = write   (2)
x = execute (1)
```

**1C.2) เปลี่ยนสิทธิ์**

```bash
touch /tmp/lab0/script.sh
ls -l /tmp/lab0/script.sh                 # ยังไม่มี x (execute)
chmod +x /tmp/lab0/script.sh              # เพิ่มสิทธิ์ execute
chmod 755 /tmp/lab0/script.sh             # rwxr-xr-x (ใช้เลขฐาน 8)
ls -l /tmp/lab0/script.sh
```

📝 **สังเกต:** `755` = owner(rwx=7), group(r-x=5), others(r-x=5) — นี่คือสิทธิ์ปกติของ script ที่ทุกคนรันได้

### 1D: ดู Process และจัดการ

**1D.1) ดู process ที่กำลังทำงาน**

```bash
ps aux                    # แสดง process ทั้งหมดในระบบ
ps aux | head -10         # ดูแค่ 10 บรรทัดแรก
ps -ef --forest           # แสดงเป็น tree (เห็นความสัมพันธ์ parent-child)
top -n 1                  # แสดง process แบบ real-time (1 รอบ)
```

**1D.2) ค้นหาและ kill process**

```bash
sleep 300 &               # รัน process ใน background
echo "PID = $!"
ps aux | grep sleep       # ค้นหา process
kill $!                   # ส่ง signal SIGTERM (ขอให้หยุด)
```

```bash
sleep 300 &
kill -9 $!                # ส่ง SIGKILL (บังคับหยุดทันที)
```

📝 **สังเกต:** `kill` ส่ง SIGTERM (15) ซึ่ง process สามารถ handle ได้ แต่ `kill -9` ส่ง SIGKILL ที่ process หลบไม่ได้

### 1E: ตรวจสอบระบบ

**1E.1) ข้อมูลระบบ**

```bash
uname -a                  # ข้อมูล kernel
hostname                  # ชื่อเครื่อง
uptime                    # เครื่องเปิดมานานเท่าไหร่
whoami                    # user ปัจจุบัน
id                        # UID, GID ของ user
```

**1E.2) ดู disk และ memory**

```bash
df -h                     # disk usage (-h = human readable)
du -sh /tmp               # ขนาดของ directory
free -h                   # memory usage
```

**1E.3) ดู network เบื้องต้น**

```bash
ip addr                   # แสดง IP address
ping -c 3 8.8.8.8         # ทดสอบ network connectivity
ss -tlnp                  # แสดง port ที่เปิดอยู่ (listening)
```

### 1F: เครื่องมือจัดการข้อความ

**1F.1) `cut` — ตัดคอลัมน์**

```bash
echo "alice:1000:/home/alice" | cut -d: -f1       # ตัดเอาฟิลด์ที่ 1 (alice)
echo "alice:1000:/home/alice" | cut -d: -f1,3     # ฟิลด์ที่ 1 และ 3
head -5 /etc/passwd | cut -d: -f1,6               # username และ home dir
```

**1F.2) `sort` และ `uniq`**

```bash
echo -e "banana\napple\ncherry\napple" > /tmp/fruits.txt
sort /tmp/fruits.txt                    # เรียงลำดับ
sort /tmp/fruits.txt | uniq             # ลบบรรทัดซ้ำ (ต้อง sort ก่อน)
sort /tmp/fruits.txt | uniq -c          # นับจำนวนที่ซ้ำ
```

**1F.3) `tr` — แปลงตัวอักษร**

```bash
echo "Hello World" | tr 'A-Z' 'a-z'       # แปลงเป็นตัวเล็ก
echo "Hello World" | tr 'a-z' 'A-Z'       # แปลงเป็นตัวใหญ่
echo "hello   world" | tr -s ' '          # ลด space ซ้ำเหลือตัวเดียว
```

**1F.4) `awk` — ประมวลผลข้อความขั้นต้น**

```bash
echo "alice 90 85 78" | awk '{print $1, $3}'       # พิมพ์ฟิลด์ที่ 1 และ 3
ps aux | awk '{print $1, $2, $11}' | head -10      # user, PID, command
df -h | awk 'NR>1 {print $1, $5}'                  # filesystem, %used
```

📝 **สังเกต:** `awk` แบ่ง field ด้วย space โดยอัตโนมัติ — `$1` คือฟิลด์แรก, `$2` คือฟิลด์ที่สอง, `NR` คือเลขบรรทัด

### คำถาม

> **คำถาม 1.1:** `ls -la` แสดงข้อมูลอะไรบ้างในแต่ละคอลัมน์?
>
> ```
> ตอบ: 
# รายละเอียดคอลัมน์จากคำสั่ง `ls -la`

คำสั่ง `ls -la` ในระบบปฏิบัติการ Unix/Linux ใช้สำหรับแสดงรายชื่อไฟล์และโฟลเดอร์ทั้งหมด (รวมถึงไฟล์ซ่อน) พร้อมรายละเอียดเชิงลึก โดยแบ่งข้อมูลออกเป็น 7 คอลัมน์หลัก ดังนี้:

| คอลัมน์ | หัวข้อ (Field) | คำอธิบาย (Description) | ตัวอย่างข้อมูล |
| :---: | :--- | :--- | :--- |
| **1** | **Permissions** | สิทธิ์การเข้าถึง (Type, User, Group, Others) | `-rwxr-xr--` |
| **2** | **Links** | จำนวน Hard links ที่เชื่อมโยงมายังไฟล์/โฟลเดอร์ | `1` |
| **3** | **Owner** | ชื่อผู้ใช้งานที่เป็นเจ้าของไฟล์ (Username) | `tae` |
| **4** | **Group** | ชื่อกลุ่มที่ไฟล์สังกัดอยู่ (Group Name) | `staff` |
| **5** | **File Size** | ขนาดของไฟล์ (หน่วยเริ่มต้นเป็น Bytes) | `4096` |
| **6** | **Last Modified** | วันที่และเวลาที่มีการแก้ไขข้อมูลล่าสุด | `Mar 22 11:30` |
| **7** | **File Name** | ชื่อไฟล์หรือโฟลเดอร์ (ไฟล์ซ่อนจะขึ้นต้นด้วย `.` ) | `.bashrc` |

---

## เจาะลึกคอลัมน์ Permissions (คอลัมน์ที่ 1)

ในฐานะ **Cyber Security Analyst** คอลัมน์นี้สำคัญที่สุดในการตรวจสอบความปลอดภัย:

| ตำแหน่ง | กลุ่ม (Set) | ความหมาย | สัญลักษณ์ |
| :--- | :--- | :--- | :--- |
| **ตัวที่ 1** | **Type** | ประเภทของไฟล์ | `d` (Folder), `-` (File), `l` (Link) |
| **ตัวที่ 2-4** | **Owner** | สิทธิ์ของเจ้าของไฟล์ | `r` (Read), `w` (Write), `x` (Execute) |
| **ตัวที่ 5-7** | **Group** | สิทธิ์ของกลุ่ม | `r` (Read), `w` (Write), `x` (Execute) |
| **ตัวที่ 8-10** | **Others** | สิทธิ์ของบุคคลอื่น | `r` (Read), `w` (Write), `x` (Execute) |

---

## ตัวอย่างการอ่าน (Analysis Example)
หากพบผลลัพธ์:  
`-rw-r--r--  1 tae  staff  1024  Mar 22 11:30  config.php`

1. **`-`**: เป็นไฟล์ธรรมดา (Regular file)
2. **`rw-`**: เจ้าของ (`tae`) อ่านและแก้ไขได้ แต่รัน (Execute) ไม่ได้
3. **`r--`**: กลุ่ม (`staff`) อ่านได้อย่างเดียว
4. **`r--`**: คนอื่นๆ (`Others`) อ่านได้อย่างเดียว
5. **`1024`**: ไฟล์มีขนาด 1 KB

>
> ```

> **คำถาม 1.2:** สิทธิ์ `644` หมายความว่าอย่างไร? ใครทำอะไรได้บ้าง?
>
> ```
> ตอบ:
># การวิเคราะห์สิทธิ์ไฟล์ (File Permissions): 644

ในระบบ Linux/Unix ตัวเลข **644** คือการระบุสิทธิ์ในรูปแบบ **Octal (เลขฐานแปด)** ซึ่งเป็นค่ามาตรฐานที่ใช้บ่อยที่สุดสำหรับไฟล์ทั่วไป

---

## 1. โครงสร้างตัวเลข (Numerical Breakdown)

ตัวเลข 3 หลัก จะเรียงลำดับความสำคัญตามกลุ่มผู้ใช้งาน ดังนี้:

| หลักที่ | กลุ่มผู้ใช้งาน (User Class) | ค่าตัวเลข | สิทธิ์ที่ได้รับ |
| :---: | :--- | :---: | :--- |
| **1** | **Owner** (เจ้าของไฟล์) | **6** | อ่าน และ เขียน (`rw-`) |
| **2** | **Group** (สมาชิกในกลุ่ม) | **4** | อ่านอย่างเดียว (`r--`) |
| **3** | **Others** (บุคคลอื่น) | **4** | อ่านอย่างเดียว (`r--`) |

---

## 2. วิธีคำนวณค่าตัวเลข (The Math behind 644)

ค่าตัวเลขแต่ละหลักมาจากการรวมกันของสิทธิ์พื้นฐาน 3 ประเภท:
* **4** = Read (อ่าน)
* **2** = Write (เขียน/แก้ไข)
* **1** = Execute (รันโปรแกรม)
* **0** = No Permission (ไม่มีสิทธิ์)

### สูตรคำนวณ:
* **Owner (6):** มาจาก `4 (Read) + 2 (Write) = 6`
* **Group (4):** มาจาก `4 (Read) + 0 = 4`
* **Others (4):** มาจาก `4 (Read) + 0 = 4`

---

## 3. ใครทำอะไรได้บ้าง? (Who can do what?)

| ผู้ใช้งาน | อ่าน (Read) | เขียน (Write) | รัน (Execute) |
| :--- | :---: | :---: | :---: |
| **เจ้าของ (Owner)** | ✅ | ✅ | ❌ |
| **กลุ่ม (Group)** | ✅ | ❌ | ❌ |
| **คนอื่น (Others)** | ✅ | ❌ | ❌ |

---

## 4. มุมมองด้านความปลอดภัย (Security Perspective)

ในฐานะ **Cyber Security Analyst** สิทธิ์ `644` มีความหมายดังนี้:

* **ความปลอดภัย:** ดีในระดับหนึ่ง เพราะป้องกันไม่ให้คนอื่นมาแก้ไข (Modify) หรือลบไฟล์ของเรา
* **ความเสี่ยง:** หากไฟล์นั้นมีข้อมูลที่เป็นความลับ (เช่น `.env` ที่เก็บ Password) สิทธิ์ `644` จะถือว่า **ไม่ปลอดภัย** เพราะ `Others` ยังสามารถอ่าน (Read) ไฟล์ได้
* **ข้อแนะนำ:** หากเป็นไฟล์ความลับ ควรเปลี่ยนเป็น `600` (เจ้าของอ่าน/เขียนได้คนเดียว) หรือ `640` (เจ้าของและกลุ่มอ่านได้)

---

### คำสั่งที่เกี่ยวข้อง (Useful Commands)
หากต้องการตั้งค่าไฟล์ให้เป็น 644:
```bash
chmod 644 filename.txt
>
> ```

> **คำถาม 1.3:** `kill` กับ `kill -9` ต่างกันอย่างไร? ควรใช้ตัวไหนก่อน? เพราะอะไร?
>
> ```
> ตอบ:
>
# Linux Process Termination Guide

## 1. คำสั่ง `kill` (Default / SIGTERM)
- **คำสั่ง:** `kill <PID>`
- **การทำงาน:** ส่งสัญญาณเบอร์ 15 เพื่อบอกให้โปรแกรม "ปิดตัวอย่างสุภาพ"
- **ข้อดี:** ปลอดภัยต่อข้อมูล, เคลียร์หน่วยความจำเรียบร้อย
- **ข้อเสีย:** หากโปรแกรมค้าง (Not Responding) จะปิดไม่ได้

## 2. คำสั่ง `kill -9` (SIGKILL)
- **คำสั่ง:** `kill -9 <PID>`
- **การทำงาน:** ส่งสัญญาณเบอร์ 9 ไปยัง Kernel ให้ฆ่า Process นั้นทิ้งทันทีโดยไม่ผ่านตัวโปรแกรม
- **ข้อดี:** ปิดได้แน่นอน ไม่ว่าโปรแกรมจะค้างแค่ไหน
- **ข้อเสีย:** เสี่ยงต่อข้อมูลสูญหาย (Data Corruption) และอาจทิ้งไฟล์ขยะไว้ในระบบ

---

### Best Practice (ลำดับการใช้งาน)
1. พยายามปิดด้วยวิธีปกติของโปรแกรมก่อน (เช่น `exit` หรือ `ctrl+c`)
2. ใช้ `kill <PID>` เพื่อขอให้ปิด
3. หากผ่านไปสักพักยังไม่ปิด ให้ใช้ `kill -9 <PID>` เป็นทางเลือกสุดท้าย
>
> ```

> **คำถาม 1.4:** เขียนคำสั่งที่แสดงเฉพาะ username ของ user ทั้งหมดใน `/etc/passwd` แล้วเรียงตามตัวอักษร (ใช้ `cut`, `sort`, `|`)
>
> ```
> ตอบ:
>
>
> ```

---

## ความรู้เบื้องต้น — File Descriptor

ใน Linux ทุกครั้งที่ process เปิดไฟล์ kernel จะให้ตัวเลขกำกับเรียกว่า **File Descriptor (fd)** — เป็นจำนวนเต็มเริ่มจาก 0 ทุก process เกิดมาพร้อมกับ fd มาตรฐาน 3 ตัว:

```
┌──────────┬────┬─────────────────────────┐
│ ชื่อ      │ fd │ หน้าที่                    │
├──────────┼────┼─────────────────────────┤
│ stdin    │  0 │ รับ input (ปกติ = keyboard)│
│ stdout   │  1 │ ส่ง output ปกติ            │
│ stderr   │  2 │ ส่ง error message          │
└──────────┴────┴─────────────────────────┘
```

```
              ┌──────────┐
 keyboard ──▶ │ 0 stdin  │
              │          │
              │ process  │
              │          │
              │ 1 stdout │──▶ terminal
              │ 2 stderr │──▶ terminal
              └──────────┘
```

---

## Part 2 — File Descriptor

**2.1) ดู fd ของ shell ปัจจุบัน**

```bash
ls -l /proc/$$/fd
```

📝 **สังเกต:** จะเห็น fd 0, 1, 2 ชี้ไปที่ terminal device (`/dev/pts/X`) — นี่คือ stdin, stdout, stderr ของ shell

**2.2) ดู fd ของ process ที่กำลังทำงาน**

เปิด 2 terminal:

**Terminal 1** — เปิดไฟล์ค้างไว้:
```bash
sleep 300 &
echo "PID = $!"
```

**Terminal 2** — ดู fd ของ process นั้น (แทน `<PID>` ด้วยตัวเลขจาก Terminal 1):
```bash
ls -l /proc/<PID>/fd
```

📝 **สังเกต:** `sleep` มี fd 0, 1, 2 เหมือน shell เพราะ child process สืบทอด fd จาก parent

**2.3) เขียนโปรแกรม C เปิดไฟล์แล้วดู fd**

สร้างไฟล์ `/tmp/show_fd.c`:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd1 = open("/tmp/aaa.txt", O_CREAT | O_WRONLY, 0644);
    int fd2 = open("/tmp/bbb.txt", O_CREAT | O_WRONLY, 0644);

    printf("fd1 = %d\n", fd1);
    printf("fd2 = %d\n", fd2);

    // แสดง fd ทั้งหมดของ process นี้
    char cmd[64];
    sprintf(cmd, "ls -l /proc/%d/fd", getpid());
    system(cmd);

    close(fd1);
    close(fd2);
    return 0;
}
```

```bash
gcc -Wall /tmp/show_fd.c -o /tmp/show_fd
/tmp/show_fd
```

📝 **สังเกต:** fd ใหม่จะได้เลข 3, 4, ... ต่อจาก stderr (2) เสมอ — kernel จะให้เลขที่ว่างตัวเล็กที่สุด

### คำถาม

> **คำถาม 2.1:** fd 0, 1, 2 ของ shell ชี้ไปที่ device อะไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 2.2:** เมื่อเปิดไฟล์ใหม่ 2 ไฟล์ ได้ fd เลขอะไรบ้าง? เพราะอะไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 2.3:** ถ้าปิด fd 3 แล้วเปิดไฟล์ใหม่อีกตัว จะได้ fd เลขอะไร? เพราะอะไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Output Redirection (`>`, `>>`, `2>`)

**3.1) Redirect stdout ไปไฟล์ (`>`)**

```bash
echo "hello world" > /tmp/out.txt
cat /tmp/out.txt
```

📝 **สังเกต:** `>` เปลี่ยน fd 1 (stdout) จาก terminal ไปเป็นไฟล์ — ข้อความจึงไม่แสดงบนจอ แต่ไปอยู่ในไฟล์แทน

```
              ┌──────────┐
 keyboard ──▶ │ 0 stdin  │
              │          │
              │  echo    │
              │          │
              │ 1 stdout │──▶ /tmp/out.txt  (แทน terminal)
              │ 2 stderr │──▶ terminal
              └──────────┘
```

**3.2) Redirect แบบ append (`>>`)**

```bash
echo "line 1" > /tmp/log.txt
echo "line 2" >> /tmp/log.txt
echo "line 3" >> /tmp/log.txt
cat /tmp/log.txt
```

📝 **สังเกต:** `>` เขียนทับ (overwrite) แต่ `>>` เขียนต่อท้าย (append)

**3.3) Redirect stderr (`2>`)**

```bash
ls /no/such/path
ls /no/such/path 2> /tmp/err.txt
cat /tmp/err.txt
```

📝 **สังเกต:** error message ไม่แสดงบนจอ เพราะถูกส่งไปไฟล์ — `2>` หมายถึง redirect fd 2 (stderr)

**3.4) Redirect ทั้ง stdout และ stderr**

```bash
ls /tmp /no/such/path > /tmp/all.txt 2>&1
cat /tmp/all.txt
```

📝 **สังเกต:** `2>&1` หมายถึง "ส่ง fd 2 ไปที่เดียวกับ fd 1" — ผลลัพธ์ปกติและ error จึงรวมอยู่ในไฟล์เดียวกัน

**3.5) แยก stdout กับ stderr ไปคนละไฟล์**

```bash
ls /tmp /no/such/path > /tmp/good.txt 2> /tmp/bad.txt
echo "=== stdout ==="
cat /tmp/good.txt
echo "=== stderr ==="
cat /tmp/bad.txt
```

### คำถาม

> **คำถาม 3.1:** `>` กับ `>>` ต่างกันอย่างไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.2:** ถ้าต้องการบันทึก error log ของคำสั่ง `gcc` ลงไฟล์ โดยไม่ให้แสดงบนจอ จะเขียนคำสั่งอย่างไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.3:** `2>&1` หมายความว่าอย่างไร? ทำไมต้องเขียน `&` นำหน้า `1`?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — Input Redirection (`<`)

**4.1) ใช้ `<` อ่าน input จากไฟล์แทน keyboard**

```bash
echo "3 + 5" > /tmp/calc.txt
bc < /tmp/calc.txt
```

📝 **สังเกต:** `bc` อ่านจากไฟล์แทน keyboard — `<` เปลี่ยน fd 0 (stdin) จาก terminal เป็นไฟล์

```
              ┌──────────┐
/tmp/calc ──▶ │ 0 stdin  │  (แทน keyboard)
              │          │
              │    bc    │
              │          │
              │ 1 stdout │──▶ terminal
              │ 2 stderr │──▶ terminal
              └──────────┘
```

**4.2) ใช้ input + output redirection พร้อมกัน**

```bash
echo -e "5 * 7\n10 + 20" > /tmp/math.txt
bc < /tmp/math.txt > /tmp/result.txt
cat /tmp/result.txt
```

📝 **สังเกต:** ทั้ง stdin และ stdout ถูก redirect พร้อมกัน — process ไม่ได้ใช้ terminal เลย

**4.3) เขียนโปรแกรม C อ่านจาก stdin**

สร้างไฟล์ `/tmp/upper.c`:

```c
#include <stdio.h>
#include <ctype.h>

int main() {
    int c;
    while ((c = getchar()) != EOF) {
        putchar(toupper(c));
    }
    return 0;
}
```

```bash
gcc -Wall /tmp/upper.c -o /tmp/upper
echo "hello os class" | /tmp/upper
echo "hello os class" > /tmp/input.txt
/tmp/upper < /tmp/input.txt
```

📝 **สังเกต:** `getchar()` อ่านจาก fd 0 — ไม่สนว่า fd 0 ต่อกับ keyboard, ไฟล์, หรือ pipe ก็ได้

### คำถาม

> **คำถาม 4.1:** ทำไมโปรแกรมเดียวกันใช้ได้ทั้ง keyboard, `<`, และ `|` โดยไม่ต้องแก้โค้ด?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.2:** เขียนคำสั่งที่อ่าน input จาก `/tmp/input.txt` แล้วเขียน output ลง `/tmp/output.txt` โดยใช้โปรแกรม `/tmp/upper`
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — Pipe (`|`)

**5.1) Pipe พื้นฐาน**

```bash
ls /usr/bin | head -10
```

📝 **สังเกต:** `|` เชื่อม stdout ของ `ls` เข้ากับ stdin ของ `head` — ไม่ต้องสร้างไฟล์กลาง

```
┌──────────┐  pipe   ┌──────────┐
│    ls    │────────▶│   head   │
│ 1:stdout │  fd[0]  │ 0:stdin  │
└──────────┘         └──────────┘
                          │
                     1:stdout ──▶ terminal
```

**5.2) เปรียบเทียบ pipe กับ redirect ผ่านไฟล์**

วิธีที่ 1 — ใช้ไฟล์กลาง:
```bash
ls /usr/bin > /tmp/files.txt
wc -l < /tmp/files.txt
```

วิธีที่ 2 — ใช้ pipe (ทำสิ่งเดียวกัน แต่ไม่ต้องสร้างไฟล์):
```bash
ls /usr/bin | wc -l
```

📝 **สังเกต:** pipe ทำงานเหมือน "ไฟล์ชั่วคราวใน memory" ที่เชื่อม stdout ของคำสั่งซ้ายเข้ากับ stdin ของคำสั่งขวา

**5.3) Pipe หลายขั้น**

```bash
cat /etc/passwd | cut -d: -f1 | sort | head -10
```

📝 **สังเกต:** สามารถต่อ pipe กี่ขั้นก็ได้ — แต่ละ `|` สร้าง process ใหม่ 1 ตัว

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│   cat    │────▶│   cut    │────▶│   sort   │────▶│   head   │──▶ terminal
└──────────┘     └──────────┘     └──────────┘     └──────────┘
```

**5.4) ดู process ที่ถูกสร้างจาก pipe**

```bash
sleep 60 | sleep 60 | sleep 60 &
ps --forest
```

📝 **สังเกต:** `ps --forest` จะแสดงว่า shell สร้าง process ลูก 3 ตัว — ตัวละ 1 คำสั่งใน pipe chain

หลังดูแล้วให้ kill ทิ้ง:
```bash
kill %1
```

**5.5) เขียนโปรแกรม C ที่ทำงานกับ pipe**

สร้างไฟล์ `/tmp/count_lines.c`:

```c
#include <stdio.h>

int main() {
    int lines = 0;
    int c;
    while ((c = getchar()) != EOF) {
        if (c == '\n') lines++;
    }
    printf("total lines: %d\n", lines);
    return 0;
}
```

```bash
gcc -Wall /tmp/count_lines.c -o /tmp/count_lines
cat /etc/passwd | /tmp/count_lines
ls /usr/bin | /tmp/count_lines
```

📝 **สังเกต:** โปรแกรมที่อ่าน stdin / เขียน stdout สามารถร่วมงานกับคำสั่งอื่นผ่าน pipe ได้ทันที — นี่คือปรัชญา Unix: *"Write programs that do one thing and do it well. Write programs to work together."*

### คำถาม

> **คำถาม 5.1:** `ls /usr/bin | wc -l` สร้าง process กี่ตัว? อะไรบ้าง?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.2:** Pipe ต่างจากการ redirect ผ่านไฟล์กลางอย่างไร? (ข้อดี/ข้อเสีย)
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.3:** เขียนคำสั่งที่ใช้ pipe เพื่อนับจำนวน `.conf` files ใน `/etc/` (ไม่รวม subdirectory)
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.4:** ออกแบบปรัชญา Unix ให้โปรแกรม "ทำสิ่งเดียว แล้วเชื่อมกัน" — ข้อดีของแนวคิดนี้คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 6 — ฝึกรวม (Challenge)

**6.1)** เขียนคำสั่ง **เดียว** (ใช้ pipe) ที่แสดงชื่อ user 5 อันดับแรกที่มี home directory อยู่ใน `/home` จากไฟล์ `/etc/passwd`

> ```
> ตอบ:
>
>
> ```

**6.2)** เขียนคำสั่งที่รัน `find /etc -name "*.conf"` แล้วเก็บผลลัพธ์ใน `/tmp/confs.txt` และเก็บ error (permission denied) ใน `/tmp/confs_err.txt`

> ```
> ตอบ:
>
>
> ```

**6.3)** เขียนโปรแกรม C ที่อ่าน stdin ทีละบรรทัด แล้วพิมพ์เฉพาะบรรทัดที่มีคำว่า `root` — ทดสอบด้วย `cat /etc/passwd | ./program`

> ```
> ตอบ:
>
>
> ```

---

## สรุปความสัมพันธ์

```
┌────────────────────────────────────────────────────────┐
│                     Process                            │
│                                                        │
│   fd 0 (stdin)  ◀── keyboard / file (<) / pipe (|)    │
│   fd 1 (stdout) ──▶ terminal / file (>) / pipe (|)    │
│   fd 2 (stderr) ──▶ terminal / file (2>)              │
│   fd 3, 4, ...  ──▶ open() ได้เพิ่มเอง                  │
│                                                        │
└────────────────────────────────────────────────────────┘

Output Redirection:     cmd > file      (stdout → file, overwrite)
                        cmd >> file     (stdout → file, append)
                        cmd 2> file     (stderr → file)
                        cmd > f 2>&1    (stdout + stderr → file)

Input Redirection:      cmd < file      (stdin ← file)

Pipe:                   cmd1 | cmd2     (cmd1 stdout → cmd2 stdin)
```
