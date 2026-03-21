# Lab 1: Operating-System Operations
---

## วัตถุประสงค์

1. อธิบายขั้นตอน Bootstrap → Kernel Load ได้จากสิ่งที่เห็นจริง
2. ระบุ System Daemons ที่ทำงานอยู่บนระบบได้
3. แยกแยะ Hardware Interrupt กับ Software Interrupt (Exception / Trap) ได้

---

## Part 1 — Bootstrap Program & Kernel Load

**1.1) เปิดดู Boot Log จริง**

```bash
dmesg | head -50
```

📝 **สังเกต:** บรรทัดแรกๆ จะแสดงข้อมูล kernel version, CPU, memory detection — นี่คือสิ่งที่เกิดขึ้นหลัง bootstrap program โหลด kernel สำเร็จ

**1.2) ดู GRUB bootloader (Bootstrap Program)**

```bash
cat /boot/grub/grub.cfg | grep menuentry
```

📝 **สังเกต:** แต่ละ `menuentry` คือตัวเลือก OS ที่ GRUB (bootstrap program) ให้เลือก — GRUB ทำหน้าที่โหลด kernel ขึ้นมา

**1.3) ดู Kernel ที่กำลังทำงาน**

```bash
uname -r
ls /boot/vmlinuz-*
```

📝 **สังเกต:** ไฟล์ `vmlinuz-*` คือ kernel image ที่ GRUB โหลดเข้า memory

**1.4) Reboot VM ดู boot process**

1. ใน VirtualBox → Settings → System → ติ๊ก **Enable EFI**  (ถ้าต้องการเห็น UEFI)
2. รีบูต VM แล้วกดปุ่มใดก็ได้เพื่อเข้า GRUB menu
3. ใน GRUB กด `e` เพื่อดู boot parameter ของ kernel

### คำถาม

> **คำถาม 1.1:** Bootstrap program ในเครื่องนี้ชื่ออะไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 1.2:** Kernel version ที่ใช้อยู่คืออะไร? (จาก `uname -r`)
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 1.3:** จาก `dmesg` สิ่งแรกที่ kernel ทำหลัง boot คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 2 — System Daemons

### ขั้นตอน

**2.1) แสดง daemon ทั้งหมดที่กำลังทำงาน**

```bash
systemctl list-units --type=service --state=running
```

📝 **สังเกต:** ทุกตัวที่แสดงคือ **system daemon** — service ที่ทำงานอยู่นอก kernel เช่น `sshd`, `cron`, `networkd`

**2.2) ดูรายละเอียด daemon ตัวหนึ่ง**

```bash
systemctl status cron
```

📝 **สังเกต:** ดู PID, สถานะ, และ log ล่าสุดของ daemon

**2.3) ลองหยุดและเริ่ม daemon**

```bash
sudo systemctl stop cron
systemctl status cron        # verify it has stopped
sudo systemctl start cron
systemctl status cron        # verify it is running again
```

📝 **สังเกต:** daemon สามารถ start/stop ได้โดย kernel ไม่ต้อง reboot — เพราะมันทำงาน **นอก kernel**

### คำถาม

> **คำถาม 2.1:** มี service กี่ตัวที่กำลัง running อยู่?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 2.2:** `cron` daemon ทำหน้าที่อะไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 2.3:** ทำไม daemon ถึงเรียกว่า "services provided outside of the kernel"?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Interrupt: Hardware & Software

### 3A: Hardware Interrupt

**3A.1) ดูจำนวน hardware interrupt**

```bash
cat /proc/interrupts
```

📝 **สังเกต:** แต่ละบรรทัดคือ interrupt จาก device ต่างๆ (keyboard, timer, disk, network)

**3A.2) Hardware Interrupt**

เปิด terminal 2 หน้าต่าง:

**Terminal 1** — watch interrupt counter:
```bash
watch -n 1 "cat /proc/interrupts | head -5"
```

**Terminal 2** — สร้าง interrupt โดยกดคีย์บอร์ด, ขยับเมาส์, หรือ:
```bash
ping -c 5 8.8.8.8
```

📝 **สังเกต:** ตัวเลข interrupt count ใน Terminal 1 เพิ่มขึ้นเรื่อยๆ — ทุกครั้งที่ device ส่งสัญญาณมา CPU จะถูก interrupt

### 3B: Software Interrupt — Exception (Error)

**3B.1) สร้าง Division by Zero**

```bash
python3 -c "print(1/0)"
```

📝 **สังเกต:** `ZeroDivisionError` — นี่คือ **exception** ที่เกิดจาก software error

**3B.2) สร้าง Segmentation Fault**

```bash
# create a small C program that accesses invalid memory
echo '#include <stdio.h>
int main() {
    int *p = NULL;
    *p = 42;
    return 0;
}' > /tmp/segfault.c

gcc /tmp/segfault.c -o /tmp/segfault
/tmp/segfault
```

📝 **สังเกต:** `Segmentation fault` — kernel จับได้ว่าโปรแกรมเข้าถึง memory ที่ไม่ได้รับอนุญาต แล้วส่ง **trap** มาหยุดโปรแกรม

**3B.3) ดู kernel log หลัง segfault**

```bash
dmesg | grep -i segfault
```

> ถ้ายังไม่เจอ ให้รัน `/tmp/segfault` อีกครั้งแล้วดูใหม่:
> ```bash
> /tmp/segfault
> dmesg | tail -10
> ```

📝 **สังเกต:** จะเห็นบรรทัดคล้าย `segfault at 0 ip ... sp ... error 6 in segfault` — แสดงว่า kernel เป็นคน handle trap นี้ พร้อมบันทึก address ที่ผิดพลาด

### 3C: Software Interrupt — Infinite Loop & Process Problems

**3C.1) สร้าง infinite loop แล้วจัดการ**

```bash
# run infinite loop in background
yes > /dev/null &
echo "PID = $!"

# check CPU usage
top -n 1 | head -15

# kill process
kill $!
```

### คำถาม

> **คำถาม 3.1:** จาก `/proc/interrupts` — device อะไรสร้าง interrupt มากที่สุด?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.2:** Hardware interrupt กับ Software interrupt ต่างกันอย่างไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.3:** Division by Zero (SIGFPE) กับ Segmentation Fault (SIGSEGV) เกิดจากสาเหตุต่างกันอย่างไร? Kernel จัดการเหมือนหรือต่างกัน?
>
> ```
> ตอบ:
>
>
> ```

---

## สรุปความสัมพันธ์

```
Power On
    │
    ▼
┌─────────────────┐
│ Bootstrap (GRUB) │ ← Part 1: seen in grub.cfg, boot log
└────────┬────────┘
         ▼
┌─────────────────┐
│   Kernel Load    │ ← Part 1: seen in dmesg, uname -r
└────────┬────────┘
         ▼
┌─────────────────┐
│ System Daemons   │ ← Part 2: seen in systemctl
│ (cron, sshd...)  │
└────────┬────────┘
         ▼
┌─────────────────────────────────────┐
│    Kernel: Interrupt Driven          │
│                                      │
│  Hardware Interrupt ← Part 3A       │
│  (keyboard, NIC, timer)              │
│                                      │
│  Software Interrupt ← Part 3B/3C    │
│  ├─ Exception (div/0, segfault)     │
│  └─ Infinite loop, bad process      │
└─────────────────────────────────────┘
```

