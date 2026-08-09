# 🦀 Rust Ownership, Borrowing, Clone & Copy — คู่มือฉบับสมบูรณ์

## สารบัญ

1. [Ownership คืออะไร?](#1-ownership-คืออะไร)
2. [Ownership vs Clone vs Copy](#2-ownership-vs-clone-vs-copy)
3. [Owner มีกี่แบบ?](#3-owner-มีกี่แบบ)
4. [Borrow มีกี่แบบ?](#4-borrow-มีกี่แบบ)
5. [Ownership Rules สรุป 3 กฎเหล็ก](#5-ownership-rules-สรุป-3-กฎเหล็ก)
6. [การรับส่ง Parameter ในฟังก์ชัน — ทุกรูปแบบ](#6-การรับส่ง-parameter-ในฟังก์ชัน--ทุกรูปแบบ)
7. [เปรียบเทียบ `x: T` vs `x: &T` vs `x: &mut T` vs `mut x: T` vs `mut x: &T` vs `mut x: &mut T`](#7-เปรียบเทียบทุกรูปแบบ-parameter)
8. [Lifetime กับ Borrowing](#8-lifetime-กับ-borrowing)
9. [สรุปตาราง Cheat Sheet](#9-สรุปตาราง-cheat-sheet)
10. [Anti-patterns และ Common Mistakes](#10-anti-patterns-และ-common-mistakes)

---

## 1. Ownership คืออะไร?

Ownership คือระบบการจัดการหน่วยความจำ (Memory Management) ของ Rust ที่ **ไม่ต้องใช้ Garbage Collector** โดยทุกค่า (value) ใน Rust จะมี **เจ้าของ (owner)** เพียงตัวเดียวเท่านั้น เมื่อ owner ออกจาก scope ค่าจะถูก drop (ทำลาย) ทันที

```rust
fn main() {
    let s = String::from("hello"); // s เป็น owner ของ String "hello"
    println!("{}", s);             // ใช้งานได้ปกติ
}                                  // s ออกจาก scope → String ถูก drop → หน่วยความจำถูกคืน
```

### ทำไมต้อง Ownership?

| ภาษา | วิธีจัดการ Memory | ข้อเสีย |
|------|-------------------|---------|
| C/C++ | Manual (`malloc`/`free`) | Memory leak, dangling pointer, double free |
| Java/Go/Python | Garbage Collector (GC) | GC pause, ใช้ memory เพิ่ม, ไม่ deterministic |
| **Rust** | **Ownership System** | **ไม่มี runtime cost, ปลอดภัย 100% ตอน compile** |

---

## 2. Ownership vs Clone vs Copy

### 2.1 Ownership (Move Semantics) — การย้ายความเป็นเจ้าของ

เมื่อ assign ค่าจาก variable หนึ่งไปอีกตัว สำหรับ type ที่ **ไม่ได้** implement `Copy` trait จะเกิด **move** — ownership ถูกย้ายไป และ variable เดิมจะใช้งานไม่ได้อีก

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1; // ⚡ Move! ownership ย้ายจาก s1 → s2

    // println!("{}", s1); // ❌ Compile Error! s1 ถูก move ไปแล้ว
    println!("{}", s2);    // ✅ OK, s2 เป็น owner ตัวใหม่
}
```

**สิ่งที่เกิดขึ้นใน Memory:**

```
ก่อน Move:
Stack                     Heap
┌─────────┐              ┌─────────────┐
│ s1      │───────────▶  │ "hello"     │
│ ptr     │              └─────────────┘
│ len: 5  │
│ cap: 5  │
└─────────┘

หลัง Move:
Stack                     Heap
┌─────────┐
│ s1      │  (invalid)
│ xxxxxxx │
└─────────┘
┌─────────┐              ┌─────────────┐
│ s2      │───────────▶  │ "hello"     │
│ ptr     │              └─────────────┘
│ len: 5  │
│ cap: 5  │
└─────────┘
```

> [!IMPORTANT]
> **Move ไม่ได้ copy ข้อมูลบน Heap** แค่ย้าย pointer ไป — ทำให้ **เร็วมาก** (O(1)) ไม่ว่า String จะยาวแค่ไหน

---

### 2.2 Clone — การ Deep Copy

`Clone` คือการ **สำเนาข้อมูลทั้งหมด** รวมถึงข้อมูลบน Heap ได้ตัวใหม่ที่เป็นอิสระจากตัวเดิม ต้องเรียก `.clone()` อย่างชัดเจน (explicit)

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.clone(); // 📋 Deep copy! สร้าง String ใหม่บน Heap

    println!("s1 = {}", s1); // ✅ OK, s1 ยังใช้ได้
    println!("s2 = {}", s2); // ✅ OK, s2 เป็นสำเนาที่เป็นอิสระ
}
```

**สิ่งที่เกิดขึ้นใน Memory:**

```
หลัง Clone:
Stack                     Heap
┌─────────┐              ┌─────────────┐
│ s1      │───────────▶  │ "hello"     │  ← ข้อมูลชุดเดิม
│ ptr     │              └─────────────┘
│ len: 5  │
│ cap: 5  │
└─────────┘
┌─────────┐              ┌─────────────┐
│ s2      │───────────▶  │ "hello"     │  ← ข้อมูลชุดใหม่ (คนละที่)
│ ptr     │              └─────────────┘
│ len: 5  │
│ cap: 5  │
└─────────┘
```

> [!WARNING]
> `.clone()` อาจ **แพง** (expensive) สำหรับข้อมูลขนาดใหญ่ เพราะต้อง allocate memory ใหม่และ copy ทุก byte — ใช้เมื่อจำเป็นเท่านั้น!

#### Custom Clone

```rust
#[derive(Debug)]
struct Player {
    name: String,
    score: u32,
    inventory: Vec<String>,
}

// implement Clone ด้วยมือ
impl Clone for Player {
    fn clone(&self) -> Self {
        println!("🔄 Cloning player: {}", self.name);
        Player {
            name: self.name.clone(),
            score: self.score,                // u32 เป็น Copy → clone ฟรี
            inventory: self.inventory.clone(), // Vec ต้อง deep clone
        }
    }
}

fn main() {
    let p1 = Player {
        name: String::from("Alice"),
        score: 100,
        inventory: vec![String::from("Sword"), String::from("Shield")],
    };

    let p2 = p1.clone(); // เรียก custom clone

    println!("p1: {:?}", p1); // ✅ ยังใช้ได้
    println!("p2: {:?}", p2); // ✅ สำเนาอิสระ
}
```

---

### 2.3 Copy — การ Bitwise Copy อัตโนมัติ

`Copy` คือ trait พิเศษที่ทำให้ค่าถูก **copy โดยอัตโนมัติ** เมื่อ assign หรือส่งเข้าฟังก์ชัน **ไม่เกิด move** เหมาะกับ type ขนาดเล็กที่เก็บบน Stack ทั้งหมด

```rust
fn main() {
    let x: i32 = 42;
    let y = x; // 📋 Copy! (ไม่ใช่ Move)

    println!("x = {}", x); // ✅ OK, x ยังใช้ได้
    println!("y = {}", y); // ✅ OK, y เป็นสำเนา
}
```

#### ประเภทที่ implement Copy โดยธรรมชาติ:

| ประเภท | ตัวอย่าง |
|--------|---------|
| Integer types | `i8`, `i16`, `i32`, `i64`, `i128`, `u8`, `u16`, `u32`, `u64`, `u128`, `isize`, `usize` |
| Floating point | `f32`, `f64` |
| Boolean | `bool` |
| Character | `char` |
| Tuples (ถ้าสมาชิกทุกตัว Copy) | `(i32, f64)`, `(bool, char)` |
| Arrays (ถ้าสมาชิกทุกตัว Copy) | `[i32; 5]`, `[bool; 10]` |
| References (`&T`) | `&String`, `&Vec<i32>` — **shared reference เป็น Copy** |

#### ประเภทที่ **ไม่** implement Copy:

| ประเภท | เหตุผล |
|--------|--------|
| `String` | มี pointer ไป Heap — ถ้า bitwise copy จะเกิด double free |
| `Vec<T>` | มี pointer ไป Heap |
| `Box<T>` | มี pointer ไป Heap |
| `&mut T` | Mutable reference ห้ามมีมากกว่า 1 ตัว |
| Struct/Enum ที่มี field ไม่เป็น Copy | ถ้าแม้แต่ field เดียวไม่ Copy ทั้ง struct ก็ไม่ Copy |

#### Custom Copy:

```rust
#[derive(Debug, Clone, Copy)] // ✅ ทุก field เป็น Copy ได้
struct Point {
    x: f64,
    y: f64,
}

fn main() {
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = p1; // Copy! ไม่ใช่ Move

    println!("p1 = {:?}", p1); // ✅ ยังใช้ได้
    println!("p2 = {:?}", p2); // ✅ สำเนา
}
```

```rust
// ❌ ไม่สามารถ derive Copy ได้ เพราะ String ไม่ Copy
#[derive(Debug, Clone, Copy)] // Compile Error!
struct User {
    name: String, // ← String ไม่ Copy
    age: u32,
}
```

> [!NOTE]
> **Copy เป็น subtrait ของ Clone** — ทุก type ที่ `Copy` ต้อง `Clone` ด้วย แต่ไม่ใช่ทุก type ที่ `Clone` จะ `Copy` ได้

---

### 2.4 สรุปเปรียบเทียบ Ownership vs Clone vs Copy

| คุณสมบัติ | Move (Ownership) | Clone | Copy |
|-----------|-----------------|-------|------|
| **เกิดขึ้นเมื่อ** | Assign / ส่งเข้าฟังก์ชัน | เรียก `.clone()` | Assign / ส่งเข้าฟังก์ชัน |
| **Explicit/Implicit** | Implicit (อัตโนมัติ) | **Explicit** (ต้องเรียกเอง) | Implicit (อัตโนมัติ) |
| **ตัวเดิมใช้ได้อีกไหม** | ❌ ไม่ได้ | ✅ ได้ | ✅ ได้ |
| **Cost** | O(1) — แค่ย้าย pointer | O(n) — deep copy | O(1) — bitwise copy ขนาดเล็ก |
| **ข้อมูลบน Heap** | ไม่ copy, ย้าย pointer | Copy ทั้งหมด | ไม่มีข้อมูลบน Heap |
| **ใช้กับ type ไหน** | ทุก type ที่ไม่ Copy | ทุก type ที่ impl Clone | เฉพาะ type ขนาดเล็กบน Stack |

---

## 3. Owner มีกี่แบบ?

Owner ใน Rust มี **2 แบบหลัก**:

### 3.1 Immutable Owner — เจ้าของแบบอ่านอย่างเดียว

```rust
fn main() {
    let s = String::from("hello"); // immutable owner
    // s.push_str(" world");       // ❌ ไม่สามารถแก้ไขได้
    println!("{}", s);             // ✅ อ่านได้
}
```

### 3.2 Mutable Owner — เจ้าของแบบแก้ไขได้

```rust
fn main() {
    let mut s = String::from("hello"); // mutable owner
    s.push_str(" world");              // ✅ แก้ไขได้
    println!("{}", s);                 // "hello world"
}
```

### 3.3 Owner ผ่าน Smart Pointers

นอกจาก ownership ธรรมดา ยังมี ownership ผ่าน smart pointer:

```rust
use std::rc::Rc;
use std::sync::Arc;

fn main() {
    // Box<T> — single owner บน Heap
    let boxed: Box<String> = Box::new(String::from("hello"));

    // Rc<T> — multiple owners (single-threaded) — Reference Counted
    let shared1: Rc<String> = Rc::new(String::from("shared"));
    let shared2 = Rc::clone(&shared1); // เพิ่ม reference count (ไม่ deep clone)
    println!("count = {}", Rc::strong_count(&shared1)); // 2

    // Arc<T> — multiple owners (multi-threaded) — Atomic Reference Counted
    let arc1: Arc<String> = Arc::new(String::from("thread-safe"));
    let arc2 = Arc::clone(&arc1);
}
```

| Smart Pointer | Owner กี่ตัว | Thread-safe | ใช้เมื่อ |
|--------------|-------------|-------------|---------|
| `Box<T>` | 1 ตัว | ✅ (ถ้า T: Send) | ต้องการ allocate บน Heap |
| `Rc<T>` | หลายตัว | ❌ | Shared ownership ใน single thread |
| `Arc<T>` | หลายตัว | ✅ | Shared ownership ข้าม threads |

---

## 4. Borrow มีกี่แบบ?

Borrow (การยืม) ใน Rust มี **2 แบบ**:

### 4.1 Shared/Immutable Borrow (`&T`) — ยืมแบบอ่านอย่างเดียว

- ยืมได้ **หลายตัวพร้อมกัน**
- **อ่านได้** แต่ **แก้ไขไม่ได้**

```rust
fn main() {
    let s = String::from("hello");

    let r1 = &s; // shared borrow #1
    let r2 = &s; // shared borrow #2
    let r3 = &s; // shared borrow #3 — ไม่จำกัดจำนวน!

    println!("{}, {}, {}", r1, r2, r3); // ✅ ใช้ได้ทุกตัว
    println!("{}", s);                  // ✅ owner ยังใช้ได้
}
```

### 4.2 Exclusive/Mutable Borrow (`&mut T`) — ยืมแบบแก้ไขได้

- ยืมได้ **ตัวเดียวเท่านั้น** ณ เวลาหนึ่ง
- **อ่านและแก้ไขได้**
- ระหว่างที่มี `&mut T` **ห้ามมี** `&T` อื่นอยู่

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s; // exclusive borrow
    r1.push_str(" world");

    // let r2 = &s;    // ❌ ห้ามยืมอ่านขณะมี &mut อยู่
    // let r3 = &mut s; // ❌ ห้ามมี &mut ตัวที่สอง

    println!("{}", r1); // "hello world"
}
```

### 4.3 Borrowing Rules — กฎการยืม

```rust
fn main() {
    let mut s = String::from("hello");

    // ✅ Case 1: หลาย shared borrows พร้อมกัน
    {
        let r1 = &s;
        let r2 = &s;
        println!("{} {}", r1, r2);
    } // r1, r2 หมด scope

    // ✅ Case 2: mutable borrow ตัวเดียว
    {
        let r3 = &mut s;
        r3.push_str(" world");
        println!("{}", r3);
    } // r3 หมด scope

    // ❌ Case 3: ผสม shared + mutable (ILLEGAL)
    // let r4 = &s;
    // let r5 = &mut s; // Compile Error!
    // println!("{} {}", r4, r5);

    // ✅ Case 4: Non-Lexical Lifetimes (NLL) — Rust ฉลาดพอจะเข้าใจ
    let r6 = &s;
    println!("{}", r6);   // r6 ถูกใช้ครั้งสุดท้ายตรงนี้
    let r7 = &mut s;      // ✅ OK! เพราะ r6 ไม่ถูกใช้อีกแล้ว (NLL)
    r7.push_str("!");
    println!("{}", r7);
}
```

> [!TIP]
> **NLL (Non-Lexical Lifetimes)** — Rust compiler ตรวจสอบว่า reference ถูกใช้ครั้งสุดท้ายเมื่อไหร่ ไม่ใช่แค่ดูว่าออกจาก scope เมื่อไหร่ ทำให้โค้ดยืดหยุ่นมากขึ้น

---

## 5. Ownership Rules สรุป 3 กฎเหล็ก

```
📜 กฎที่ 1: ทุกค่ามี owner ได้เพียงตัวเดียว ณ เวลาหนึ่ง
📜 กฎที่ 2: เมื่อ owner ออกจาก scope ค่าจะถูก drop
📜 กฎที่ 3: ณ เวลาหนึ่ง จะมีได้แค่:
            - หลาย shared references (&T) หรือ
            - หนึ่ง mutable reference (&mut T)
            แต่ไม่สามารถมีทั้งสองอย่างพร้อมกัน
```

---

## 6. การรับส่ง Parameter ในฟังก์ชัน — ทุกรูปแบบ

นี่คือหัวใจสำคัญที่สุด — การทำความเข้าใจว่าแต่ละรูปแบบ parameter ทำงานต่างกันอย่างไร:

---

### 6.1 `x: T` — รับ Ownership (Move/Copy เข้าฟังก์ชัน)

**ความหมาย**: ฟังก์ชันรับ ownership ของค่า ตัวแปรเดิมจะ **ใช้ไม่ได้** (ถ้าไม่ Copy)

**บริบทที่ใช้**:
- ฟังก์ชันต้องการ **เป็นเจ้าของ** ข้อมูลถาวร
- ฟังก์ชันจะ **เปลี่ยนแปลง** หรือ **ทำลาย** ข้อมูล
- ฟังก์ชันต้อง **ส่งต่อ ownership** ให้คนอื่น
- ฟังก์ชันเก็บค่าไว้ใน struct หรือ return ออกไป

```rust
// ===== ตัวอย่าง 1: รับ ownership ของ String =====
fn consume_string(s: String) {
    println!("I own: {}", s);
} // s ถูก drop ตรงนี้ — memory ถูกคืน

fn main() {
    let greeting = String::from("hello");
    consume_string(greeting); // ownership ย้ายไป
    // println!("{}", greeting); // ❌ Compile Error! greeting ถูก move แล้ว
}

// ===== ตัวอย่าง 2: รับ Copy type (i32) — ไม่ move =====
fn double_value(x: i32) -> i32 {
    x * 2
}

fn main() {
    let num = 42;
    let result = double_value(num); // num ถูก copy (ไม่ move)
    println!("num = {}, result = {}", num, result); // ✅ num ยังใช้ได้
}

// ===== ตัวอย่าง 3: เก็บ ownership ไว้ใน struct =====
struct Database {
    connection_string: String,
}

impl Database {
    fn new(conn: String) -> Self {
        Database {
            connection_string: conn, // ย้าย ownership เข้า struct
        }
    }
}

fn main() {
    let conn = String::from("postgres://localhost:5432/mydb");
    let db = Database::new(conn); // ownership ย้ายไป Database
    // println!("{}", conn);      // ❌ conn ถูก move แล้ว
}

// ===== ตัวอย่าง 4: Transform แล้วส่ง ownership กลับ =====
fn to_uppercase(s: String) -> String {
    s.to_uppercase() // สร้าง String ใหม่, s ถูก drop
}

fn main() {
    let name = String::from("alice");
    let upper = to_uppercase(name); // name ย้ายเข้า → upper ได้ค่าใหม่กลับมา
    println!("{}", upper); // "ALICE"
}
```

---

### 6.2 `mut x: T` — รับ Ownership + แก้ไข Local ได้

**ความหมาย**: ฟังก์ชันรับ ownership และ **สามารถแก้ไขค่า local copy** ได้ภายในฟังก์ชัน `mut` ตรงนี้เป็น **binding-level mutability** — แก้ไขตัวแปร local เท่านั้น ไม่ส่งผลกลับ caller

**บริบทที่ใช้**:
- รับ ownership มาแล้วต้อง **แก้ไขค่าก่อนใช้** ภายในฟังก์ชัน
- สร้าง **accumulator/builder** จากค่าที่รับมา

```rust
// ===== ตัวอย่าง 1: แก้ไข String ที่รับ ownership มา =====
fn append_greeting(mut s: String) -> String {
    s.push_str(", welcome!"); // ✅ แก้ไข s ได้เพราะประกาศ mut
    s                          // ส่ง ownership กลับ
}

fn main() {
    let name = String::from("Alice");
    let result = append_greeting(name); // name move เข้าไป
    println!("{}", result); // "Alice, welcome!"
}

// ===== ตัวอย่าง 2: mut x กับ Copy type =====
fn countdown(mut n: i32) {
    while n > 0 {
        println!("{}...", n);
        n -= 1; // แก้ไข local copy ได้
    }
    println!("Go!");
}

fn main() {
    let start = 5;
    countdown(start);
    println!("start ยังเป็น {}", start); // ✅ start ยังเป็น 5 (เพราะ Copy)
}

// ===== ตัวอย่าง 3: Builder Pattern =====
fn build_query(mut parts: Vec<String>) -> String {
    parts.sort();                        // แก้ไข Vec ได้
    parts.dedup();                       // ลบ duplicates
    parts.join(" AND ")                  // สร้าง query
}

fn main() {
    let conditions = vec![
        String::from("age > 18"),
        String::from("active = true"),
        String::from("age > 18"), // duplicate
    ];
    let query = build_query(conditions);
    println!("WHERE {}", query);
    // conditions ถูก move แล้ว ใช้ไม่ได้
}
```

> [!NOTE]
> **`mut x: T` vs `x: T`** — ทั้งคู่รับ ownership เหมือนกัน ต่างกันแค่ `mut` ทำให้ตัวแปร local แก้ไขค่าได้ `mut` เป็นแค่ **implementation detail** ของฟังก์ชัน — **caller ไม่จำเป็นต้องรู้** ว่า parameter เป็น `mut` หรือไม่

---

### 6.3 `x: &T` — Shared/Immutable Borrow (ยืมอ่าน)

**ความหมาย**: ฟังก์ชัน **ยืมค่ามาอ่าน** ไม่แก้ไข ไม่รับ ownership ตัวแปรเดิมยังใช้ได้

**บริบทที่ใช้**:
- ฟังก์ชันต้องการ **อ่านค่า** เท่านั้น
- ไม่ต้องการ ownership — ข้อมูลยัง belong to caller
- ใช้บ่อยที่สุดใน Rust — **เป็น default ที่ควรเลือกก่อน**

```rust
// ===== ตัวอย่าง 1: อ่านค่า String =====
fn print_length(s: &String) {
    println!("'{}' has {} characters", s, s.len());
    // s.push_str("!"); // ❌ ไม่สามารถแก้ไขผ่าน &T
}

fn main() {
    let msg = String::from("hello");
    print_length(&msg); // ส่ง reference (ยืม)
    print_length(&msg); // ✅ ยืมได้อีกกี่ครั้งก็ได้!
    println!("{}", msg); // ✅ msg ยังใช้ได้
}

// ===== ตัวอย่าง 2: ใช้ &str แทน &String (Idiomatic Rust) =====
fn print_length_better(s: &str) { // &str ยืดหยุ่นกว่า &String
    println!("'{}' has {} characters", s, s.len());
}

fn main() {
    let owned = String::from("hello");
    print_length_better(&owned);    // ✅ &String → &str (deref coercion)
    print_length_better("world");   // ✅ &str literal ก็ใช้ได้
}

// ===== ตัวอย่าง 3: คำนวณจาก borrowed data =====
fn total_score(scores: &Vec<i32>) -> i32 {
    scores.iter().sum()
}

fn average_score(scores: &Vec<i32>) -> f64 {
    let total: i32 = scores.iter().sum();
    total as f64 / scores.len() as f64
}

fn main() {
    let scores = vec![90, 85, 78, 92, 88];

    // ยืมได้หลายครั้ง หลายฟังก์ชัน
    let total = total_score(&scores);
    let avg = average_score(&scores);

    println!("Total: {}, Average: {:.1}", total, avg);
    println!("Scores: {:?}", scores); // ✅ ยังใช้ได้
}

// ===== ตัวอย่าง 4: ยืมหลายตัวพร้อมกัน =====
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}

fn main() {
    let s1 = String::from("hello");
    let s2 = String::from("hi");
    let result = longest(&s1, &s2);
    println!("Longest: {}", result);
}
```

> [!TIP]
> **Idiomatic Rust**: ใช้ `&str` แทน `&String`, ใช้ `&[T]` แทน `&Vec<T>` เพราะรับได้ทั้ง owned และ borrowed data (ผ่าน Deref coercion)

---

### 6.4 `x: &mut T` — Exclusive/Mutable Borrow (ยืมแก้ไข)

**ความหมาย**: ฟังก์ชัน **ยืมค่ามาแก้ไข** โดยไม่รับ ownership การแก้ไขจะ **ส่งผลกลับ caller**

**บริบทที่ใช้**:
- ฟังก์ชันต้อง **แก้ไขข้อมูลของ caller** โดยตรง
- ไม่ต้องการ ownership — แค่ต้องการแก้ไขชั่วคราว
- เหมือน pass by reference ในภาษาอื่น

```rust
// ===== ตัวอย่าง 1: แก้ไข String ผ่าน &mut =====
fn make_uppercase(s: &mut String) {
    *s = s.to_uppercase(); // แก้ไขค่าที่ s ชี้อยู่
}

fn main() {
    let mut name = String::from("alice");
    make_uppercase(&mut name); // ส่ง mutable reference
    println!("{}", name); // "ALICE" — ค่าเปลี่ยนแล้ว!
}

// ===== ตัวอย่าง 2: แก้ไข Vec =====
fn add_item(list: &mut Vec<String>, item: &str) {
    list.push(String::from(item));
}

fn remove_duplicates(list: &mut Vec<String>) {
    list.sort();
    list.dedup();
}

fn main() {
    let mut shopping_list = Vec::new();

    add_item(&mut shopping_list, "Milk");
    add_item(&mut shopping_list, "Bread");
    add_item(&mut shopping_list, "Milk"); // duplicate

    println!("Before: {:?}", shopping_list);
    // ["Milk", "Bread", "Milk"]

    remove_duplicates(&mut shopping_list);
    println!("After: {:?}", shopping_list);
    // ["Bread", "Milk"]
}

// ===== ตัวอย่าง 3: แก้ไข struct fields =====
struct BankAccount {
    owner: String,
    balance: f64,
}

fn deposit(account: &mut BankAccount, amount: f64) {
    account.balance += amount;
    println!("💰 Deposited {:.2}. New balance: {:.2}", amount, account.balance);
}

fn withdraw(account: &mut BankAccount, amount: f64) -> Result<(), String> {
    if amount > account.balance {
        Err(format!("Insufficient funds! Balance: {:.2}", account.balance))
    } else {
        account.balance -= amount;
        println!("💸 Withdrew {:.2}. New balance: {:.2}", amount, account.balance);
        Ok(())
    }
}

fn main() {
    let mut acct = BankAccount {
        owner: String::from("Alice"),
        balance: 1000.0,
    };

    deposit(&mut acct, 500.0);       // balance: 1500.0
    withdraw(&mut acct, 200.0).ok(); // balance: 1300.0
    println!("Final: {:.2}", acct.balance);
}

// ===== ตัวอย่าง 4: Swap สองค่า =====
fn swap_values(a: &mut i32, b: &mut i32) {
    let temp = *a;
    *a = *b;
    *b = temp;
}

fn main() {
    let mut x = 10;
    let mut y = 20;
    swap_values(&mut x, &mut y);
    println!("x = {}, y = {}", x, y); // x = 20, y = 10
}
```

---

### 6.5 `mut x: &T` — Mutable Binding ของ Immutable Reference

**ความหมาย**: ตัวแปร `x` สามารถ **เปลี่ยนไปชี้ reference อื่น** ได้ แต่ **ไม่สามารถแก้ไขค่า** ที่ reference ชี้อยู่

**บริบทที่ใช้**:
- ต้องการ **เลือก reference** ที่จะใช้ระหว่างการทำงาน
- Iterate ผ่าน references หลายตัว
- ค่อนข้างหายาก ไม่ค่อยใช้บ่อย

```rust
// ===== ตัวอย่าง 1: เปลี่ยน reference ที่ชี้ =====
fn find_longest<'a>(items: &'a [String]) -> &'a String {
    let mut longest: &String = &items[0]; // ชี้ไปตัวแรก

    for item in items.iter() {
        if item.len() > longest.len() {
            longest = item; // ✅ เปลี่ยนไปชี้ตัวอื่นได้
        }
        // longest.push_str("!"); // ❌ แก้ไขค่าที่ชี้ไม่ได้
    }

    longest
}

fn main() {
    let words = vec![
        String::from("short"),
        String::from("medium length"),
        String::from("the longest string here"),
    ];
    println!("Longest: {}", find_longest(&words));
}

// ===== ตัวอย่าง 2: เลือก config ตาม condition =====
fn select_config<'a>(
    mut config: &'a str, // สามารถเปลี่ยน reference ได้
    override_val: Option<&'a str>,
) -> &'a str {
    if let Some(val) = override_val {
        config = val; // ✅ เปลี่ยนไปชี้ค่าใหม่
    }
    config
}

fn main() {
    let default = "production";
    let result = select_config(default, Some("staging"));
    println!("Config: {}", result); // "staging"
}
```

---

### 6.6 `mut x: &mut T` — Mutable Binding ของ Mutable Reference

**ความหมาย**: ทั้ง **เปลี่ยน reference** ไปชี้ที่อื่นได้ **และแก้ไขค่า** ที่ reference ชี้อยู่ได้ — ยืดหยุ่นที่สุด

**บริบทที่ใช้**:
- ต้องการ **สลับไปแก้ไข target อื่น** ระหว่างการทำงาน
- **หายากมาก** ในทางปฏิบัติ

```rust
// ===== ตัวอย่าง 1: สลับ target ที่จะแก้ไข =====
fn fill_larger(a: &mut Vec<i32>, b: &mut Vec<i32>, value: i32) {
    // เลือก Vec ที่ใหญ่กว่า แล้วเพิ่ม value เข้าไป
    let mut target: &mut Vec<i32> = if a.len() >= b.len() { a } else { b };
    target.push(value);

    // สามารถเปลี่ยนไปชี้ตัวอื่นได้
    // (ในกรณีนี้ต้องระวัง borrowing rules)
}

// ===== ตัวอย่าง 2: Iterate แล้วแก้ไขแต่ละตัว =====
fn increment_all(values: &mut [i32]) {
    for mut val in values.iter_mut() {
        // val เป็น &mut i32
        // mut val ทำให้เราสามารถ reassign val ได้ (แม้ปกติไม่จำเป็น)
        *val += 1;
    }
}

fn main() {
    let mut nums = vec![1, 2, 3, 4, 5];
    increment_all(&mut nums);
    println!("{:?}", nums); // [2, 3, 4, 5, 6]
}
```

---

## 7. เปรียบเทียบทุกรูปแบบ Parameter

### 7.1 ตารางเปรียบเทียบ 6 แบบ

| รูปแบบ | Ownership | อ่านค่า | แก้ไขค่า | เปลี่ยน ref | caller ใช้ต่อได้ | ตัวอย่าง Signature |
|--------|-----------|---------|---------|------------|----------------|-------------------|
| `x: T` | ✅ ย้ายเข้า | ✅ | ❌ | — | ❌ (ถ้าไม่ Copy) | `fn eat(s: String)` |
| `mut x: T` | ✅ ย้ายเข้า | ✅ | ✅ (local) | — | ❌ (ถ้าไม่ Copy) | `fn build(mut v: Vec<i32>)` |
| `x: &T` | ❌ ยืม | ✅ | ❌ | ❌ | ✅ | `fn read(s: &str)` |
| `x: &mut T` | ❌ ยืม | ✅ | ✅ (caller's) | ❌ | ✅ | `fn write(s: &mut String)` |
| `mut x: &T` | ❌ ยืม | ✅ | ❌ | ✅ | ✅ | `fn scan(mut p: &str)` |
| `mut x: &mut T` | ❌ ยืม | ✅ | ✅ (caller's) | ✅ | ✅ | `fn update(mut p: &mut Vec<i32>)` |

### 7.2 ภาพรวม — เลือกแบบไหนดี?

```
เริ่มถามตัวเอง:
│
├─ ต้องการ ownership ไหม?
│   ├─ ✅ ต้องเป็นเจ้าของ → ต้องแก้ไข local? 
│   │   ├─ ✅ → mut x: T
│   │   └─ ❌ → x: T
│   │
│   └─ ❌ แค่ยืม → ต้องแก้ไขค่าของ caller ไหม?
│       ├─ ✅ → x: &mut T
│       └─ ❌ → x: &T  ← ⭐ เลือกอันนี้เป็นค่า default
```

> [!IMPORTANT]
> **Rule of Thumb**: เริ่มจาก `&T` ก่อนเสมอ แล้วค่อย "upgrade" ไปเป็น `&mut T` หรือ `T` เมื่อจำเป็น

### 7.3 ตัวอย่างรวม — ใช้ทุกแบบใน Struct เดียว

```rust
#[derive(Debug, Clone)]
struct TodoList {
    name: String,
    items: Vec<String>,
}

impl TodoList {
    // x: T — รับ ownership เพื่อเก็บใน struct
    fn new(name: String) -> Self {
        TodoList {
            name,
            items: Vec::new(),
        }
    }

    // x: &T — ยืมอ่านอย่างเดียว
    fn contains(&self, item: &str) -> bool {
        self.items.iter().any(|i| i == item)
    }

    // x: &mut T — ยืมแก้ไข (self)
    fn add_item(&mut self, item: &str) {
        if !self.contains(item) {
            self.items.push(String::from(item));
        }
    }

    // mut x: T — รับ ownership + แก้ไข local
    fn merge(mut self, other: TodoList) -> TodoList {
        for item in other.items {
            if !self.items.contains(&item) {
                self.items.push(item);
            }
        }
        self
    }

    // x: &T — ยืมอ่าน เพื่อ display
    fn summary(&self) -> String {
        format!("{}: {} items", self.name, self.items.len())
    }
}

fn main() {
    let mut work = TodoList::new(String::from("Work")); // new() รับ ownership
    work.add_item("Write report");                       // add_item() ยืมแก้ไข &mut self
    work.add_item("Review PR");

    let mut personal = TodoList::new(String::from("Personal"));
    personal.add_item("Buy groceries");
    personal.add_item("Exercise");

    println!("{}", work.summary());     // summary() ยืมอ่าน &self
    println!("{}", personal.summary());

    // merge() รับ ownership ของทั้งสองตัว
    let combined = work.merge(personal);
    println!("Combined: {:?}", combined);

    // work และ personal ถูก move แล้ว ใช้ไม่ได้
}
```

---

## 8. Lifetime กับ Borrowing

เมื่อฟังก์ชัน **return reference** Rust ต้องรู้ว่า reference นั้นจะ **อยู่ได้นานแค่ไหน** — นี่คือ **Lifetime**

```rust
// ❌ ไม่ระบุ lifetime — Compile Error!
// fn first_word(s: &str) -> &str { ... }
// Rust ไม่รู้ว่า return reference มาจากไหน

// ✅ ระบุ lifetime 'a — return reference มีอายุเท่ากับ input
fn first_word<'a>(s: &'a str) -> &'a str {
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    s
}

// ✅ Lifetime Elision — Rust สามารถ infer lifetime ได้ในหลายกรณี
// กฎ: ถ้ามี input reference ตัวเดียว → output lifetime = input lifetime
fn first_word_elided(s: &str) -> &str { // ไม่ต้องเขียน 'a เอง
    let bytes = s.as_bytes();
    for (i, &byte) in bytes.iter().enumerate() {
        if byte == b' ' {
            return &s[..i];
        }
    }
    s
}

fn main() {
    let sentence = String::from("hello world");
    let word = first_word(&sentence);
    println!("First word: {}", word); // "hello"
}
```

### Lifetime กับหลาย References

```rust
// ต้องระบุ lifetime เมื่อมีหลาย input references
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str {
    if s1.len() > s2.len() { s1 } else { s2 }
}

// Lifetime ใน struct — struct ที่เก็บ reference
struct Excerpt<'a> {
    text: &'a str, // ← reference ต้องอยู่ได้นานอย่างน้อยเท่า struct
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence;
    {
        let excerpt = Excerpt {
            text: &novel[..16], // ยืม slice จาก novel
        };
        first_sentence = excerpt.text;
    }
    // ✅ OK เพราะ novel ยังอยู่
    println!("First sentence: {}", first_sentence);
}
```

---

## 9. สรุปตาราง Cheat Sheet

### เลือกวิธีส่ง Parameter

```
┌────────────────────────────────────────────────────────────────┐
│                     Parameter Passing Cheat Sheet              │
├──────────────┬─────────────┬──────────────┬────────────────────┤
│ Pattern      │ ฟังก์ชัน    │ Caller       │ ใช้เมื่อ            │
├──────────────┼─────────────┼──────────────┼────────────────────┤
│ x: T         │ เป็นเจ้าของ   │ ใช้ต่อ ❌     │ เก็บค่า, ส่งต่อ,    │
│              │ อ่านได้       │              │ transform           │
├──────────────┼─────────────┼──────────────┼────────────────────┤
│ mut x: T     │ เป็นเจ้าของ   │ ใช้ต่อ ❌     │ แก้ไข + เก็บ/ส่งต่อ  │
│              │ อ่าน+เขียน    │              │                     │
├──────────────┼─────────────┼──────────────┼────────────────────┤
│ x: &T        │ ยืมอ่าน      │ ใช้ต่อ ✅     │ ⭐ DEFAULT!        │
│              │              │              │ ดูค่า, คำนวณ         │
├──────────────┼─────────────┼──────────────┼────────────────────┤
│ x: &mut T    │ ยืมแก้ไข     │ ใช้ต่อ ✅     │ แก้ไขข้อมูล caller   │
│              │              │ (ค่าเปลี่ยน)  │ in-place             │
├──────────────┼─────────────┼──────────────┼────────────────────┤
│ mut x: &T    │ เปลี่ยน ref   │ ใช้ต่อ ✅     │ หายาก:              │
│              │ อ่านค่าเท่านั้น│              │ เลือก ref ระหว่างทาง  │
├──────────────┼─────────────┼──────────────┼────────────────────┤
│ mut x: &mut T│ เปลี่ยน ref   │ ใช้ต่อ ✅     │ หายากมาก:           │
│              │ + แก้ไขค่า    │ (ค่าเปลี่ยน)  │ สลับ target + แก้ไข  │
└──────────────┴─────────────┴──────────────┴────────────────────┘
```

### Move vs Clone vs Copy

```
┌─────────────────────────────────────────────────────────┐
│              Move vs Clone vs Copy                       │
├───────────┬──────────┬──────────────┬────────────────────┤
│           │ Move     │ Clone        │ Copy               │
├───────────┼──────────┼──────────────┼────────────────────┤
│ ต้นทุน     │ O(1)     │ O(n) แพง!    │ O(1)               │
│ Explicit  │ อัตโนมัติ  │ .clone()     │ อัตโนมัติ            │
│ ตัวเดิม    │ ❌ ตาย    │ ✅ อยู่       │ ✅ อยู่              │
│ ใช้กับ     │ Heap type│ ทุก Clone    │ Stack-only types   │
│ ตัวอย่าง   │ String   │ String       │ i32, f64, bool     │
└───────────┴──────────┴──────────────┴────────────────────┘
```

---

## 10. Anti-patterns และ Common Mistakes

### ❌ Anti-pattern 1: Clone ทุกที่เพื่อหนี Borrow Checker

```rust
// ❌ BAD — clone เยอะเกินไป
fn process(data: &Vec<String>) {
    let data_clone = data.clone(); // ทำไมต้อง clone?
    for item in data_clone.iter() {
        println!("{}", item);
    }
}

// ✅ GOOD — ยืมอ่านก็พอ
fn process(data: &[String]) {
    for item in data.iter() {
        println!("{}", item);
    }
}
```

### ❌ Anti-pattern 2: ใช้ `&String` แทน `&str`

```rust
// ❌ BAD — จำกัดเกินไป
fn greet(name: &String) {
    println!("Hello, {}!", name);
}

// ✅ GOOD — รับได้ทั้ง String และ &str
fn greet(name: &str) {
    println!("Hello, {}!", name);
}

fn main() {
    let owned = String::from("Alice");
    greet(&owned);    // ✅ ทั้ง &String (deref → &str)
    greet("Bob");     // ✅ ทั้ง &str literal
}
```

### ❌ Anti-pattern 3: รับ Ownership โดยไม่จำเป็น

```rust
// ❌ BAD — ทำไมต้อง own String ทั้งที่แค่อ่าน?
fn count_words(s: String) -> usize {
    s.split_whitespace().count()
}

// ✅ GOOD — ยืมอ่านก็พอ
fn count_words(s: &str) -> usize {
    s.split_whitespace().count()
}
```

### ❌ Anti-pattern 4: ใช้ `&Vec<T>` แทน `&[T]`

```rust
// ❌ BAD
fn sum(numbers: &Vec<i32>) -> i32 {
    numbers.iter().sum()
}

// ✅ GOOD — รับได้ทั้ง Vec, array, และ slice
fn sum(numbers: &[i32]) -> i32 {
    numbers.iter().sum()
}

fn main() {
    let v = vec![1, 2, 3];
    let a = [4, 5, 6];

    sum(&v);       // ✅ Vec
    sum(&a);       // ✅ Array
    sum(&v[0..2]); // ✅ Slice
}
```

---

## บทสรุป

```
🎯 หลักการเลือก Parameter ใน Rust:

1. เริ่มจาก &T (shared borrow) เสมอ
2. ถ้าต้องแก้ไข → &mut T
3. ถ้าต้องเป็นเจ้าของ → T
4. ใช้ &str แทน &String, ใช้ &[T] แทน &Vec<T>
5. Clone เมื่อจำเป็นจริงๆ เท่านั้น
6. ใช้ Copy สำหรับ type ขนาดเล็กบน Stack
```

> [!TIP]
> Rust Ownership ดูซับซ้อนตอนแรก แต่เมื่อเข้าใจแล้ว จะช่วยให้เขียนโค้ดที่ **ปลอดภัย, เร็ว, และไม่มี data race** โดยไม่ต้องพึ่ง Garbage Collector 🦀
