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
>
> ```

> **คำถาม 1.3:** `write()` ถูกเรียกกี่ครั้ง?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 1.4:** ทำไม `printf` 4-5 บรรทัดถึงอาจกลายเป็น `write()` แค่ครั้งเดียว? (Hint: buffering)
>
> ```
> ตอบ:
>
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

### 2.2 ตัวอย่าง: เขียนไฟล์ 2 วิธี เปรียบเทียบกัน

สร้างไฟล์ `compare_write.c`:

```c
/* compare_write.c — compare fwrite vs write */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main(void)
{
    const char *msg = "Hello from OS Lab!\n";

    /* === Library function === */
    FILE *fp = fopen("out_lib.txt", "w");
    if (fp == NULL) { perror("fopen"); return 1; }
    fprintf(fp, "%s", msg);
    fclose(fp);
    printf("[Library]  Write Success\n");

    /* === System call === */
    int fd = open("out_sys.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) { perror("open"); return 1; }
    write(fd, msg, strlen(msg));
    close(fd);
    printf("[Syscall] Write Success\n");

    return 0;
}
```

คอมไพล์และรัน:

```bash
gcc -Wall -o compare_write compare_write.c
./compare_write
cat out_lib.txt
cat out_sys.txt
diff out_lib.txt out_sys.txt && echo "same content !"
```

**ลอง strace เปรียบเทียบ:**

```bash
strace -e trace=openat,write,close ./compare_write
```

> **สังเกต:** ทั้ง `fopen` และ `open` สุดท้ายก็เรียก `openat()` เหมือนกัน

### 2.3 ตัวอย่าง: อ่านไฟล์ด้วย syscall

สร้างไฟล์ `read_file.c`:

```c
/* read_file.c — read file using open + read */
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(void)
{
    char buf[256];

    int fd = open("out_sys.txt", O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    ssize_t n = read(fd, buf, sizeof(buf) - 1);
    if (n < 0) {
        perror("read");
        close(fd);
        return 1;
    }

    buf[n] = '\0';   /* null-terminate string */
    close(fd);

    printf("Read %zd bytes:\n%s", n, buf);
    return 0;
}
```

```bash
gcc -Wall -o read_file read_file.c
./read_file
```

### 2.4 แบบฝึกหัดที่ 2 — mycp: Copy ไฟล์ด้วย syscall

**โจทย์:** เติมโค้ดให้โปรแกรม `mycp.c` ทำงานเหมือน `cp` อย่างง่าย โดยใช้ syscall ที่เรียนมาจากตัวอย่าง 2.2 และ 2.3

```
./mycp <source_file> <dest_file>
```

**โค้ด (เติม `_____` ให้ถูกต้อง — มี 2 จุด):**

```c
/* mycp.c — copy file using syscalls */
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

#define BUF_SIZE 4096

int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src> <dst>\n", argv[0]);
        return 1;
    }

    int src_fd = open(argv[1], O_RDONLY);
    if (src_fd < 0) { perror("open src"); return 1; }

    int dst_fd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (dst_fd < 0) { perror("open dst"); close(src_fd); return 1; }

    char buf[BUF_SIZE];
    ssize_t n;

    while ((n = read(_____, buf, BUF_SIZE)) > 0) {
        write(_____, buf, n);
    }

    if (n < 0) perror("read");

    close(src_fd);
    close(dst_fd);
    return 0;
}
```

> **Hint:** ดูตัวอย่าง 2.3 — `read()` ต้องอ่านจาก fd ไหน? แล้ว `write()` ต้องเขียนไป fd ไหน?

**ทดสอบ:**

```bash
gcc -Wall -o mycp mycp.c
echo "This is a test file with some content." > sample.txt
./mycp sample.txt copy.txt
diff sample.txt copy.txt && echo "PASS: files are identical"
```

**ทดสอบ strace:**

```bash
strace -e trace=openat,read,write,close ./mycp sample.txt copy2.txt
```

> **คำถาม 2.1:** `read()` และ `write()` ถูกเรียกกี่ครั้ง? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 3 — Process System Calls

### 3.1 ทฤษฎี: fork / exec / wait

```
  Parent (PID=100)
      │
      ├── fork() ────► Child (PID=101)    ← copy of parent
      │                    │
      │                    └── exec("ls")  ← replace program with new one
      │
      └── wait() ← wait for child to finish
```

| System Call   | หน้าที่ |
|:--------------|:--------|
| `fork()`      | สร้าง process ลูก (สำเนาของแม่) |
| `getpid()`    | คืน PID ของ process ปัจจุบัน |
| `getppid()`   | คืน PID ของ process แม่ |
| `execlp()`    | แทนที่โปรแกรมปัจจุบันด้วยโปรแกรมใหม่ |
| `wait()`      | รอ process ลูกจบ |

### 3.2 ตัวอย่าง: fork เบื้องต้น

สร้างไฟล์ `fork_basic.c`:

```c
/* fork_basic.c — first fork */
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    printf("Before fork: PID = %d\n", getpid());

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        /* === Child process === */
        printf("  [Child]  PID = %d, Parent PID = %d\n",
               getpid(), getppid());
    } else {
        /* === Parent process === */
        printf("  [Parent] PID = %d, Child PID = %d\n",
               getpid(), pid);
        wait(NULL);   /* wait for child to finish */
        printf("  [Parent] Child Completed\n");
    }

    return 0;
}
```

```bash
gcc -Wall -o fork_basic fork_basic.c
./fork_basic
```

**ผลลัพธ์ตัวอย่าง (PID จะต่างกัน):**

```
Before fork: PID = 5001
  [Parent] PID = 5001, Child PID = 5002
  [Child]  PID = 5002, Parent PID = 5001
  [Parent] Child Completed
```

**ลองรันหลายครั้ง** — ลำดับ [Parent] กับ [Child] อาจสลับกัน ทำไม?

### 3.3 ตัวอย่าง: fork + exec + wait

สร้างไฟล์ `fork_exec.c`:

```c
/* fork_exec.c — create child to run ls command */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void)
{
    printf("Parent PID = %d\n", getpid());

    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        printf("  Child PID = %d -> exec(ls -la)\n", getpid());
        execlp("ls", "ls", "-la", NULL);
        perror("execlp");
        exit(1);
    }

    int status;
    waitpid(pid, &status, 0);

    if (WIFEXITED(status))
        printf("Parent: Child exit code %d\n",
               WEXITSTATUS(status));

    return 0;
}
```

```bash
gcc -Wall -o fork_exec fork_exec.c
./fork_exec
```

**strace ดู process:**

```bash
strace -f -e trace=clone,execve,wait4 ./fork_exec
```

> **หมายเหตุ:** `-f` = ติดตาม child ด้วย, `clone` = syscall จริงที่ `fork()` เรียก

### 3.4 แบบฝึกหัดที่ 3 — fork + exec เปลี่ยนคำสั่ง

ลองแก้ไฟล์ `fork_exec.c` โดยเปลี่ยน `execlp("ls", "ls", "-la", NULL)` เป็นคำสั่งอื่น แล้วรันดู:

**ลองที่ 1:**
```c
execlp("date", "date", NULL);
```

**ลองที่ 2:**
```c
execlp("whoami", "whoami", NULL);
```

**ลองที่ 3:**
```c
execlp("uname", "uname", "-a", NULL);
```

**ลองที่ 4 (ใส่คำสั่งผิด):**
```c
execlp("xyznotfound", "xyznotfound", NULL);
```

> **คำถาม 3.1:** ลองที่ 4 เกิดอะไรขึ้น? ทำไม `perror` ถึงพิมพ์ออกมาได้?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.2:** ถ้า `execlp` สำเร็จ โค้ดบรรทัดหลัง `execlp` จะถูกรันไหม? ทำไม?
>
> ```
> ตอบ:
>
>
> ```

> **คำถาม 3.3:** ลอง `strace -f ./fork_exec` — หา syscall `execve` ดูว่า argument เป็นอะไร
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
