# Lab 3: The Role of the Linker and Loader
---

## วัตถุประสงค์

1. เข้าใจขั้นตอนการแปลง source code (`.c`) เป็น executable ที่รันได้
2. ใช้คำสั่ง `gcc` แยกขั้นตอน compile และ link ได้
3. อธิบายความแตกต่างระหว่าง static linking กับ dynamic linking
4. ใช้เครื่องมือ `file`, `nm`, `ldd` เพื่อตรวจสอบไฟล์แต่ละขั้นตอน

## สิ่งที่ต้องเตรียม

- ระบบ Linux (Ubuntu/Debian) หรือ WSL บน Windows
- ติดตั้ง gcc และ build tools:

```bash
sudo apt install gcc build-essential
```

---

## Part 1 — สร้าง Source Code

### Step 1.1 — สร้างโฟลเดอร์และไฟล์ต่างๆ

```bash
mkdir -p ~/lab_linker && cd ~/lab_linker
```

สร้างไฟล์ `helper.h`:

```c
// helper.h
#ifndef HELPER_H
#define HELPER_H
void greet(const char *name);
#endif
```

สร้างไฟล์ `helper.c`:

```c
// helper.c
#include <stdio.h>

void greet(const char *name) {
    printf("Hello, %s!\n", name);
}
```

สร้างไฟล์ `main.c` ที่ใช้ทั้ง math library และ helper:

```c
// main.c
#include <stdio.h>
#include <math.h>
#include "helper.h"

int main() {
    double x = 2.0;
    printf("sqrt(%.1f) = %.4f\n", x, sqrt(x));
    greet("Lab Student");
    return 0;
}
```

---

## Part 2 — Compile: สร้าง Object File

### Step 2.1 — Compile แยกเป็น object file ด้วย flag `-c`

```bash
gcc -c main.c -o main.o
gcc -c helper.c -o helper.o
```

### Step 2.2 — ตรวจสอบไฟล์ที่ได้

```bash
ls -la *.o
file main.o
file helper.o
```

> **คำถาม 2.1:** คำสั่ง `file main.o` แสดงผลว่าอะไร? ไฟล์ชนิดนี้คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

### Step 2.3 — ดู symbol table ด้วย `nm`

```bash
nm main.o
```

> **คำถาม 2.2:** ดูผลของ `nm main.o` — สัญลักษณ์ `U` หมายถึงอะไร? ลองหา `sqrt` และ `greet` ในผลลัพธ์แล้วอธิบายว่าทำไมมันเป็น `U`
>
> ```
> ตอบ:
>
>
> ```

💡 **Hint:** U = Undefined — หมายถึง symbol ที่ถูกใช้แต่ยังไม่มี definition อยู่ในไฟล์นี้

---

## Part 3 — Link: สร้าง Executable

### Step 3.1 — ลอง link โดยไม่ใส่ `-lm` (จะเกิด error)

```bash
gcc -o main main.o helper.o
```

> **คำถาม 3.1:** เกิด error อะไร? ทำไมถึง link ไม่สำเร็จ?
>
> ```
> ตอบ:
>
>
> ```

### Step 3.2 — Link ให้สมบูรณ์ด้วย `-lm`

```bash
gcc -o main main.o helper.o -lm
```

### Step 3.3 — ตรวจสอบ executable ที่ได้

```bash
file main
ls -la main
```

> **คำถาม 3.2:** เปรียบเทียบผลของ `file main.o` กับ `file main` — ต่างกันอย่างไร?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 4 — Dynamic Libraries และ Loader

### Step 4.1 — ดู dynamic dependencies ด้วย `ldd`

```bash
ldd main
```

> **คำถาม 4.1:** `ldd` แสดง library อะไรบ้าง? `libm.so` และ `libc.so` คืออะไร?
>
> ```
> ตอบ:
>
>
> ```

### Step 4.2 — เปรียบเทียบ Static Linking กับ Dynamic Linking

```bash
gcc -o main_static main.o helper.o -lm -static
ls -la main main_static
ldd main_static
file main_static
```

> **คำถาม 4.2:** เปรียบเทียบขนาดของ `main` กับ `main_static` — ทำไม static ถึงใหญ่กว่ามาก?
>
> ```
> ตอบ:
>
>
> ```

---

## Part 5 — รันโปรแกรม

### Step 5.1 — รัน executable ทั้งสองแบบ

```bash
./main
./main_static
```

> **คำถาม 5.1:** ผลลัพธ์ของทั้งสองคำสั่งเหมือนกันหรือไม่? อธิบายว่าทำไม
>
> ```
> ตอบ:
>
>
> ```

---

## Part 6 — ทำทุกขั้นตอนในคำสั่งเดียว

ปกติเราใช้ `gcc` รวม compile + link ในคำสั่งเดียว:

```bash
gcc -o main_onestep main.c helper.c -lm
```

> **คำถาม 6.1:** คำสั่งนี้ทำอะไรให้เราอัตโนมัติบ้าง? (อ้างอิง Part 2–3)
>
> ```
> ตอบ:
>
>
> ```

---

## สรุปขั้นตอน

| ขั้นตอน | Input | Output | คำสั่ง | ตรวจสอบด้วย |
|---------|-------|--------|--------|------------|
| Compile | `.c` | `.o` | `gcc -c main.c` | `file`, `nm` |
| Link | `.o` + libs | executable | `gcc -o main *.o -lm` | `file`, `nm` |
| Load & Run | executable | process | `./main` | `ldd` |

---

## โจทย์เพิ่มเติม (Bonus)

1. ลองใช้ `gcc -S main.c` เพื่อดู assembly output แล้วเปิดดูด้วย `cat main.s`

2. ลองใช้ `objdump -d main.o` เพื่อดู disassembly ของ object file

3. สร้าง shared library จาก `helper.c`:

```bash
gcc -shared -fPIC -o libhelper.so helper.c
gcc -o main_shared main.c -L. -lhelper -lm
LD_LIBRARY_PATH=. ./main_shared
```

4. ลองใช้ `strace ./main` เพื่อดูว่า loader ทำอะไรบ้างตอนโหลดโปรแกรมเข้า memory
