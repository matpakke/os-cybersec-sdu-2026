# Lab 2: System Calls บน Linux
---

## วัตถุประสงค์

1. เข้าใจว่า `printf()` / `scanf()` เรียก system call อะไรอยู่เบื้องหลัง
2. เปรียบเทียบ library function กับ system call โดยตรง
3. ใช้ file I/O system calls: `open()`, `read()`, `write()`, `close()`
4. ใช้ process system calls: `fork()`, `exec()`, `wait()`, `getpid()`
5. ติดตาม system call ด้วยคำสั่ง `strace`

---

## สิ่งที่ต้องเตรียม

```bash
sudo apt update && sudo apt install gcc strace -y
mkdir -p ~/syscall_lab && cd ~/syscall_lab
```

---

## Part 1 — Simple Program

### 1.1 Hello World

สร้างไฟล์ `hello.c`:

```c
/* hello.c — simplest program */
#include <stdio.h>

int main(void)
{
    printf("Hello, World!\n");
    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -o hello hello.c
./hello
```

ใช้ `strace` ดู:

```bash
strace ./hello
```

ให้ลองสังเกตหา syscall ลองกรองเฉพาะ `write`:

```bash
strace -e trace=write ./hello
```

> **คำถาม 1.1:** เลข `1` ใน `write(1, ...)` คืออะไร?
>
> ```
> ตอบ: write(1, "Hello, World!\n", 14Hello, World!
>)         = 14
>+++ exited with 0 +++
>
>
> ```

### 1.2 printf + scanf

สร้างไฟล์ `greet.c`:

```c
/* greet.c — get name and greet */
#include <stdio.h>

int main(void)
{
    char name[64];

    printf("What is your name? ");
    scanf("%63s", name);
    printf("Hello, %s! Welcome to OS Lab.\n", name);

    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -o greet greet.c
./greet
```

ลองดู syscall:

```bash
strace -e trace=read,write ./greet
```
> **คำถาม 1.2:** `scanf()` เรียก syscall อะไร? fd เป็นเลขอะไร?
>
> ```
> ตอบ:read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000\241\2\0\0\0\0\0"..., 832) = 832
>write(1, "What is your name? ", 19What is your name? )     = 19
>read(0
>, "\n", 1024)                     = 1
>read(0Nattawoot
>, "Nattawoot\n", 1024)            = 10
>write(1, "Hello, Nattawoot! Welcome to OS "..., 37Hello, Nattawoot! Welcome to OS Lab.
>) = 37
>+++ exited with 0 +++
>
> ```


**สิ่งที่ควรสังเกต:**

- `printf()` → เรียก `write(1, ...)` เพื่อพิมพ์ออกจอ
- `scanf()` → เรียก `read(0, ...)` เพื่ออ่านจาก keyboard (fd 0 = stdin)
- ตัวเลข 0 และ 1 คือ **file descriptor** ที่ OS เปิดให้ทุกโปรแกรมอัตโนมัติ

### 1.3 แบบฝึกหัดที่ 1 — เครื่องคิดเลข + strace

**โจทย์:** เขียนโปรแกรม `calc.c` ที่:

1. ถามผู้ใช้ใส่ตัวเลข 2 ตัว (ใช้ `printf` + `scanf`)
2. พิมพ์ผลบวก ลบ คูณ หาร

```c
/* calc.c */
#include <stdio.h>

int main(void)
{
    int a, b;

    printf("Enter two numbers: ");
    scanf("%d %d", &a, &b);

    printf("%d + %d = %d\n", a, b, a + b);
    printf("%d - %d = %d\n", a, b, a - b);
    printf("%d * %d = %d\n", a, b, a * b);

    if (b != 0)
        printf("%d / %d = %d\n", a, b, a / b);
    else
        printf("Cannot divide by zero!\n");

    return 0;
}
```

**ทดสอบ:**

```bash
gcc -Wall -o calc calc.c
./calc
```

รันคำสั่งด้านล่างแล้วตอบคำถาม

```bash
echo "10 3" | strace -e trace=read,write ./calc
```

> **คำถาม 1.2:** `read()` ถูกเรียกกี่ครั้ง?
>
> ```
> ตอบ:2 ครั้ง
>ครั้งแรก: read(3, "\177ELF...", 832) — เป็นการอ่านหัวไฟล์ (Header) ของไฟล์ executable หรือ library (สังเกตจากค่า \177ELF) เพื่อตรวจสอบรูปแบบไฟล์ก่อนรัน
>
>ครั้งที่สอง: read(0, "10 3\n", 4096) — เป็นการอ่านข้อมูลจาก Standard Input (fd 0) ซึ่งก็คือตัวเลข "10 3" ที่ผู้ใช้พิมพ์เข้ามานั่นเอง
>
> ```

> **คำถาม 1.3:** `write()` ถูกเรียกกี่ครั้ง?
>
> ```
> ตอบ:4 ครั้ง
>หากดูตามบรรทัดที่ปรากฏใน log:
>write(1, "Enter two numbers: 10 + 3 = 13\n", 31)
>
>write(1, "10 - 3 = 7\n", 11)
>
>write(1, "10 * 3 = 30\n", 12)
>
>write(1, "10 / 3 = 3\n", 11)
>(หมายเหตุ: เลข 1 ในพารามิเตอร์ตัวแรกคือ Standard Output หรือหน้าจอนั่นเองครับ)
>
> ```

> **คำถาม 1.4:** ทำไม `printf` 4-5 บรรทัดถึงอาจกลายเป็น `write()` แค่ครั้งเดียว? (Hint: buffering)
>
> ```
> ตอบ:เพราะกลไก Standard I/O Buffering ครับ
>
>โดยปกติแล้ว ฟังก์ชันตระกูล printf ในภาษา C จะไม่ส่งข้อมูลไปที่ Kernel (ผ่าน write) ทันทีทุกครั้งที่เรียกใช้งาน แต่มันจะเก็บข้อมูลไว้ใน Buffer (หน่วยความจำชั่วคราว) ในระดับ User-space ก่อน เพื่อประสิทธิภาพสูงสุด
>
>เหตุผลที่อาจกลายเป็น write() ครั้งเดียว:
>
>Full Buffering: หากโปรแกรมตรวจพบว่า Output ไม่ได้ส่งออกหน้าจอโดยตรง (เช่น เขียนลงไฟล์) มันจะรอจนกว่า Buffer จะเต็ม (มักจะเป็น 4KB หรือ 8KB) แล้วค่อยสั่ง write() รวบยอดทีเดียว
>
>Line Buffering: หากส่งออกหน้าจอ (Terminal) ปกติจะ write เมื่อเจอตัวอักษรขึ้นบรรทัดใหม่ (\n) แต่ถ้าเราสั่ง printf ติดๆ กันโดยไม่มี \n หรือมีการตั้งค่า Buffer แบบพิเศษ ข้อมูลทั้งหมดจะถูกมัดรวมกันแล้วส่งผ่าน System Call write() เพียงครั้งเดียวเพื่อ ลด Overhead ในการสลับโหมดระหว่าง User Mode และ Kernel Mode ครับ
>
> ```

---

## Part 2 — File I/O

### 2.1 Library Function vs System Call

```
  User Program
       │
       ▼
  ┌──────────────────┐
  │  Library (libc)  │   ← printf(), fopen(), fread() ...
  │    User Space    │      has buffer, provides convenience
  └──────────────────┘
       │  system call
       ▼
  ┌──────────────────┐
  │     Kernel       │   ← write(), open(), read() ...
  │   Kernel Space   │      accesses hardware directly
  └──────────────────┘
```

| Library Function | System Call ที่เรียกข้างใน |
|:-----------------|:--------------------------|
| `printf()`       | `write()`                 |
| `scanf()`        | `read()`                  |
| `fopen()`        | `open()` / `openat()`     |
| `fread()`        | `read()`                  |
| `fwrite()`       | `write()`                 |
| `fclose()`       | `close()`                 |

## Part 2 — File I/O ด้วย System Call

### 2.1 เปรียบเทียบ: Library vs Syscall

สร้างไฟล์ `write_test.c`:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main(void) {
    /* Library function */
    FILE *fp = fopen("lib.txt", "w");
    fprintf(fp, "from library\n");
    fclose(fp);

    /* System call */
    int fd = open("sys.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644);
    write(fd, "from syscall\n", 13);
    close(fd);
    return 0;
}
```

```bash
gcc -Wall -o write_test write_test.c
strace -e trace=openat,write,close ./write_test
cat lib.txt
cat sys.txt
```

> **คำถาม 2.1:** ทั้ง `fopen` และ `open` สุดท้ายเรียก syscall ตัวเดียวกันหรือไม่? ตัวไหน?
>
> ```
> ตอบ:ใช่ครับ ทั้งคู่เรียกใช้ System Call ตัวเดียวกัน คือ openat
>
>strace -e trace=openat,write,close ./write_test
>cat lib.txt
>cat sys.txt
>openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
>close(3)                                = 0
>openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
>close(3)                                = 0
>openat(AT_FDCWD, "lib.txt", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
>write(3, "from library\n", 13)          = 13
>close(3)                                = 0
>openat(AT_FDCWD, "sys.txt", O_WRONLY|O_CREAT|O_TRUNC, 0644) = 3
>write(3, "from syscall\n", 13)          = 13
>close(3)                                = 0
>+++ exited with 0 +++
>from library
>from syscall
>                                    
> 
>ข้อสังเกตเพิ่มเติมที่น่าสนใจจากภาพ:
>Flag การเปิดไฟล์: สังเกตว่า fopen(..., "w") ถูก Library แปลงค่าเป็น Flag ของระบบอย่าง O_WRONLY|O_CREAT|O_TRUNC ให้โดยอัตโนมัติ ซึ่งตรงกับที่เราเขียนใน open() แบบเป๊ะๆ
>
>File Descriptor (fd): ทั้งสองไฟล์ได้รับเลข 3 เหมือนกัน (แต่คนละช่วงเวลา) เพราะหลังจาก fclose หรือ close ไฟล์แรกไปแล้ว เลข 3 ก็ว่างลง ระบบจึงนำกลับมาใช้ใหม่ให้ไฟล์ที่สอง
>
>Permission: * lib.txt มีโหมดเป็น 0666 (เพราะ fopen มักใช้ค่า default นี้แล้วให้ umask จัดการ)
>
>sys.txt มีโหมดเป็น 0644 ตามที่เรากำหนดไว้ในโค้ด open(..., 0644)
>
>ทำไมต้องมี openat แทนที่จะเป็น open เฉยๆ?
>ใน Linux ยุคใหม่ open() มักจะถูกครอบด้วย openat() ครับ ความเจ๋งของมันคือมันสามารถระบุ "Directory" ที่สัมพันธ์กับไฟล์ได้ (ใช้ AT_FDCWD หมายถึง Current Working Directory) ซึ่งช่วยป้องกันช่องโหว่ทางความปลอดภัยบางประเภทได้ดีกว่า open แบบดั้งเดิมครับ
>
> ```

### 2.2 แบบฝึกหัด — เติม fd ให้ถูก

**โจทย์:** เติม `_____` (2 จุด) ให้โปรแกรม copy ไฟล์ทำงานได้

```c
/* mycp.c */
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
int main(int argc, char *argv[]) {
    if (argc != 3) { printf("Usage: mycp src dst\n"); return 1; }
    int src = open(argv[1], O_RDONLY);
    int dst = open(argv[2], O_WRONLY|O_CREAT|O_TRUNC, 0644);
    char buf[4096];
    ssize_t n;
    while ((n = read(_____, buf, 4096)) > 0)
        write(_____, buf, n);
    close(src);
    close(dst);
    return 0;
}
```

> **Hint:** `read()` อ่านจาก fd ไหน? `write()` เขียนไป fd ไหน?

ทดสอบ:

```bash
gcc -Wall -o mycp mycp.c
echo "Hello OS Lab" > sample.txt
./mycp sample.txt copy.txt
diff sample.txt copy.txt && echo "PASS"
```

> **คำถาม 2.2:** ใช้ strace ดู — `read()` และ `write()` ถูกเรียกอย่างละกี่ครั้ง? ทำไม?
>
> ```
> strace -e trace=openat,read,write,close ./mycp sample.txt copy2.txt
> ```
>
> ```
> ตอบ:
> จุดที่ 1: เติม src เพราะเราต้องการอ่านข้อมูลจาก File Descriptor ของไฟล์ต้นทางที่เปิดไว้
> จุดที่ 2: เติม dst เพราะเราต้องการนำข้อมูลที่อ่านได้ไปเขียนลงใน File Descriptor ของไฟล์ปลายทาง
>
> ```

---

## Part 3 — Process System Calls

### 3.1 ทฤษฎี: fork / exec / wait

```
  Parent (PID=100)
      │
      ├── fork() ────► Child (PID=101)    ← copy ของ parent
      │                    │
      │                    └── exec("ls")  ← แทนที่ด้วยโปรแกรมใหม่
      │
      └── wait() ← รอ child จบ
```

| System Call   | หน้าที่ |
|:--------------|:--------|
| `fork()`      | สร้าง process ลูก (สำเนาของแม่) |
| `exec()`      | แทนที่โปรแกรมปัจจุบันด้วยโปรแกรมใหม่ |
| `wait()`      | รอ process ลูกจบ |

### 3.2 ตัวอย่าง: fork + exec + wait

สร้างไฟล์ `run_cmd.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
int main(void) {
    pid_t pid = fork();
    if (pid == 0) {
        printf("[Child  PID=%d] running ls...\n", getpid());
        execlp("ls", "ls", "-l", NULL);
        perror("exec failed");
        exit(1);
    }
    wait(NULL);
    printf("[Parent PID=%d] child done\n", getpid());
    return 0;
}
```

```bash
gcc -Wall -o run_cmd run_cmd.c
./run_cmd
```

**ลองเปลี่ยนคำสั่ง** — แก้ `execlp("ls", "ls", "-l", NULL)` เป็น:

```c
execlp("date", "date", NULL);           /* แสดงวันที่ */
execlp("whoami", "whoami", NULL);       /* แสดง username */
execlp("xyznotfound", "xyznotfound", NULL);  /* คำสั่งผิด */
```

> **คำถาม 3.1:** ถ้าใส่คำสั่งผิด (xyznotfound) เกิดอะไรขึ้น? ทำไม `perror` ถึงทำงาน?
>
> ```
> ตอบ:เมื่อใส่คำสั่งผิด โปรแกรมจะพิมพ์ข้อความแสดงความผิดพลาดว่า "exec failed: No such file or directory" สาเหตุที่ perror ทำงาน: เนื่องจากฟังก์ชันตระกูล exec (เช่น execlp) จะคืนค่า (Return) กลับมาที่โปรแกรมเดิมก็ต่อเมื่อ "เกิดความผิดพลาด" เท่านั้น (เช่น หาไฟล์ไม่เจอ หรือไม่มีสิทธิ์เข้าถึง) เมื่อมันทำงานไม่สำเร็จ มันจึงไหลลงมาทำงานที่บรรทัดถัดไปซึ่งก็คือ perror เพื่อดึงรหัสความผิดพลาดจากระบบออกมาแสดงผลครับ
>
> ```

> **คำถาม 3.2:** ถ้า `exec` สำเร็จ โค้ดบรรทัดหลัง exec จะถูกรันไหม? ทำไม?
>
> ```
> ตอบ:ไม่ถูกรันครับ
>สาเหตุ: เพราะเมื่อ exec ทำงานสำเร็จ ระบบปฏิบัติการจะทำการ "ทับร่าง" (Overlay) กระบวนการ (Process) เดิมทั้งหมดด้วยโปรแกรมใหม่ที่ระบุไว้
>หน่วยความจำ (Code, Data, Stack) ของโปรแกรมเดิมจะถูกกวาดทิ้งแล้วแทนที่ด้วยโปรแกรมใหม่
>ดังนั้น Child process จะเปลี่ยนไปรันคำสั่งใหม่ (เช่น date หรือ ls) ตั้งแต่จุดเริ่มต้นของโปรแกรมนั้นๆ
>โค้ดที่เหลือในไฟล์เดิมของเราจึงหายไปจากหน่วยความจำและไม่มีโอกาสได้ถูกประมวลผลอีกต่อไปครับ
>
>
> ```

### 3.3 strace ดู process syscalls

```bash
strace -f -e trace=clone,execve,wait4 ./run_cmd
```

> **คำถาม 3.3:** `fork()` จริงๆ แล้วเรียก syscall ชื่ออะไร? (`-f` หมายถึงอะไร?)
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — รวมร่าง: File I/O + Process

### 4.1 ตัวอย่าง: parent_child_write

โปรแกรมนี้ fork child 1 ตัว ทั้ง parent และ child เขียนข้อความลงไฟล์เดียวกัน

สร้างไฟล์ `parent_child_write.c`:

```c
/* parent_child_write.c — parent and child write to same file */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    /* create empty file */
    int fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    close(fd);

    pid_t pid = fork();
    if (pid < 0) { perror("fork"); return 1; }

    if (pid == 0) {
        /* === Child === */
        fd = open("output.txt", O_WRONLY | O_APPEND);
        char msg[] = "Hello from Child!\n";
        write(fd, msg, strlen(msg));
        close(fd);
        printf("[Child  PID=%d] wrote to output.txt\n", getpid());
        exit(0);
    }

    /* === Parent === */
    wait(NULL);  /* wait for child to finish writing */

    fd = open("output.txt", O_WRONLY | O_APPEND);
    char msg[] = "Hello from Parent!\n";
    write(fd, msg, strlen(msg));
    close(fd);
    printf("[Parent PID=%d] wrote to output.txt\n", getpid());

    /* display file contents */
    printf("\n=== output.txt ===\n");
    execlp("cat", "cat", "output.txt", NULL);

    return 0;
}
```

**คอมไพล์และรัน:**

```bash
gcc -Wall -o parent_child_write parent_child_write.c
./parent_child_write
```

**ผลลัพธ์ที่คาดหวัง:**

```
[Child  PID=1235] wrote to output.txt
[Parent PID=1234] wrote to output.txt

=== output.txt ===
Hello from Child!
Hello from Parent!
```

**รันด้วย strace:**

```bash
strace -f -e trace=openat,write,close,clone,wait4 ./parent_child_write
```

---

### 4.2 คำถามจากการรัน (ตอบโดยทดลองจริง)

**คำถามที่ 1 — ลำดับการเขียน**

ลองรันโปรแกรม 5 ครั้ง แล้วดู `output.txt` ทุกครั้ง:

```bash
for i in 1 2 3 4 5; do
    ./parent_child_write > /dev/null 2>&1
    echo "--- Run $i ---"
    cat output.txt
done
```

> **คำถาม 4.1:** Child ขึ้นก่อน Parent ทุกครั้งไหม? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

**คำถามที่ 2 — ถ้าไม่มี wait()**

ลองแก้โค้ด **comment บรรทัด `wait(NULL)` ออก** แล้วรันใหม่ 5 ครั้ง:

```c
    /* wait(NULL); */   /* <-- comment this out */
```

```bash
gcc -Wall -o parent_child_write parent_child_write.c
for i in 1 2 3 4 5; do
    ./parent_child_write > /dev/null 2>&1
    echo "--- Run $i ---"
    cat output.txt
done
```

> **คำถาม 4.2:** ลำดับเปลี่ยนไหม? Parent อาจเขียนก่อน Child ได้ไหม? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

**คำถามที่ 3 — ถ้าไม่ใส่ O_APPEND**

ลองแก้ทั้ง 2 จุดจาก `O_WRONLY | O_APPEND` เป็น `O_WRONLY` (เอา `O_APPEND` ออก):

```c
    fd = open("output.txt", O_WRONLY);   /* no O_APPEND */
```

แล้วรันใหม่:

```bash
gcc -Wall -o parent_child_write parent_child_write.c
./parent_child_write
cat output.txt
```

> **คำถาม 4.3:** เกิดอะไรขึ้นกับเนื้อหาในไฟล์? ข้อความทับกันไหม? ทำไม?
>
> **Hint:** ถ้าไม่มี `O_APPEND` ทั้ง parent และ child จะเริ่มเขียนที่ตำแหน่ง 0 ของไฟล์
>
> ```
> ตอบ:
>
>
> ```

**คำถามที่ 4 — นับ syscall ด้วย strace**

รันโปรแกรม (version ดั้งเดิมที่มี wait และ O_APPEND) ด้วย:

```bash
strace -f -e trace=openat,write,clone,wait4 ./parent_child_write 2>&1 | grep -E "openat|write|clone|wait4"
```

> **คำถาม 4.4:** `clone()` ถูกเรียกกี่ครั้ง? (clone = fork)
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.5:** `openat()` ถูกเรียกกี่ครั้ง? แต่ละครั้งเปิดไฟล์อะไร?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 4.6:** `write()` ที่เขียนลง `output.txt` มีกี่ครั้ง?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — สรุป

### สรุปสิ่งที่เรียน

```
printf("hello")
    │
    └──► write(1, "hello", 5)      ← actual syscall
             │
             └──► Kernel sends data to terminal

scanf("%d", &x)
    │
    └──► read(0, buf, ...)         ← actual syscall
             │
             └──► Kernel reads from keyboard

fopen("f.txt","w")   →   open("f.txt", O_WRONLY|O_CREAT)
fork()               →   clone()  (create new process)
```

### คำถามท้ายแลป

> **คำถาม 5.1:** `printf()` เรียก syscall อะไร? แล้วทำไมบางทีหลาย `printf` ถึงกลายเป็น `write` แค่ครั้งเดียว?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.2:** `open()` คืนค่าอะไร? มันคืออะไรในมุมมองของ kernel?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.3:** `fork()` คืน 0 หมายความว่าอะไร? คืนค่ามากกว่า 0 ล่ะ?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.4:** ทำไม `exec` ถึงไม่ return ถ้าสำเร็จ?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.5:** ถ้า parent ไม่เรียก `wait()` จะเกิดอะไรกับ child ที่จบแล้ว? (Hint: zombie process)
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 5.6:** `O_APPEND` สำคัญอย่างไร เมื่อหลาย process เขียนไฟล์เดียวกัน?
>
> ```
> ตอบ:
>
>
> ```

---

### อ้างอิงเพิ่มเติม

- `man 2 open` / `man 2 read` / `man 2 write` — คู่มือ syscall
- `man 2 fork` / `man 2 execve` / `man 2 wait`
- `man 1 strace` — เครื่องมือดู syscall

---
