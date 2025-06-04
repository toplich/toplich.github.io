# Налаштування кластера з vSAN у vCenter 8


> У цій статті я покажу, як створити повноцінний кластер у vCenter 8 з увімкненими функціями **DRS**, **HA**, **vSAN**, налаштувати **мережу через Distributed Switch**, додати **Witness Host**, оновити хости офлайн і створити **VM Storage Policy** для віртуальних машин.

---

## 🔹 1. Створення кластера

1. У vCenter:
   - Правою кнопкою на **Datacenter** → `New Cluster`.
2. Назвіть кластер.
3. Увімкніть:
   - ✅ DRS (Distributed Resource Scheduler)
   - ✅ HA (High Availability)
   - ✅ vSAN (Virtual SAN)

---

## 🔹 2. Додавання гіпервізорів (ESXi)

- Якщо хости нові — додаємо до кластеру.
- Якщо вже є віртуальні машини — переносимо у кластер через `Move to...`.

---

## 🔹 3. Налаштування мережі (Distributed Switch)

### ✅ Створення vDS

- `Datacenter` → `Create Distributed Switch`
- Вкажіть кількість uplinks (залежить від кількості зовнішніх підключень).
- Назва: `vDS_Main`.

### 🌐 Port Groups

Створити принаймні:

- `vDS_WAN` – зовнішня мережа
- `vDS_LAN` – внутрішня мережа

### 📥 Призначення хостів до vDS

- Прив’язати NIC до uplinks на кожному хості.

### 🧠 VMkernel налаштування

Для кожного хоста:

| Port Group | Призначення | IP | Gateway | Використання |
|------------|-------------|----|---------|--------------|
| `vDS_WAN`  | Зовнішня     | так | так     | Management |
| `vDS_LAN`  | Внутрішня    | так | ні      | vMotion, vSAN |

---

## 🔹 4. Підготовка до vSAN

- Мінімум 3 хости (або 2 + Witness)
- Диски:
  - Cache Tier: SSD або NVMe
  - Capacity Tier: HDD або RAID0
- Співвідношення SSD до HDD: **1:7**
- Хости мають бути на однаковій версії ESXi
- Розділіть трафік (vSAN, vMotion, Management) через VLAN або фізичні інтерфейси

---

## 🔹 5. Додавання Witness Host

1. Завантажити **vSAN Witness Appliance OVF**
2. Установити через `Deploy OVF Template`
3. **Не додавати в кластер**, залишити в Datacenter
4. Налаштувати VMkernel:
   - окремий для доступу до vCenter
   - окремий для vSAN

---

## 🔹 6. Оновлення хостів через Lifecycle Manager (офлайн)

1. Завантажити ZIP (Image Depot) з сайту VMware
2. У `Lifecycle Manager`:
   - `Action` → `Import Updates`
3. У `Inventory` → `Host` → `Updates` → `Image`:
   - Вибрати імпортований образ
   - Перевірити Vendor Addon, Firmware & Drivers
4. Запустити:
   - `Image Compliance`
   - `Remediate`

---

## 🔹 7. Створення vSAN кластеру

1. `Cluster` → `Configure` → `vSAN` → `Services`
2. Тип:
   - `vSAN HCI` – Compute + Storage
   - `Two Node vSAN` або `Single site`
   - Увімкнути `vSAN ESA`, якщо доступно
3. Налаштування:
   - ✅ Allow reduced redundancy
   - ❌ Compression/Deduplication
   - ❌ Encryption
4. Клеймити диски:
   - Cache Tier: SSD
   - Capacity Tier: HDD
5. Вказати Preferred/Secondary домени
6. Призначити Witness та його диски

---

## 🔹 8. Якщо vSAN створено на 1 хості

> Лише для тесту!

- Availability → `No data redundancy`

---

## 🔹 9. Створення VM Storage Policy

1. `Policies and Profiles` → `VM Storage Policies` → `New`
2. Назва: `vSAN-NoRedundancy`
3. Правила:

```text
Availability: No data redundancy
Space efficiency: No preference
Encryption: No preference
Advanced:
- Disk stripes per object: 1
- IOPS limit: 0
- Object reservation: Thin provisioning
- Flash cache reservation: 0%
- Checksum: Enabled
- Force provisioning: Yes
```

🔹 10. Міграція віртуальних машин

Виберіть vSAN Datastore
Встановіть створену політику зберігання

✅ Результат

Кластер з повним набором можливостей (DRS + HA + vSAN)
Мережа організована через vDS з розділенням трафіку
Готовність до масштабування та fault tolerance



