# Lab 4: Loadable Kernel Module (LKM) บน Linux
---

## วัตถุประสงค์

1. เข้าใจแนวคิด Kernel Module ว่าคืออะไร ทำไมต้องมี
2. เขียน kernel module อย่างง่ายที่พิมพ์ข้อความตอน load/unload ได้
3. ใช้คำสั่ง `insmod`, `rmmod`, `lsmod`, `modinfo` ได้
4. อ่าน kernel log ด้วย `dmesg` ได้
5. เขียน kernel module ที่ดักจับ keyboard event และแสดงผลผ่าน `/proc` ได้

---

## สิ่งที่ต้องเตรียม

- ระบบ Linux (Ubuntu/Debian) — **ไม่แนะนำ WSL** เพราะ WSL ไม่รองรับ `insmod` โดยตรง
- ติดตั้ง kernel headers และ build tools:

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r) -y
```

ตรวจสอบว่า header ติดตั้งสำเร็จ:

```bash
ls /lib/modules/$(uname -r)/build
```

> ถ้าเห็นไฟล์/โฟลเดอร์ออกมาแสดงว่าพร้อมใช้งาน

```bash
mkdir -p ~/lkm_lab && cd ~/lkm_lab
```

---

## ความรู้พื้นฐาน — Kernel Module คืออะไร?

```
┌─────────────────────────────────────┐
│           User Space                │
│  ┌──────────┐  ┌──────────────┐    │
│  │  app.out  │  │   shell      │    │
│  └────┬─────┘  └──────┬───────┘    │
│       │               │            │
├───────┼───────────────┼────────────┤  ← system call boundary
│       ▼               ▼            │
│           Kernel Space              │
│  ┌──────────────────────────────┐  │
│  │         Linux Kernel          │  │
│  │                               │  │
│  │  ┌─────────┐  ┌───────────┐  │  │
│  │  │ module A │  │ module B  │  │  │
│  │  │ (loaded) │  │ (loaded)  │  │  │
│  │  └─────────┘  └───────────┘  │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Loadable Kernel Module (LKM)** คือโค้ดที่สามารถ **โหลดเข้า** และ **ถอดออก** จาก kernel ได้ตอนระบบรันอยู่ โดยไม่ต้อง reboot

ตัวอย่างของ kernel module ที่ใช้ในชีวิตจริง:
- **Device drivers** — เช่น driver ของ USB, เสียง, กราฟิก
- **Filesystem** — เช่น `ext4`, `ntfs`
- **Network protocols** — เช่น `iptable_filter`

> **ทำไมต้องเป็น module?** ถ้ารวมทุกอย่างไว้ใน kernel image ตัวเดียว kernel จะใหญ่มาก และทุกครั้งที่ต้องเพิ่ม/แก้ driver ก็ต้อง compile kernel ใหม่ทั้งหมด

---

## Part 1 — Hello Kernel Module

### 1.1 เขียน module แรก

สร้างไฟล์ `hello.c`:

```c
/* hello.c — first kernel module */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Student");
MODULE_DESCRIPTION("Hello World Kernel Module");

static int __init hello_init(void)
{
    printk(KERN_INFO "hello: module loaded\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello: module unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

สังเกตความแตกต่างจากโปรแกรม C ปกติ:

| โปรแกรม C ปกติ | Kernel Module |
|---|---|
| `#include <stdio.h>` | `#include <linux/module.h>` |
| `main()` | `module_init()` / `module_exit()` |
| `printf()` | `printk()` |
| รันใน user space | รันใน kernel space |

### 1.2 เขียน Makefile

สร้างไฟล์ `Makefile`:

```makefile
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

> **หมายเหตุ:** บรรทัดที่ขึ้นต้นด้วย `make` ต้องเป็น **Tab** ไม่ใช่ space

### 1.3 compile module

```bash
make
```

ตรวจสอบไฟล์ที่ได้:

```bash
ls -la hello.ko
file hello.ko
```

> **คำถาม 1.1:** ไฟล์ `.ko` ย่อมาจากอะไร? ต่างจาก `.o` และ `.so` อย่างไร?
>
> ```
> ตอบ:
>
>
> ```

### 1.4 ดูข้อมูล module

```bash
modinfo hello.ko
```

> **คำถาม 1.2:** `modinfo` แสดงข้อมูลอะไรบ้าง? ข้อมูลเหล่านี้มาจากไหนในโค้ด?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 2 — Load / Unload Module

### 2.1 ดู module ที่โหลดอยู่ในระบบ

```bash
lsmod | head -20
```

`lsmod` อ่านข้อมูลจาก `/proc/modules`

### 2.2 โหลด module

```bash
sudo insmod hello.ko
```

ตรวจสอบว่าโหลดสำเร็จ:

```bash
lsmod | grep hello
```

ดู kernel log:

```bash
sudo dmesg | tail -5
```

> **คำถาม 2.1:** ข้อความที่เห็นใน `dmesg` ตรงกับบรรทัดไหนในโค้ด?
>
> ```
> ตอบ:
>
>
> ```

### 2.3 ถอด module

```bash
sudo rmmod hello
```

ดู log อีกครั้ง:

```bash
sudo dmesg | tail -5
```

ตรวจสอบว่าถอดออกแล้ว:

```bash
lsmod | grep hello
```

> **คำถาม 2.2:** ถ้าลอง `insmod hello.ko` ซ้ำโดยไม่ `rmmod` ก่อน จะเกิดอะไรขึ้น? (ลองทำจริง)
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Module Parameters

### 3.1 เพิ่ม parameter ให้ module

สร้างไฟล์ `hello_param.c`:

```c
/* hello_param.c — module with parameters */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/moduleparam.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Student");
MODULE_DESCRIPTION("Module with parameters");

static char *name = "World";
static int count = 1;

module_param(name, charp, 0644);
MODULE_PARM_DESC(name, "Name to greet");

module_param(count, int, 0644);
MODULE_PARM_DESC(count, "Number of greetings");

static int __init hello_param_init(void)
{
    int i;
    for (i = 0; i < count; i++)
        printk(KERN_INFO "hello_param: [%d] Hello, %s!\n", i, name);
    return 0;
}

static void __exit hello_param_exit(void)
{
    printk(KERN_INFO "hello_param: goodbye, %s\n", name);
}

module_init(hello_param_init);
module_exit(hello_param_exit);
```

แก้ `Makefile`:

```makefile
obj-m += hello.o
obj-m += hello_param.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### 3.2 compile และทดสอบ

```bash
make
sudo insmod hello_param.ko name="Linux" count=3
sudo dmesg | tail -5
```

> **คำถาม 3.1:** ลองเปลี่ยน parameter เป็นค่าอื่นแล้วดู `dmesg` — เขียน parameter ที่ใช้และผลลัพธ์ที่ได้
>
> ```
> ตอบ:
>
>
> ```

### 3.3 ดู parameter ขณะ module ทำงาน

```bash
cat /sys/module/hello_param/parameters/name
cat /sys/module/hello_param/parameters/count
```

อย่าลืม unload:

```bash
sudo rmmod hello_param
```

> **คำถาม 3.2:** เลข `0644` ใน `module_param()` หมายถึงอะไร? (Hint: เหมือน file permission)
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — Keystroke Counter

ใน Part นี้เราจะสร้าง module ที่ **ดักจับการกดคีย์บอร์ด** แล้วนับจำนวนปุ่มที่กด นักศึกษาสามารถ `cat /proc/keystroke_count` แล้วเห็นตัวเลขเพิ่มขึ้นทุกครั้งที่กดแป้นพิมพ์

```
┌─────────────┐    กดปุ่ม     ┌──────────────────┐
│  Keyboard   │─────────────▶│   Linux Kernel    │
└─────────────┘              │                    │
                              │  keyboard_notifier │
                              │       │            │
                              │       ▼            │
                              │  ┌────────────┐   │
                              │  │ keycount   │   │  ← module ของเรา
                              │  │ module     │   │
                              │  │ count++ !  │   │
                              │  └────────────┘   │
                              └──────────────────┘
                                       │
                                       ▼
                              /proc/keystroke_count  ← cat แล้วเห็นตัวเลข
```

### 4.1 เขียน Keystroke Counter Module

สร้างไฟล์ `keycount.c`:

```c
/* keycount.c — count keystrokes, show via /proc */
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/keyboard.h>
#include <linux/proc_fs.h>
#include <linux/seq_file.h>
#include <linux/atomic.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Student");
MODULE_DESCRIPTION("Keystroke counter — visible via /proc");

static atomic_t key_count = ATOMIC_INIT(0);

/* ===== Keyboard Notifier ===== */

static int key_notify(struct notifier_block *nb,
                      unsigned long action, void *data)
{
    struct keyboard_notifier_param *param = data;

    /* นับเฉพาะตอนกดลง (key down) ไม่นับตอนปล่อย */
    if (action == KBD_KEYSYM && param->down)
        atomic_inc(&key_count);

    return NOTIFY_OK;
}

static struct notifier_block key_nb = {
    .notifier_call = key_notify,
};

/* ===== /proc/keystroke_count ===== */

static int keycount_show(struct seq_file *m, void *v)
{
    seq_printf(m, "Keystrokes: %d\n", atomic_read(&key_count));
    return 0;
}

static int keycount_open(struct inode *inode, struct file *file)
{
    return single_open(file, keycount_show, NULL);
}

static const struct proc_ops keycount_pops = {
    .proc_open    = keycount_open,
    .proc_read    = seq_read,
    .proc_lseek   = seq_lseek,
    .proc_release = single_release,
};

/* ===== Init / Exit ===== */

static int __init keycount_init(void)
{
    proc_create("keystroke_count", 0444, NULL, &keycount_pops);
    register_keyboard_notifier(&key_nb);
    printk(KERN_INFO "keycount: module loaded — /proc/keystroke_count ready\n");
    return 0;
}

static void __exit keycount_exit(void)
{
    unregister_keyboard_notifier(&key_nb);
    remove_proc_entry("keystroke_count", NULL);
    printk(KERN_INFO "keycount: module unloaded (total keys: %d)\n",
           atomic_read(&key_count));
}

module_init(keycount_init);
module_exit(keycount_exit);
```

### 4.2 อัปเดต Makefile

```makefile
obj-m += hello.o
obj-m += hello_param.o
obj-m += keycount.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

### 4.3 Compile และโหลด

```bash
make
sudo insmod keycount.ko
sudo dmesg | tail -3
```

ตรวจสอบว่า `/proc/keystroke_count` ถูกสร้าง:

```bash
cat /proc/keystroke_count
```

📝 **สังเกต:** ควรเห็น `Keystrokes: 0` (หรือเลขน้อยๆ จากคำสั่ง cat เอง)

### 4.4 ทดลอง — กดแป้นพิมพ์แล้วดูตัวเลข

**ทดลองที่ 1** — กดคีย์บอร์ดใน terminal สักพัก แล้ว:

```bash
cat /proc/keystroke_count
```

📝 **สังเกต:** ตัวเลขเพิ่มขึ้นตามจำนวนปุ่มที่กด!

**ทดลองที่ 2** — เปิด 2 terminal:

**Terminal 1** — watch แบบ real-time:
```bash
watch -n 1 cat /proc/keystroke_count
```

**Terminal 2** — พิมพ์อะไรก็ได้:
```bash
echo "สวัสดี"
ls -la
whoami
```

📝 **สังเกต:** ตัวเลขใน Terminal 1 จะเพิ่มขึ้นเรื่อยๆ ตามที่กดแป้นพิมพ์ใน Terminal 2

**ทดลองที่ 3** — ดู dmesg ตอน unload:

```bash
sudo rmmod keycount
sudo dmesg | tail -3
```

📝 **สังเกต:** จะเห็นข้อความบอกจำนวน key ทั้งหมดที่กดตั้งแต่โหลด module

### คำถาม

> **คำถาม 4.1:** ตอนโหลด module ครั้งแรก `cat /proc/keystroke_count` ได้ตัวเลขเท่าไหร่? ทำไมอาจไม่ใช่ 0?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.2:** เปรียบเทียบ `printk()` กับ `/proc` — ข้อดีของการใช้ `/proc` แสดงข้อมูลคืออะไร? ทำไมจึงดีกว่าดูแค่ `dmesg`?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.3:** ทำไมในโค้ดใช้ `atomic_inc()` / `atomic_read()` แทนที่จะใช้ `count++` ธรรมดา? (Hint: คิดถึง race condition)
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.4:** module นี้ดักจับ keystroke ได้ทุกปุ่มในระบบ — ถ้าเป็น module ที่ไม่น่าไว้ใจ (malicious) จะอันตรายอย่างไร? ทำไม `insmod` จึงต้องใช้ `sudo`?
>
> ```
> ตอบ:
>
>
> ```

---

## สรุปคำสั่งที่ใช้ในแลป

| คำสั่ง | หน้าที่ |
|---|---|
| `make` | compile kernel module จาก Makefile |
| `insmod` | โหลด module เข้า kernel |
| `rmmod` | ถอด module ออกจาก kernel |
| `lsmod` | แสดง module ที่โหลดอยู่ |
| `modinfo` | แสดงข้อมูลของไฟล์ `.ko` |
| `dmesg` | ดู kernel log (ข้อความจาก `printk`) |
| `/proc/` | pseudo-filesystem ที่ kernel ใช้เปิดเผยข้อมูลให้ user space |

---

## สรุป

ในแลปนี้เราได้เรียนรู้:

1. **Kernel module** คือโค้ดที่โหลด/ถอดจาก kernel ได้โดยไม่ต้อง reboot
2. โครงสร้างของ module ประกอบด้วย `init` function, `exit` function, และ metadata
3. ใช้ `printk()` แทน `printf()` เพราะทำงานใน kernel space
4. ใช้ `module_param()` เพื่อรับค่าจากผู้ใช้ตอน `insmod`
5. คำสั่ง `insmod`, `rmmod`, `lsmod`, `modinfo`, `dmesg` เป็นเครื่องมือพื้นฐานในการจัดการ module
6. สร้าง `/proc` entry เพื่อให้ user space อ่านข้อมูลจาก kernel ได้แบบ real-time
7. ใช้ `keyboard_notifier` เพื่อดักจับ keyboard event ใน kernel space
