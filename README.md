# สรุปรูปแบบ Function Parameter ใน Rust (Ownership & Borrowing)

เอกสารนี้สรุปความเข้าใจเกี่ยวกับการส่ง Parameter เข้าฟังก์ชันในภาษา Rust ซึ่งมีการจัดการเรื่อง Ownership และ Borrowing ที่เข้มงวด โดยแบ่งออกเป็น 6 รูปแบบหลัก

---

## 1. ตารางเปรียบเทียบและตัวอย่างการใช้งานจริง (Real-World Examples)

| รูปแบบ | Ownership | การอ่าน | การแก้ไข | เปลี่ยน Ref | Caller ใช้ต่อได้? | ตัวอย่างฟังก์ชันในโลกจริง (Real-World Use Case) |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- |
| `x: T` | ✅ ย้ายเข้า | ✅ | ❌ | - | ❌ | **`std::fs::remove_file(path: PathBuf)`**<br>ฟังก์ชันที่รับข้อมูลไปใช้งานเป็นครั้งสุดท้ายแล้วทำลายทิ้ง หรือส่งข้อมูลข้าม Thread |
| `mut x: T` | ✅ ย้ายเข้า | ✅ | ✅ (local) | - | ❌ | **`Builder Pattern (เช่น reqwest::RequestBuilder)`**<br>รับ Struct ของตัวเองเข้ามา ดัดแปลงค่าภายในฟังก์ชัน แล้ว Return ค่ากลับไป |
| `x: &T` | ❌ ยืม | ✅ | ❌ | ❌ | ✅ | **`std::fs::read_to_string(path: &Path)`**<br>ฟังก์ชันที่ต้องการแค่อ่านข้อมูลไปประมวลผล (เช่น ปริ้นค่า, อ่านไฟล์) โดยไม่เปลี่ยนแปลงข้อมูลต้นทาง |
| `x: &mut T` | ❌ ยืม | ✅ | ✅ (caller) | ❌ | ✅ | **`std::io::Read::read(&mut self, buf: &mut [u8])`**<br>ฟังก์ชันที่ต้องการเขียนข้อมูลอัปเดตลงใน Buffer หรืออัปเดตสถานะ (State) ของ Object เดิม |
| `mut x: &T` | ❌ ยืม | ✅ | ❌ | ✅ | ✅ | **`Text Parser / Tokenizer`**<br>การเขียนฟังก์ชันสแกนข้อความที่ตัวแปร `x` (pointer) ต้องถูกเลื่อนไปชี้ที่ตำแหน่งถัดไปเรื่อยๆ (`x = &x[1..]`) |
| `mut x: &mut T`| ❌ ยืม | ✅ | ✅ (caller) | ✅ | ✅ | **`Tree/Graph Traversal`**<br>การเดินไปตาม Node ในโครงสร้างข้อมูล และสามารถอัปเดตค่าใน Node นั้นๆ ได้ รวมถึงเลื่อน Pointer ไป Node อื่นได้ |

*(หมายเหตุ: ถ้า `T` มีการ Implement `Copy` Trait เช่น `i32`, `bool` ตัว Caller จะยังคงใช้งานได้เสมอเพราะเป็นการทำสำเนา (Copy) ไม่ใช่การย้าย (Move))*

---

## 2. รายละเอียดและโค้ดตัวอย่าง 6 รูปแบบ (Simple Examples)

### 2.1 `x: T` (Move & Immutable)
เป็นการย้าย (Move) เจ้าของเข้ามาในฟังก์ชัน ใช้อ่านได้อย่างเดียว แก้ไขไม่ได้
```rust
fn eat(s: String) {
    println!("1. อ่านค่า: {}", s);
    // s.push_str("!"); // ❌ Error: แก้ไขไม่ได้
}

fn main() {
    let s = String::from("hello");
    eat(s);
    // println!("{}", s); // ❌ Error: Caller ใช้ต่อไม่ได้ เพราะถูกย้าย Ownership ไปแล้ว
}
```

### 2.2 `mut x: T` (Move & Mutable)
เป็นการย้ายเจ้าของเข้ามา แต่ให้สิทธิ์แก้ไขค่าได้ (การแก้ไขจะมีผลแค่ในฟังก์ชันนี้เท่านั้น)
```rust
fn build(mut v: Vec<i32>) {
    v.push(1); // ✅ แก้ไขได้
    println!("2. ค่าในฟังก์ชัน: {:?}", v);
}

fn main() {
    let v = vec![10, 20];
    build(v);
    // println!("{:?}", v); // ❌ Error: Caller ใช้ต่อไม่ได้แล้ว
}
```

### 2.3 `x: &T` (Immutable Borrow)
ยืมมาอ่านได้อย่างเดียว แก้ไขข้อมูลของ Caller ไม่ได้ และเปลี่ยนให้ `x` ไปชี้ที่อื่นก็ไม่ได้
```rust
fn read(s: &str) {
    println!("3. อ่านค่า: {}", s);
    // s = "new"; // ❌ Error: เปลี่ยน ref ไม่ได้
}

fn main() {
    let s = String::from("hello");
    read(&s);
    println!("Caller ยังใช้ต่อได้: {}", s); // ✅
}
```

### 2.4 `x: &mut T` (Mutable Borrow)
ยืมมาเพื่อแก้ไขข้อมูลต้นฉบับของ Caller ได้ แต่ตัวแปร `x` เปลี่ยนไปชี้ที่อื่นไม่ได้
```rust
fn write(s: &mut String) {
    s.push_str(" world!"); // ✅ แก้ไขข้อมูลของ Caller
    // s = &mut String::from("new"); // ❌ Error: เปลี่ยน ref ไม่ได้
}

fn main() {
    let mut s = String::from("hello");
    write(&mut s);
    println!("ค่าของ Caller เปลี่ยนเป็น: {}", s); // ✅ ผลลัพธ์: hello world!
}
```

### 2.5 `mut x: &T` (Mutable Reference to Immutable Data)
ยืมมาอ่านอย่างเดียว (แก้ต้นฉบับไม่ได้) **แต่ตัว `x` เองสามารถเปลี่ยนไปชี้ที่ข้อมูลตัวอื่นได้** ภายในฟังก์ชัน
```rust
fn scan(mut p: &str) {
    println!("5. ค่าเดิม: {}", p);
    p = "new reference"; // ✅ เปลี่ยนให้ p ไปชี้ข้อความอื่นได้
    println!("5. เปลี่ยน ref เป็น: {}", p);
}

fn main() {
    let s = String::from("hello");
    scan(&s);
    println!("Caller ได้ค่าเดิม: {}", s); // ✅
}
```

### 2.6 `mut x: &mut T` (Mutable Reference to Mutable Data)
ทำได้ทุกอย่าง: ยืมมาแก้ไขข้อมูลต้นฉบับของ Caller ก็ได้ และตัว `x` เองก็เปลี่ยนไปชี้ที่ข้อมูลตัวอื่น (ชั่วคราว) ได้ด้วย
```rust
fn update(mut p: &mut Vec<i32>) {
    p.push(99); // ✅ แก้ไขข้อมูลของ Caller
    
    let mut another_vec = vec![1, 2, 3];
    p = &mut another_vec; // ✅ เปลี่ยน ref ไปชี้ตัวอื่น
    p.push(4); // แก้ไขข้อมูลตัวใหม่ (ไม่เกี่ยวกับ Caller แล้ว)
}

fn main() {
    let mut v = vec![10, 20];
    update(&mut v);
    println!("Caller ถูกแก้เป็น: {:?}", v); // ✅ ผลลัพธ์: [10, 20, 99]
}
```