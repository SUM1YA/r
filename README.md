# Understand Ownership

> Rust Ownership Cheat Sheet สำหรับอ่านทบทวนบนมือถือ

## 1. Mental Model

จำ 5 คำนี้ก่อน:

- **Owner** = คนที่เป็นเจ้าของ value
- **Move** = ย้าย Ownership
- **Clone** = สร้างข้อมูลอีกชุด
- **Copy** = copy value แบบ implicit
- **Borrow** = ยืมโดยไม่เอา Ownership

ภาพรวม:

```text
Ownership
├── Move   → ย้ายเจ้าของ
├── Clone  → สร้างข้อมูลใหม่
└── Copy   → copy ค่าอัตโนมัติ

Borrow
├── &T     → ยืมอ่าน
└── &mut T → ยืมแก้
```

## 2. Ownership คืออะไร?

Rust กำหนดว่า value แต่ละตัวมี Owner ในแต่ละช่วงเวลา

```rust
let s = String::from("hello");
```

`s` เป็น Owner ของ `String`

`String` มีข้อมูลบน heap:

```text
s
│
▼
String
├── pointer ──► heap: "hello"
├── length = 5
└── capacity = 5
```

เมื่อ `s` หลุด scope Rust จะ `drop` และคืน memory ให้เอง

```rust
{
    let s = String::from("hello");
} // drop(s)
```

นี่คือเหตุผลที่ Rust ไม่ต้องมี Garbage Collector เพื่อจัดการ memory

## 3. Scope

Ownership ผูกกับ scope

```rust
{
    let s = String::from("hello");
    println!("{}", s);
} // s ถูก drop
```

หลัง `}` ตัวแปร `s` ไม่มีอยู่แล้ว

## 4. Move

ถ้า type ไม่ใช่ `Copy` การ assign มักเป็นการ Move

```rust
let s1 = String::from("hello");
let s2 = s1;
```

หลัง Move:

```text
s1 ──X──► String

s2 ─────► String ─────► heap
```

ดังนั้น:

```rust
println!("{}", s2); // OK
println!("{}", s1); // ERROR
```

เหตุผลหลักคือป้องกัน double free

```text
ถ้า s1 และ s2 เป็น owner พร้อมกัน:

s1 ──┐
     ├──► heap
s2 ──┘

ตอน drop:
s1 → free(heap)
s2 → free(heap)  // ❌ double free
```

Rust จึงให้ Ownership มี owner เดียว และหลัง Move ตัวเดิมจะใช้งานไม่ได้

## 5. Function กับ Move

Parameter แบบ `T` รับ Ownership

```rust
fn print_string(s: String) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");
    print_string(s);

    // println!("{}", s); // ERROR
}
```

ภาพ:

```text
main
s
│
│ Move
▼
print_string
s
│
└── Owner
```

Function จึงสามารถ `drop` value เมื่อจบ scope ได้

## 6. Return Ownership

Function สามารถคืน Ownership ได้

```rust
fn create_name() -> String {
    String::from("Adam")
}

fn main() {
    let name = create_name();
}
```

ภาพ:

```text
create_name()
     │
     │ return Ownership
     ▼
   name
```

หรือส่งเข้าแล้วส่งกลับ:

```rust
fn process(s: String) -> String {
    s
}

fn main() {
    let s = String::from("hello");
    let s = process(s);
}
```

Ownership:

```text
main:s
  │
  │ Move
  ▼
process:s
  │
  │ return
  ▼
main:s
```

## 7. Clone

`clone()` สร้างข้อมูลใหม่แบบ explicit

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
```

ตอนนี้มีข้อมูล 2 ชุด:

```text
s1 ─────► String ─────► heap A
s2 ─────► String ─────► heap B
```

จึงใช้ได้ทั้งคู่:

```rust
println!("{}", s1);
println!("{}", s2);
```

Clone อาจมีค่าใช้จ่าย เพราะต้องสร้าง/คัดลอกข้อมูลใหม่

```rust
let a = String::from("large data...");
let b = a.clone();
```

อย่าใช้ `clone()` เพียงเพื่อแก้ compiler error โดยไม่เข้าใจว่าทำไมต้อง duplicate ข้อมูล

## 8. Copy

`Copy` คือการ copy value แบบ implicit

```rust
let x = 10;
let y = x;

println!("{}", x); // OK
println!("{}", y); // OK
```

`i32` เป็น `Copy`

แต่:

```rust
let s1 = String::from("hello");
let s2 = s1;

println!("{}", s1); // ERROR
```

`String` ไม่ใช่ `Copy`

## 9. Copy vs Clone

| | Copy | Clone |
|---|---|---|
| เรียกแบบ | implicit | explicit |
| Syntax | `let b = a` | `let b = a.clone()` |
| ตัวเดิมใช้ต่อ | ได้ | ได้ |
| สร้างข้อมูลใหม่ | copy value | สร้าง duplicate |
| เหมาะกับ | simple values | ข้อมูลที่ต้อง duplicate |
| ตัวอย่าง | `i32`, `bool`, `char` | `String`, `Vec<T>` |

จำง่าย:

```text
Copy  = "copy ให้เอง"
Clone = "ฉันสั่งให้ copy"
```

## 10. Type ที่เป็น Copy

ตัวอย่างที่พบบ่อย:

```rust
i8 i16 i32 i64 i128
u8 u16 u32 u64 u128
f32 f64
bool
char
```

Tuple จะ `Copy` ถ้าสมาชิกทุกตัว `Copy`

```rust
let a = (10, 20);
let b = a;

println!("{:?}", a); // OK
```

แต่:

```rust
let a = (10, String::from("Adam"));
let b = a;

// println!("{:?}", a); // ERROR
```

เพราะ `String` ไม่ใช่ `Copy`

## 11. Borrowing

Borrow = ยืม value โดยไม่เอา Ownership

ใช้ `&`

```rust
let s = String::from("hello");
let r = &s;
```

ภาพ:

```text
s ─────► String ─────► heap
          ▲
          │
          r
```

`s` ยังเป็น Owner

`r` เป็น Reference/Borrow ไม่ใช่ Owner

```rust
println!("{}", r);
println!("{}", s);
```

ทั้งคู่ใช้ได้

## 12. Immutable Borrow: `&T`

```rust
let s = String::from("hello");

let r1 = &s;
let r2 = &s;

println!("{}", r1);
println!("{}", r2);
```

สามารถมี Immutable Borrow หลายตัวพร้อมกันได้

```text
        ┌──► &s → r1 → read
        │
s ──────┤
        │
        └──► &s → r2 → read
```

เพราะไม่มีตัวไหนแก้ข้อมูล

## 13. Mutable Borrow: `&mut T`

```rust
let mut s = String::from("hello");

let r = &mut s;
r.push_str(" world");

println!("{}", r);
```

`&mut` ให้สิทธิ์แก้ไข value

```text
s ─────► String
          ▲
          │
        &mut s
          │
          ▼
        modify
```

Ownership ยังเป็นของ `s`

## 14. Borrow Rules

กฎหลักจำสั้น ๆ:

```text
หลาย &T
    OR
หนึ่ง &mut T

แต่ห้าม &T และ &mut T
ใช้งานพร้อมกันในช่วงเดียวกัน
```

ตัวอย่างที่ OK:

```rust
let s = String::from("hello");

let r1 = &s;
let r2 = &s;
let r3 = &s;

println!("{} {} {}", r1, r2, r3);
```

ตัวอย่างที่ผิด:

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &mut s;

println!("{}", r1);
println!("{}", r2);
```

เพราะมี Immutable Borrow ที่ยังใช้งานอยู่ และ Mutable Borrow พร้อมกัน

## 15. Non-Lexical Lifetimes (NLL)

Rust ดูว่า Reference ถูกใช้งานถึงตรงไหน ไม่ใช่แค่ดู scope ของตัวแปร

ดังนั้นโค้ดนี้ OK:

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;

println!("{} {}", r1, r2);

let r3 = &mut s;
r3.push_str(" world");

println!("{}", r3);
```

เหตุผลคือ `r1` และ `r2` ไม่มีการใช้งานหลัง `println!`

ภาพ:

```text
r1/r2
│
├── ใช้งาน
│
└── END

          ↓

r3 = &mut s
│
└── ใช้งาน
```

## 16. Function Parameter: `x: T`

```rust
fn process(x: String) {
    println!("{}", x);
}
```

`x: String` หมายถึง function รับ Ownership

```rust
let s = String::from("hello");
process(s);

// s ใช้ไม่ได้
```

ใช้เมื่อ function ต้องเป็นเจ้าของข้อมูล เช่น ต้องเก็บข้อมูลต่อ:

```rust
struct Database {
    users: Vec<String>,
}

impl Database {
    fn add_user(&mut self, user: String) {
        self.users.push(user);
    }
}
```

`add_user` ต้องรับ `String` เพราะ `Vec` ต้องกลายเป็น Owner ของมัน

## 17. Function Parameter: `x: &T`

```rust
fn print_name(x: &String) {
    println!("{}", x);
}

fn main() {
    let name = String::from("Adam");

    print_name(&name);

    println!("{}", name); // OK
}
```

Function ยืมข้อมูลและอ่านได้ แต่ไม่แก้ไข

ใช้เมื่อ:

- ต้องการอ่าน
- ไม่ต้องการเป็น Owner
- ไม่ต้องการแก้ caller's data

## 18. `&String` vs `&str`

ถ้า function แค่ต้องการอ่านข้อความ มักใช้ `&str`

```rust
fn print_text(text: &str) {
    println!("{}", text);
}
```

รับได้ทั้ง:

```rust
let s = String::from("hello");

print_text(&s);
print_text("hello");
```

ดังนั้นมัก prefer:

```rust
fn print_text(text: &str)
```

มากกว่า:

```rust
fn print_text(text: &String)
```

เมื่อไม่จำเป็นต้องรับเฉพาะ `String`

## 19. Function Parameter: `x: &mut T`

```rust
fn add_world(x: &mut String) {
    x.push_str(" world");
}

fn main() {
    let mut s = String::from("hello");

    add_world(&mut s);

    println!("{}", s);
}
```

Function ไม่ได้เป็น Owner แต่ได้รับสิทธิ์แก้ไขข้อมูล

```text
main
s ─────► String
          ▲
          │
          &mut
          │
      add_world
          │
          ▼
        modify
```

ใช้เมื่อ function ต้องแก้ข้อมูลของ caller แต่ไม่ควรรับ Ownership

## 20. `mut x: T`

นี่ต่างจาก `x: &mut T`

```rust
fn process(mut x: String) {
    x.push_str(" world");
}
```

มี 2 เรื่อง:

```text
mut x
│
└── x เป็น mutable binding

x: String
│
└── x เป็น Owner
```

ดังนั้น:

```rust
fn process(mut x: String) {
    x.push_str(" world");
}
```

function รับ Ownership และสามารถแก้ `String` ที่ x เป็นเจ้าของได้

Caller:

```rust
let s = String::from("hello");

process(s);

// s ใช้ไม่ได้
```

`mut` ไม่ได้ทำให้ caller แก้ `s` ได้

## 21. `x: &mut T` vs `mut x: T`

นี่คือจุดที่ต้องแยกให้ออก

### `mut x: T`

```rust
fn f(mut x: String) {
    x.push_str("!");
}
```

ความหมาย:

```text
function เป็น Owner
และ x เป็น mutable binding
```

Caller:

```text
s ──Move──► x
```

หลัง function รับไป `s` ใช้ไม่ได้

### `x: &mut T`

```rust
fn f(x: &mut String) {
    x.push_str("!");
}
```

ความหมาย:

```text
function ไม่ได้เป็น Owner
แต่ยืมไปแก้
```

Caller:

```text
s ─────► String
          ▲
          │ &mut
          x
```

หลัง function จบ caller ยังเป็น Owner

## 22. `mut x: &mut T`

สามารถเขียนได้:

```rust
fn f(mut x: &mut String) {
    x.push_str("!");

    let mut other = String::from("other");
    x = &mut other;
}
```

ตรงนี้ `mut` กับ `&mut` คนละหน้าที่

```text
mut x
→ เปลี่ยน binding ของ x ได้

&mut T
→ แก้ข้อมูลที่ x อ้างถึงได้
```

โดยทั่วไป function ไม่ต้องใช้ `mut x: &mut T` ถ้าแค่ต้องการแก้ข้อมูล

```rust
fn f(x: &mut String) {
    x.push_str("!");
}
```

ก็พอ

## 23. Function Parameter ทั้ง 4 แบบ

สมมติ:

```rust
struct User {
    name: String,
}
```

### `x: User`

```rust
fn process(x: User) {
    println!("{}", x.name);
}
```

```text
รับ Ownership
→ function เป็น Owner
→ caller เสีย Ownership
→ แก้ข้อมูลไม่ได้ถ้า binding ไม่ mutable
```

### `mut x: User`

```rust
fn process(mut x: User) {
    x.name.push_str(" Smith");
}
```

```text
รับ Ownership
→ function เป็น Owner
→ x mutable
→ caller เสีย Ownership
```

### `x: &User`

```rust
fn process(x: &User) {
    println!("{}", x.name);
}
```

```text
Borrow
→ อ่านได้
→ แก้ไม่ได้
→ caller ยังเป็น Owner
→ มีหลาย immutable borrow ได้
```

### `x: &mut User`

```rust
fn process(x: &mut User) {
    x.name.push_str(" Smith");
}
```

```text
Borrow
→ อ่านได้
→ แก้ได้
→ caller ยังเป็น Owner
→ มี mutable borrow ได้ทีละหนึ่ง
```

## 24. ตาราง Function Parameter

| Parameter | Owner ใน function? | Borrow? | อ่าน | แก้ข้อมูล | Caller ยังเป็น Owner? |
|---|---:|---:|---:|---:|---:|
| `x: T` | ✅ | ❌ | ✅ | ถ้า `mut` | ❌ |
| `mut x: T` | ✅ | ❌ | ✅ | ✅ | ❌ |
| `x: &T` | ❌ | ✅ | ✅ | ❌ | ✅ |
| `x: &mut T` | ❌ | ✅ | ✅ | ✅ | ✅ |
| `mut x: &mut T` | ❌ | ✅ | ✅ | ✅ | ✅ |

## 25. `mut` กับ `&mut` จำแบบนี้

> **`mut` = mutable binding**
>
> **`&mut` = mutable reference**

ตัวอย่าง:

```rust
let mut s = String::from("hello");
```

`mut s`:

```text
s สามารถถูกเปลี่ยน/แก้ได้
```

ตัวอย่าง:

```rust
let r = &mut s;
```

`&mut s`:

```text
r สามารถแก้ value ที่ s เป็น Owner ได้
```

ใน function:

```rust
fn f(mut x: String)
```

```text
x เป็น Owner
x เป็น mutable binding
```

ส่วน:

```rust
fn f(x: &mut String)
```

```text
x ไม่ใช่ Owner
x เป็น mutable reference
```

## 26. Function Design: ควรเลือกอะไร?

คิดจากคำถามนี้:

```text
Function ต้องการ Ownership ไหม?
│
├── ใช่ → T
│
└── ไม่
    │
    ├── แค่อ่าน → &T
    │
    └── ต้องแก้ → &mut T
```

ตัวอย่าง:

```rust
// อ่าน
fn print_user(user: &User) {
    println!("{}", user.name);
}

// แก้
fn rename(user: &mut User) {
    user.name = String::from("Eva");
}

// รับไปเป็นเจ้าของ
fn save_user(user: User) {
    // store user
}
```

นี่เป็น rule ที่ใช้ได้บ่อยมากในการออกแบบ function

## 27. Example: String Processor

```rust
fn print_text(text: &str) {
    println!("text = {}", text);
}

fn add_suffix(text: &mut String) {
    text.push('!');
}

fn consume_text(text: String) {
    println!("consumed = {}", text);
}

fn main() {
    let mut text = String::from("hello");

    // Borrow เพื่ออ่าน
    print_text(&text);

    // Mutable Borrow เพื่อแก้
    add_suffix(&mut text);

    // Borrow เพื่ออ่านอีกครั้ง
    print_text(&text);

    // Move Ownership
    consume_text(text);

    // text ใช้ไม่ได้แล้ว
}
```

ลำดับ:

```text
text
 │
 ├── &text
 │    └── read
 │
 ├── &mut text
 │    └── modify
 │
 └── text
      └── Move
```

## 28. Borrow ใน Function ไม่ได้สร้าง Ownership ใหม่

```rust
fn read(s: &String) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");

    read(&s);

    println!("{}", s);
}
```

ต่างจาก:

```rust
fn read(s: String) {
    println!("{}", s);
}
```

แบบแรก:

```text
s ─────────► String
              ▲
              │
             &s
```

แบบสอง:

```text
s ──Move──► function parameter
```

## 29. Mutable Borrow ต้องเริ่มจาก Mutable Owner

แบบนี้ผิด:

```rust
let s = String::from("hello");

let r = &mut s; // ERROR
```

เพราะ `s` ไม่ได้ประกาศ `mut`

ต้อง:

```rust
let mut s = String::from("hello");

let r = &mut s;
```

คิดว่า:

```text
let mut s
    ↓
เจ้าของอนุญาตให้แก้

&mut s
    ↓
ยืมสิทธิ์แก้
```

## 30. Mutable Borrow ใช้พร้อมกันไม่ได้

ผิด:

```rust
let mut s = String::from("hello");

let r1 = &mut s;
let r2 = &mut s;
```

เพราะมี Mutable Borrow 2 ตัวพร้อมกัน

```text
s
├── &mut → r1
└── &mut → r2  ❌
```

แต่ทำต่อกันได้:

```rust
let mut s = String::from("hello");

{
    let r1 = &mut s;
    r1.push('!');
}

{
    let r2 = &mut s;
    r2.push('?');
}
```

เพราะ `r1` จบก่อน `r2` เริ่ม

## 31. Immutable Borrow หลายตัวได้

```rust
let s = String::from("hello");

let r1 = &s;
let r2 = &s;
let r3 = &s;

println!("{}", r1);
println!("{}", r2);
println!("{}", r3);
```

ภาพ:

```text
          ┌── r1
          ├── r2
s ────────┼── r3
          │
          └── read only
```

## 32. `&T` และ `&mut T` กับ Function Call

ประกาศ:

```rust
fn read(s: &String) {}
fn write(s: &mut String) {}
```

เรียก:

```rust
let mut s = String::from("hello");

read(&s);
write(&mut s);
```

สังเกต:

```text
Function parameter:
&T
&mut T

ตอน call:
&s
&mut s
```

คือ function บอกว่า "ฉันต้องการ reference แบบไหน"

และ caller เป็นคนส่ง reference ให้

## 33. Ownership vs Borrow ใน Function

### Ownership

```rust
fn take(s: String) {}
```

Call:

```rust
take(s);
```

```text
caller ──Move──► function
```

### Immutable Borrow

```rust
fn read(s: &String) {}
```

Call:

```rust
read(&s);
```

```text
caller ──Borrow──► function
```

### Mutable Borrow

```rust
fn write(s: &mut String) {}
```

Call:

```rust
write(&mut s);
```

```text
caller ──Mutable Borrow──► function
```

## 34. Clone กับ Function

ถ้า function รับ Ownership แต่เราอยากเก็บของเดิมไว้:

```rust
fn consume(s: String) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");

    consume(s.clone());

    println!("{}", s);
}
```

ทำได้ แต่เกิดการ duplicate data

ถ้า function แค่ต้องอ่าน ไม่ควร clone โดยไม่จำเป็น:

```rust
fn print_text(s: &str) {
    println!("{}", s);
}

fn main() {
    let s = String::from("hello");

    print_text(&s);
}
```

เปรียบเทียบ:

```text
ต้องการอ่าน
→ &str

ต้องการแก้
→ &mut String

ต้องการรับ Ownership
→ String

ต้องการ copy เพื่อให้ทั้งสองฝั่งมีข้อมูล
→ clone()
```

## 35. ตัวอย่าง Tuple: Copy vs Clone

```rust
let adam_info = (21, "Adam");
let adam_info_copy = adam_info;

println!("{:?}", adam_info);
println!("{:?}", adam_info_copy);
```

Tuple นี้ `Copy` เพราะ:

```text
i32    → Copy
&str   → Copy
```

ดังนั้น tuple ทั้งตัว Copy ได้

แต่:

```rust
let eva_info = (18, String::from("Eva"));
let eva_info_copy = eva_info.clone();

println!("{:?}", eva_info);
println!("{:?}", eva_info_copy);
```

ต้องใช้ `clone()` เพราะ:

```text
i32      → Copy
String   → ไม่ Copy
```

ดังนั้น:

```text
(i32, &str)
    ↓
Copy ได้

(i32, String)
    ↓
Copy ไม่ได้
    ↓
Clone ได้
```

## 36. Common Mistakes

### Mistake 1: คิดว่า `mut` หมายถึง Borrow

ผิด:

```text
mut = borrow
```

ถูก:

```text
mut  = mutable binding
&mut = mutable reference
```

### Mistake 2: คิดว่า `clone()` = `Copy`

ไม่เหมือนกัน:

```text
Copy
→ implicit
→ type ต้อง implement Copy

Clone
→ explicit
→ เรียก .clone()
```

### Mistake 3: ใช้ `clone()` แก้ทุก compiler error

เช่น:

```rust
fn print_name(name: String) {}

let name = String::from("Adam");

print_name(name.clone());
```

อาจทำงาน แต่ถ้า function แค่ต้องอ่าน ควรออกแบบเป็น:

```rust
fn print_name(name: &str) {}
```

แล้ว:

```rust
print_name(&name);
```

### Mistake 4: คิดว่า `&mut T` ย้าย Ownership

ไม่ย้าย

```rust
fn modify(s: &mut String) {
    s.push('!');
}
```

Ownership ยังอยู่ที่ caller

## 37. Final Cheat Sheet

```text
OWNERSHIP
─────────
T
→ เป็น Owner
→ Move เมื่อส่งต่อ
→ caller เสีย Ownership

BORROW
──────
&T
→ ยืมอ่าน
→ ไม่เป็น Owner
→ หลายตัวพร้อมกันได้

&mut T
→ ยืมแก้
→ ไม่เป็น Owner
→ mutable borrow ได้ทีละหนึ่ง

MUT
───
mut x: T
→ x เป็น Owner
→ x เป็น mutable binding

mut x: &mut T
→ x เป็น reference
→ x แก้ข้อมูลได้
→ mut เพิ่มความสามารถในการเปลี่ยน binding ของ x

COPY
────
let b = a;
→ copy อัตโนมัติ
→ a ยังใช้ได้
→ type ต้อง Copy

CLONE
─────
let b = a.clone();
→ สร้างข้อมูลอีกชุด
→ a ยังใช้ได้
→ อาจมี allocation/copy cost
```

## 38. Decision Tree

เวลาเห็นหรือเขียน Rust function ให้ถาม:

```text
ข้อมูลนี้ต้องถูกส่ง Ownership ไปไหม?
│
├── YES
│   │
│   ├── ต้องแก้ local value?
│   │      ├── YES → mut x: T
│   │      └── NO  → x: T
│   │
│   └── ต้องคืน Ownership?
│          → T / return T
│
└── NO → ใช้ Borrow
    │
    ├── แค่อ่าน
    │     → x: &T
    │     → text มักใช้ &str
    │
    └── ต้องแก้
          → x: &mut T
```

## 39. Mental Model สุดท้าย

เมื่ออ่าน code Rust ให้ไล่ถาม:

```text
1. ใครเป็น Owner?
        ↓
2. มี Move หรือไม่?
        ↓
3. Type นี้ Copy หรือไม่?
        ↓
4. มี Clone หรือไม่?
        ↓
5. มี Borrow หรือไม่?
        ↓
6. เป็น &T หรือ &mut T?
        ↓
7. มี &T หลายตัวหรือไม่?
        ↓
8. มี &mut T ซ้อน/พร้อมกันหรือไม่?
        ↓
9. function รับ Ownership หรือ Borrow?
        ↓
10. function ต้องอ่านหรือแก้?
```

### จำให้ได้แค่บรรทัดนี้ก่อน

```text
T        = เอาของไปเป็นเจ้าของ
mut T    = เอาของไปเป็นเจ้าของ + แก้ local binding
&T       = ยืมอ่าน
&mut T   = ยืมแก้
clone()  = สร้างของใหม่
Copy     = copy ให้อัตโนมัติ
```

### Rule สำคัญที่สุดของ Borrow

```text
หลาย &T
    OR
หนึ่ง &mut T

และ
&T กับ &mut T
ห้ามใช้งานพร้อมกันในช่วงเดียวกัน
```