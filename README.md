# Weintek — Lidar Projesi

<p align="center">
  <img src="https://github.com/kenann/Weintek---Lidar/blob/main/uploadFile/image/wein-lidar.jpg?raw=true" width="450" style="border-radius: 10px; border: 1px solid #ccc;">
</p>

## 📘 Proje Hakkında

Bu projemde **Weintek HMI** üzerindeki seri port kullanımı,  
**Makro Free Protocol** uygulaması ve hafızada belirli bir bölge ayırma gibi örnekler bulunmaktadır.

---

### 📷 Proje Görselleri

<table align="center">
  <tr>
    <td align="center" width="50%">
      <img src="https://github.com/kenann/Weintek---Lidar/blob/main/uploadFile/image/weintech-lidar.jpg?raw=true" width="300" style="border-radius: 10px; border: 1px solid #ccc;"><br>
      <b>Weintek HMI Uygulaması</b>
    </td>
    <td align="center" width="50%">
      <img src="https://github.com/kenann/Weintek---Lidar/blob/main/uploadFile/image/coron-lidar.jpg?raw=true" width="300" style="border-radius: 10px; border: 1px solid #ccc;"><br>
      <b>Coron Lidar Cihazı</b>
    </td>
  </tr>
</table>

---

### ⚙️ Ana Başlıklar
- 🖥️ Weintek HMI üzerinde seri port haberleşmesi  
- Makro ile Free Protocol uygulaması  
- Hafızada veri bölgesi yönetimi  
- 🌐 Coron Lidar protokol çözümleme ve haberleşme
```  
┌───────────────┐
│ 🌐 GDS-F31     │
│   LIDAR       │
└──────┬────────┘
       │ RS232 / 485
┌──────▼────────┐
│ 🖥️ Weintek HMI │
│  Macro: Parse │
│  Rotate Obj   │
└──────┬────────┘
       │ Görsel
┌──────▼────────┐
│ 👤 Operatör    │
│    Ekranı     │
└───────────────┘
``` 
---

### 🧩 Kullanılan Ürünler

| Ürün Adı | Model | Açıklama |
|-----------|--------|----------|
| **HMI Panel** | Weintek cMT3162X | Seri port haberleşmesi ve makro kontrolü için kullanıldı |
| **Lidar Sensör** | Coron GDS-F31 | Mesafe ölçüm ve alan algılama sensörü |
| **Bağlantı Kablosu** | Serial Port Connector | Lidar ile HMI arasındaki RS232 haberleşme bağlantısı |

----
✅ PROJE ÖZETİ
Weintek HMI’de Lidar Verisi Okuma ve Grafiksel Gösterim
----

📌 1️⃣ GDS-F31 Lidar Sensörü RS232/RS485 ile HMI'ya Bağlandı

Veri, "Free Protocol Client" ile okundu ve "LW" adreslerine yazıldı.

Makro:
✔ Paket başlığı kontrolü
✔ Paket boyu doğrulama
✔ Hatalı paket işaretleme
✔ Her bir byte’ın LW’ye işlenmesi

📌 2️⃣ Veri Formatı (Protokol) Çözüldü

### 🧩 Cihazdan gelen paket:

| Alan	| Byte	| Açıklama |
|-----------|--------|----------|
| **Frame Header** |	0–3 |	AA 88 88 AA |
| **Main CMD** |	4 |	D5 |
| **Sub CMD** |	5 |	0E |
| **Data Length** |	6–9 |	Toplam uzunluk |
| **Device Status** |	10 |	0 = normal |
| **Layer Trigger States** |	12–14 |	0,1,2 (tetik durumu) |
| **Açı ve Mesafe** |	15–26 |	3 farklı katmanda |
| **XOR Check** |	27 |	Kontrol |
| **Frame Footer** |	28–31 |	88 AA AA 88 |


📌 3️⃣ Açı ve Mesafe Çözümleme

GDS-F31 LiDAR, açı ve mesafe bilgilerini **Big-Endian** formatında gönderir.
Bunları ayrıştırıp anlamlı **fiziksel değerlere** ( ° ve cm ) dönüştürüyoruz.

```txt
### Veri Yapısı (Bayt Eşlemesi)

| Bytes | Layer | Value Type | Conversion | Description |
|-------|-------|------------|------------|-------------|
| 15-16 | Outer  | Angle     | Angle = RawValue / 100 | 0.01° resolution |
| 17-18 | Outer  | Distance  | Distance = RawValue    | cm |
| 19-20 | Middle | Angle     | Angle = RawValue / 100 |  |
| 21-22 | Middle | Distance  | Distance = RawValue    | cm |
| 23-24 | Inner  | Angle     | Angle = RawValue / 100 |  |
| 25-26 | Inner  | Distance  | Distance = RawValue    | cm |

---
### ✅ Örnek HEX
Bytes 15: 0x11 - Bytes 16: 0x88 → ** AngleRaw ** = 0x1188
Bytes 17: 0x00 - Bytes 18: 0xAE → ** DistRaw **  = 0x00AE
Convert:
Angle = 0x1188 → 44936 / 100 = 449.36° → ~89.36° (wrapped)
Distance = 0x00AE = 174 cm

### Negatif İşaretli Dönüştürme (Negatif Açılar)

> ⚠ LiDAR Açı değeri ** negatif açıları ** temsil edebilir (örneğin, 0xEE99 ≈ -24,46°)

Açı değeri > 0x8000 ise:

SignedAngle = RawValue - 65536
Derece = (SignedAngle / 32768) * 180

### ✅ Örnek HEX
RawValue = 0xEE99 → 61081 decimal
SignedAngle = 61081 - 65536 = -4455
Degree = -4455 / 32768 * 180 ≈ -24.46°

           ┌──── Outer Layer
           │
  (-45°)   ▼        (0°)
            \   |
             \  |
              \ |
LIDAR  ●-------●----------→ X
              /|
             / |
            /  |
           ▲   |
        (-90°)  Middle & Inner Layers

```

📌 4️⃣ HMI Üzerinde Hex Değerleri ASCII Gösterildi

HEX2ASCII kullanıldı:
HEX2ASCII(wResponse[i], result[j], 2)

Gösterim:
| Emoji | Açıklama                                   |
| ----- | ------------------------------------------ |
| 🔵    | Frame Header / Paket Başlangıcı veya Sonu  |
| 🟢    | Main Command (ölçüm verisi)                |
| 🟠    | Sub Command / Alt Komut                    |
| ⚪     | Boş / Dolgu / Veri uzunluğu / Devam byte’ı |
| 🟡    | Trigger / Çalışma alanı bilgisi            |
| 🟣    | Ölçüm verisi (Outer/Middle/Inner)          |
| 🔴    | Checksum / Doğrulama                       |

| 0     | 1     | 2     | 3     | 4     | 5     | 6    | 7    |
| ----- | ----- | ----- | ----- | ----- | ----- | ---- | ---- |
| 🔵 aa | 🔵 88 | 🔵 88 | 🔵 aa | 🟢 d5 | 🟠 0e | ⚪ 00 | ⚪ 00 |

| 8    | 9    | 10   | 11   | 12    | 13    | 14    | 15    |
| ---- | ---- | ---- | ---- | ----- | ----- | ----- | ----- |
| ⚪ 00 | ⚪ 20 | ⚪ 00 | ⚪ 00 | 🟡 02 | 🟡 01 | 🟡 01 | 🟣 21 |

| 16    | 17   | 18    | 19   | 20   | 21   | 22   | 23   |
| ----- | ---- | ----- | ---- | ---- | ---- | ---- | ---- |
| 🟣 84 | ⚪ 00 | 🟣 ae | ⚪ 00 | ⚪ 00 | ⚪ 00 | ⚪ 00 | ⚪ 00 |

| 24   | 25   | 26    | 27    | 28    | 29    | 30    | 31 |
| ---- | ---- | ----- | ----- | ----- | ----- | ----- | -- |
| ⚪ 00 | ⚪ 00 | 🔴 f2 | 🔵 88 | 🔵 aa | 🔵 aa | 🔵 88 | —  |


⚠ “index out of range” hataları düzeltildi.

📌 5️⃣ XOR Kontrol

Byte 4–26 arasındaki tüm veriler
XORSUM ile doğrulandı:
XORSUM(wResponse[4], xor_val, 23)

Yanlışsa hata bayrağı → ekran uyarı.

📌 6️⃣ Açı ile Nesne Döndürme

### Rotate w/ Scaling Calculation

- **Data** → LIDAR’dan gelen açı bilgisi (int16 çözülmüş)
- **InputLow / InputHigh** → Verinin kapsadığı min-max açı değerleri
- **ScalingLow / ScalingHigh** → Rotate kontrolünün dönme aralığı (örn: -45° → 225°)

$$
Angle = (Data - InputLow) 
\times \frac{ScalingHigh - ScalingLow}{InputHigh - InputLow} + ScalingLow
$$

📌 7️⃣ Açı ve Trigger Durumuna Göre Renk Uyarıları
Durum	Şart	Gösterge Rengi
Normal	> 50 cm	🟢 Yeşil
Uyarı	20–50 cm	🟡 Sarı
Tehlike (Trigger)	< 20 cm	🔴 Kırmızı

Bu renk değişimi Object Status kontrolünden otomatik olarak yapılmaktadır.
sı

---
### 🔗 Etiketler
`Weintek` `Lidar` `Serial Port` `Macro` `Automation` `Coron`

---

📫 Yeni projelerde birlikte çalışmak istersen:  
**[LinkedIn profilime göz atabilirsin](https://www.linkedin.com/in/kenan-r-5ab319115/)** 🚀
