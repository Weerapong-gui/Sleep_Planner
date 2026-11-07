# Sleep Planner

โปรเจกต์กลุ่ม วิชา Mathematics for Engineering 2 (1501206)

คำนวณช่วงเวลาที่เหมาะสมในการดื่มกาแฟ เพื่อให้คาเฟอีนที่เหลือในร่างกายลดลงจนไม่รบกวนการนอน

## ไฟล์ในโปรเจกต์

| ไฟล์ | เนื้อหา |
|---|---|
| `Sleep_Planner.ipynb` | **ไฟล์หลักที่ใช้ส่งและ demo** |
| `Formula.ipynb` | โค้ดต้นฉบับ (ก่อนแยกเป็นฟังก์ชัน) |
| `GUI.ipynb` | ตัวอย่างการคำนวณหลายแก้ว |

## ติดตั้งครั้งแรก (ทุกคนต้องทำ)

```bash
git clone https://github.com/Weerapong-gui/Sleep_Planner.git
cd Sleep_Planner
python3 -m pip install --user nbstripout
python3 -m nbstripout --install
```

`nbstripout` จะลบ output ของ notebook ออกก่อน commit ทำให้ diff อ่านง่ายและ merge ชนกันน้อยลง
แต่ละคนต้องรัน `python3 -m nbstripout --install` เองในเครื่องตัวเอง (git ไม่ให้ตั้งค่านี้ผ่าน repo)

ตรวจว่าติดตั้งแล้ว:

```bash
python3 -m nbstripout --status
```

## ขั้นตอนทำงานประจำวัน

```bash
git pull                          # ดึงงานล่าสุดก่อนเริ่ม
git switch -c ชื่อ/งานที่ทำ          # แยก branch
# ... แก้ notebook แล้ว Save ...
git add Sleep_Planner.ipynb
git commit -m "อธิบายว่าแก้อะไร"
git push -u origin ชื่อ/งานที่ทำ     # แล้วเปิด Pull Request บน GitHub
```

- แบ่งกันแก้คนละเซลล์ ไม่แก้เซลล์เดียวกันพร้อมกัน
- ก่อนส่ง ให้รัน **Kernel → Restart Kernel and Run All Cells** ต้องไม่มี error

## Library ที่ใช้

`math`, `datetime`, `numpy`, `matplotlib` (มีมาให้แล้วใน Anaconda)
