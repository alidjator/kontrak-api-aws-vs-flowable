# Kontrak API — Konsolidasi (Proses "Approval Berita Acara")

Dokumen ini menggabungkan dua kontrak API terpisah menjadi satu berkas —
[KONTRAK-API-FLOWABLE.md](./KONTRAK-API-FLOWABLE.md) (endpoint yang
**sudah tersedia** dari Flowable) dan
[KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md) (endpoint
yang **harus dibangun** Aplikasi AWS) — supaya seluruh kontrak API untuk
proses "Approval Berita Acara" bisa dibaca dari satu tempat. Kedua
dokumen asal TETAP ADA dan tetap jadi rujukan utama masing-masing topik;
dokumen ini adalah salinan konsolidasi untuk kenyamanan membaca, bukan
pengganti.

Penomoran bagian di bawah **TIDAK diubah** dari dokumen asalnya masing-masing
(Bagian I memakai §1–§6, Bagian II memakai §1–§5 — dua sistem penomoran
terpisah) supaya isi & rujukan silang yang sudah diverifikasi di kedua
dokumen asal tetap akurat tanpa risiko salah nomor akibat rename ulang.
Untuk membedakan, rujukan yang menyeberang Bagian selalu ditandai eksplisit
(mis. "§3 Bagian I", "§4 Bagian II") — rujukan tanpa tanda Bagian berarti
menunjuk ke bagian yang sedang dibaca.

## Daftar Isi

**[BAGIAN I — Endpoint yang SUDAH TERSEDIA dari Flowable](#bagian-i--endpoint-yang-sudah-tersedia-dari-flowable)**

1. [Ringkasan — alur end-to-end](#1-ringkasan--alur-end-to-end)
2. [Endpoint Start Instance](#2-endpoint-start-instance)
   - 2.1. [Variabel proses yang wajib dikirim](#21-variabel-proses-yang-wajib-dikirim)
   - 2.2. [Format `dynamicApproverGroups`](#22-format-dynamicapprovergroups)
   - 2.3. [Contoh request & respons](#23-contoh-request--respons)
   - 2.4. [Keputusan DMN internal & prasyarat deployment](#24-keputusan-dmn-internal--prasyarat-deployment)
3. [Endpoint PULL-Polling Task](#3-endpoint-pull-polling-task)
   - 3.1. [Parameter query](#31-parameter-query)
   - 3.2. [Variabel proses tambahan (`includeProcessVariables`)](#32-variabel-proses-tambahan-includeprocessvariables)
   - 3.3. [Contoh request & respons](#33-contoh-request--respons)
4. [Endpoint Complete Task](#4-endpoint-complete-task)
   - 4.1. [Variabel yang wajib dikirim saat complete](#41-variabel-yang-wajib-dikirim-saat-complete)
   - 4.2. [Contoh request & respons](#42-contoh-request--respons)
5. [Endpoint Monitoring & Operasional Tambahan](#5-endpoint-monitoring--operasional-tambahan)
   - 5.1. [Hitung task aktif per grup](#51-hitung-task-aktif-per-grup)
   - 5.2. [Detail satu task](#52-detail-satu-task)
   - 5.3. [Cek status process instance (berjalan/selesai)](#53-cek-status-process-instance-berjalanselesai)
   - 5.4. [Ambil variabel proses langsung](#54-ambil-variabel-proses-langsung)
   - 5.5. [Riwayat aktivitas proses (audit trail)](#55-riwayat-aktivitas-proses-audit-trail)
6. [Catatan Implementasi & Risiko Terkait](#6-catatan-implementasi--risiko-terkait)
   - 6.1. [Risiko](#61-risiko)
   - 6.2. [Checklist implementasi & pengujian](#62-checklist-implementasi--pengujian)

**[BAGIAN II — Endpoint yang HARUS DIBANGUN Aplikasi AWS](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)**

1. [Ringkasan — dua endpoint yang harus dibangun](#1-ringkasan--dua-endpoint-yang-harus-dibangun)
2. [Endpoint A — Notifikasi Task Baru](#2-endpoint-a--notifikasi-task-baru)
   - 2.1. [Skema body request](#21-skema-body-request)
   - 2.2. [Sifat penting yang wajib dipahami](#22-sifat-penting-yang-wajib-dipahami)
3. [Endpoint B — Callback Hasil Approval](#3-endpoint-b--callback-hasil-approval)
   - 3.1. [Skema body request](#31-skema-body-request)
   - 3.2. [Sifat penting yang wajib dipahami](#32-sifat-penting-yang-wajib-dipahami)
4. [Risiko & hal yang belum teruji di server produksi](#4-risiko--hal-yang-belum-teruji-di-server-produksi)
5. [Cara mengganti URL placeholder setelah kontrak tersedia](#5-cara-mengganti-url-placeholder-setelah-kontrak-tersedia)

---

# BAGIAN I — Endpoint yang SUDAH TERSEDIA dari Flowable

*(Aplikasi AWS memanggil endpoint-endpoint ini — sudah disediakan Flowable,*
*tidak perlu dibangun. Detail lengkap: [KONTRAK-API-FLOWABLE.md](./KONTRAK-API-FLOWABLE.md).)*

---

## 1. Ringkasan — alur end-to-end

Untuk menjalankan & memantau proses "Approval Berita Acara" dari awal
sampai akhir, Aplikasi AWS memanggil 3 endpoint Flowable Process REST API
bawaan (bukan buatan proyek ini):

| No. | Endpoint | Dipanggil kapan |
|---|---|---|
| 1 | [§2 Start Instance](#2-endpoint-start-instance) | Sekali per pengajuan, untuk memulai process instance baru |
| 2 | [§3 PULL-Polling Task](#3-endpoint-pull-polling-task) | Berkala (interval sesuai kebutuhan Aplikasi AWS), sebagai jaring pengaman kalau channel PUSH (lihat [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)) gagal terkirim, ATAU untuk menemukan task yang menunggu approval |
| 3 | [§4 Complete Task](#4-endpoint-complete-task) | Setiap kali SATU approver BOD sudah menentukan keputusannya (Setuju/Tolak/Minta Revisi) — dipanggil berkali-kali per instance, satu kali per approver |

**Urutan pemanggilan dari awal sampai akhir satu process instance:**

1. Aplikasi AWS memanggil **Start Instance** (§2) → process instance baru
   dibuat, task `UserTask_ApprovalBod` untuk tiap approver di
   `dynamicApproverGroups` langsung terbentuk paralel.
2. Aplikasi AWS mengetahui adanya task baru lewat **PUSH** (Endpoint A,
   lihat [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)) atau
   lewat **PULL** (§3) sebagai jaring pengaman.
3. Setelah approver terkait menentukan keputusannya di Aplikasi AWS,
   Aplikasi AWS memanggil **Complete Task** (§4) untuk menyelesaikan task
   itu dengan keputusan `TERIMA`/`TOLAK`/`REVISI`. Langkah 2–3 berulang
   untuk **setiap** approver di `dynamicApproverGroups` (paralel, urutan
   penyelesaian bebas).
4. Setelah **SEMUA** approver selesai, Flowable otomatis memicu
   **Callback Hasil Approval** (Endpoint B, lihat
   [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)) — proses
   berakhir sesudahnya. Aplikasi AWS juga bisa memverifikasi hasil akhir
   kapan saja lewat PULL (§3, `includeProcessVariables=true`) tanpa
   menunggu callback ini.

Selain 3 endpoint inti di atas, ada 5 endpoint Flowable bawaan lain yang
**opsional** (dipakai sesuai kebutuhan operasional Aplikasi AWS, bukan
wajib untuk proses ini bisa berjalan) — mis. menghitung task aktif per
grup untuk badge notifikasi, mengecek status akhir process instance, atau
menarik riwayat audit lengkap — semuanya dikumpulkan di
[§5 Endpoint Monitoring & Operasional Tambahan](#5-endpoint-monitoring--operasional-tambahan).

```
┌────────────────┐                                       ┌────────────────┐
│  Aplikasi AWS  │──── 1. POST Start Instance (§2) ─────▶│    Flowable    │
│                │                                        │  (proses ini)  │
│                │──── 2. GET  PULL-Polling (§3) ────────▶│                │
│                │                                        │                │
│                │──── 3. POST Complete Task (§4) ───────▶│                │
│                │       (diulang tiap approver selesai)   │                │
└────────────────┘                                       └────────────────┘
        ▲                                                          │
        └──────── 4. PUSH Callback Hasil (setelah semua selesai) ──┘
                  (Endpoint B, lihat Bagian II)
```

Ketiga endpoint ini adalah Flowable REST bawaan (`title: "Flowable REST
API"`, `basePath: /flowable-rest/service`), sudah dikonfirmasi cocok
terhadap spesifikasi resmi Flowable (`reference/flowable-swagger-process.json`/`.yaml`)
DAN terhadap kode yang sudah lama berjalan di Studio ini sendiri
(`useStartProcessInstance.ts`, `useFlowableTasks.ts`, `useNotifyTasks.ts`,
`useDashboardSummary.ts`) — jadi bisa dipakai langsung sebagai contoh
implementasi yang terbukti jalan.

Di seluruh contoh URL pada dokumen ini (§2 sampai §5), `{VITE_FLOWABLE_BASE_URL}`
adalah **placeholder** untuk base URL server Flowable yang sesungguhnya
dipakai — bukan literal yang dikirim apa adanya. Nilai yang dipakai di
lingkungan yang sudah berjalan saat ini adalah
`https://api-aws.satu.solutions/flowable-rest/service` (lihat
`.env.example`) — jadi URL Start Instance di §2, misalnya, secara nyata
adalah `https://api-aws.satu.solutions/flowable-rest/service/runtime/process-instances`.
Kalau server Flowable berpindah host/domain di kemudian hari, ganti nilai
ini sesuai server yang dipakai saat itu; kredensial Basic Auth di header
`Authorization` juga perlu dikoordinasikan bersamaan dengan tim yang
mengelola server Flowable produksi. Nama `VITE_FLOWABLE_BASE_URL` sendiri
adalah env var internal Studio ini — disebut di sini hanya karena
nilainya kebetulan sama persis dengan `basePath` di spesifikasi Flowable.

---

## 2. Endpoint Start Instance

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances` |
| **Header** | `Content-Type: application/json`, `Authorization: Basic <base64 username:password>` (hanya kalau kredensial Flowable diisi — kalau kosong, header ini tidak dikirim sama sekali, lihat `useFlowableStore().authHeader`) |

### 2.1. Variabel proses yang wajib dikirim

Dikirim lewat field `variables` (array `{name, value}`):

| No. | Nama variabel | Tipe nilai | Keterangan |
|---|---|---|---|
| 1 | `jenisKlien` | string | **Persis** salah satu dari `"Baru"`, `"Existing"`, `"Tender"` (case-sensitive) — dipakai sebagai input decision DMN `keputusanApprovalBod`, lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment). **Tidak lagi menentukan cabang gateway apa pun di BPMN** (`Gateway_JenisKlien` sudah dihapus total dari proses ini) — nilai selain 3 string persis ini membuat Start Instance gagal, lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) |
| 2 | `nilaiProjectRupiah` | number | Nilai project dalam Rupiah — menentukan cabang `Gateway_KategoriNilai` (`< 1.000.000.000`, `1M–2M`, `≥ 2M`), yang lalu menghasilkan `kategoriNilai` ("Kecil"/"Menengah"/"Besar") sebagai input kedua ke decision DMN yang sama (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment)) |
| 3 | `dynamicApproverGroups` | string | Satu string berisi key/ID individu approver BOD, dipisah koma — lihat [§2.2](#22-format-dynamicapprovergroups) untuk aturan formatnya |

### 2.2. Format `dynamicApproverGroups`

`candidateGroup`/`candidateGroups` di Flowable murni string opaque — tidak
pernah divalidasi atau dicocokkan ke tabel identity manapun, jadi key
bergaya `"bod-1"` maupun ID angka polos sama-sama valid. Tapi ada 2 aturan
format yang **wajib** dipatuhi:

| No. | Aturan | Contoh benar | Contoh salah | Akibat kalau salah |
|---|---|---|---|---|
| 1 | Dikirim sebagai **satu string dipisah koma**, bukan array JSON | `"1,2,3"` | `["1", "2", "3"]` | `ScriptTask_SplitApproverGroups` (`dynamicApproverGroups.split(',')` + `java.util.ArrayList` manual) gagal atau menghasilkan `dynamicApproverGroupsList` yang salah, sehingga multi-instance `UserTask_ApprovalBod` tidak terbentuk dengan benar |
| 2 | ID angka polos boleh dipakai, tapi **wajib dikirim sebagai string JSON** | `"1"`, `"2"`, `"3"` | `1`, `2`, `3` (tanpa tanda kutip) | Flowable menyimpan variabel ini bertipe `long`, bukan `string`, sehingga operasi `.split(',')` pada aturan 1 di atas gagal |

### 2.3. Contoh request & respons

**Contoh request nyata** (3 approver BOD dengan ID angka polos):

```
POST {VITE_FLOWABLE_BASE_URL}/runtime/process-instances
Content-Type: application/json
Authorization: Basic <base64 username:password>
```

```json
{
  "processDefinitionKey": "approvalBeritaAcaraProcess",
  "businessKey": "BA-2026-00123",
  "variables": [
    { "name": "jenisKlien", "value": "Existing" },
    { "name": "nilaiProjectRupiah", "value": 1500000000 },
    { "name": "dynamicApproverGroups", "value": "1,2,3" }
  ]
}
```

**Contoh respons nyata** (`201 Created`, hanya field yang dipakai Studio
ini/berguna untuk Aplikasi AWS — respons asli Flowable punya lebih banyak
field):

```json
{
  "id": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "url": "{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "businessKey": "BA-2026-00123"
}
```

| Field respons | Keterangan |
|---|---|
| `id` | Process instance ID — dipakai untuk semua query lanjutan (PULL, Complete Task, histori, dsb.) dan akan muncul di payload kedua endpoint PUSH (lihat [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)) sebagai `processInstanceId` |
| `businessKey` | Dikembalikan apa adanya seperti yang dikirim — berguna untuk mencocokkan balik ke record internal Aplikasi AWS tanpa perlu menyimpan mapping `id`↔`businessKey` terpisah, kalau `businessKey` sendiri sudah dipakai sebagai identifier utama di sisi Aplikasi AWS |

Contoh implementasi yang terbukti jalan (pola request/respons & auth
header persis sama): `src/composables/useStartProcessInstance.ts` (fungsi
`start()`).

### 2.4. Keputusan DMN internal & prasyarat deployment

Tepat setelah `Gateway_KategoriNilai`, proses ini memanggil satu
**Business Rule Task DMN** (`DmnServiceTask_TentukanApproval`,
`flowable:type="dmn"`, memanggil decision `keputusanApprovalBod` dari
`examples/keputusan-approval-bod.dmn`) SEBELUM sampai ke task menunggu
pertama (`UserTask_ApprovalBod`). Ini **bukan panggilan API terpisah** —
Flowable menjalankan decision DMN ini sepenuhnya **di dalam mesin proses
sendiri**, secara sinkron, sebagai bagian dari transaksi Start Instance
yang sama (tidak ada wait state di antaranya). Konsekuensi praktis untuk
Aplikasi AWS:

| No. | Yang perlu dipahami | Detail |
|---|---|---|
| 1 | **Tidak ada endpoint DMN yang perlu dipanggil Aplikasi AWS** untuk langkah ini | Decision `keputusanApprovalBod` dievaluasi otomatis oleh Flowable begitu proses sampai di `DmnServiceTask_TentukanApproval` — berbeda dari mekanisme di [reference/README.md](./reference/README.md) yang membahas DMN REST API (`/dmn-repository/...`, `/dmn-rule/execute/...`) untuk fitur lain di Studio ini (deploy/uji DMN manual) yang TIDAK relevan untuk alur proses ini |
| 2 | Input decision: `jenisKlien` (dikirim langsung saat Start Instance) & `kategoriNilai` (dihitung otomatis oleh `Gateway_KategoriNilai` dari `nilaiProjectRupiah`, BUKAN dikirim langsung) | Kombinasi keduanya menentukan output `approvalBodLabel` — lihat tabel variabel tambahan di [§3.2](#32-variabel-proses-tambahan-includeprocessvariables) untuk cara membacanya |
| 3 | Output decision: `approvalBodLabel` (string, mis. `"2 BOD + Managing Director (dinamis)"`) | Murni label tampilan untuk Aplikasi AWS — **tidak dipakai untuk logika BPMN apa pun** (identitas approver tetap 100% dari `dynamicApproverGroups` yang dikirim Aplikasi AWS sendiri, bukan dari DMN) |
| 4 | **Risiko kegagalan Start Instance**: field `decisionTaskThrowErrorOnNoHits="true"` pada task ini membuat seluruh panggilan Start Instance GAGAL kalau decision tidak menemukan rule yang cocok | Karena hit policy decision ini `UNIQUE` dan rule-nya sudah mencakup semua kombinasi `jenisKlien`×`kategoriNilai` yang valid, ini HANYA terjadi kalau `jenisKlien` bukan persis salah satu dari `"Baru"`/`"Existing"`/`"Tender"` (typo, beda kapitalisasi, dsb.) — lihat risiko terkait di [§6.1](#61-risiko) |
| 5 | **Prasyarat deployment**: file `examples/approval-berita-acara.bpmn` DAN `examples/keputusan-approval-bod.dmn` harus SUDAH ter-deploy ke server Flowable sebelum Start Instance pertama kali dipanggil | Deployment ini dilakukan lewat fitur Deploy di Studio ini (bukan panggilan API rutin yang dilakukan Aplikasi AWS per transaksi) — di luar cakupan kontrak API pada dokumen ini, tapi wajib sudah selesai sebelum go-live |

---

## 3. Endpoint PULL-Polling Task

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup={key}&includeProcessVariables=true` |
| **Header** | `Authorization: Basic <base64 username:password>` (hanya kalau kredensial diisi, sama seperti §2 — endpoint GET ini tidak butuh `Content-Type` karena tidak ada body request) |

Dipakai Aplikasi AWS untuk mengambil daftar task yang sedang menunggu untuk
satu candidate group — ini **jaring pengaman** kalau channel PUSH (Endpoint
A/B, keduanya dijelaskan di [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws))
gagal terkirim, jadi Aplikasi AWS **wajib** memanggil endpoint ini secara
berkala (interval sesuai kebutuhan/SLA Aplikasi AWS sendiri, dokumen ini
tidak menetapkan angka baku) selain menunggu PUSH. `id` tiap task hasil
query ini dipakai sebagai `{taskId}` saat memanggil **Complete Task** (§4).

### 3.1. Parameter query

| No. | Parameter | Wajib? | Keterangan |
|---|---|---|---|
| 1 | `candidateGroup` | Ya | Key/ID approver — satu nilai, sama seperti salah satu elemen `dynamicApproverGroups` di [§2.1](#21-variabel-proses-yang-wajib-dikirim) |
| 2 | `includeProcessVariables` | Disarankan | `true` menyertakan variabel proses di tiap item task hasil — lihat [§3.2](#32-variabel-proses-tambahan-includeprocessvariables) |
| 3 | `processInstanceId` | Opsional | Filter tambahan ke satu process instance tertentu |
| 4 | `dueAfter` / `dueBefore` | Opsional | Filter tanggal jatuh tempo (dikirim sebagai `...T00:00:00.000Z` / `...T23:59:59.999Z`) |
| 5 | `minimumPriority` / `maximumPriority` | Opsional | Filter rentang prioritas task |
| 6 | `sort` / `order` | Opsional | Pengurutan hasil (`priority`/`dueDate`/`createTime`, `asc`/`desc`) |

Daftar lengkap parameter yang sudah dipakai & terbukti jalan di Studio ini:
lihat `TaskSearchParams`/`buildQueryString` di `useFlowableTasks.ts`.

### 3.2. Variabel proses tambahan (`includeProcessVariables`)

`includeProcessVariables=true` membuat respons menyertakan variabel proses
langsung di setiap item task — berguna kalau Aplikasi AWS butuh konteks
lengkap tanpa panggilan tambahan, termasuk untuk melengkapi payload
Endpoint A/B yang **sengaja minim** (lihat catatan di
[§2 Bagian II](#2-endpoint-a--notifikasi-task-baru)
dan [§3 Bagian II](#3-endpoint-b--callback-hasil-approval)):

| Variabel | Keterangan |
|---|---|
| `jenisKlien` | Nilai yang dikirim saat Start Instance ([§2.1](#21-variabel-proses-yang-wajib-dikirim)) |
| `nilaiProjectRupiah` | Nilai yang dikirim saat Start Instance ([§2.1](#21-variabel-proses-yang-wajib-dikirim)) |
| `kategoriNilai` | `"Kecil"`/`"Menengah"`/`"Besar"` — dihitung otomatis dari `nilaiProjectRupiah` oleh gateway di BPMN, BUKAN dikirim Aplikasi AWS (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment)) |
| `approvalBodLabel` | Label tampilan tingkat approval BOD (mis. `"2 BOD + Managing Director (dinamis)"`) — output decision DMN internal, murni untuk ditampilkan Aplikasi AWS, tidak dipakai logika BPMN apa pun (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment)) |
| `hasilApprovalBod` | Terisi sebagian atau lengkap, tergantung berapa banyak approver yang sudah menyelesaikan task-nya lewat **Complete Task** (§4) |

Contoh implementasi yang terbukti jalan (pola query string, auth header,
dan pola polling berkala dengan deteksi task baru): `src/composables/useFlowableTasks.ts`
(fungsi `search()`) dan `src/composables/useNotifyTasks.ts` (fungsi
`pollOnce()`, dipanggil berkala lewat `setInterval` — pola inilah yang
disarankan dicontoh untuk mekanisme polling Aplikasi AWS sendiri).
`src/composables/useDashboardSummary.ts` juga memakai pola auth/fetch yang
sama untuk endpoint hitung ringkasan lain (`/repository/process-definitions`,
dsb.) kalau Aplikasi AWS butuh contoh tambahan.

### 3.3. Contoh request & respons

**Contoh request nyata** (mencari task menunggu untuk approver grup `"2"`,
sekalian menyertakan variabel proses):

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup=2&includeProcessVariables=true
Authorization: Basic <base64 username:password>
```

**Contoh respons nyata** (`200 OK`, `DataResponseTaskResponse` — hanya
field yang dipakai Studio ini/berguna untuk Aplikasi AWS, lihat
`FlowableTask` di `src/types/flowable.ts`; respons asli Flowable punya
field lebih banyak per task):

```json
{
  "data": [
    {
      "id": "7a3f9c2e-1234-4abc-9def-0123456789ab",
      "name": "Approval BOD",
      "processInstanceId": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
      "assignee": null,
      "owner": null,
      "delegationState": null,
      "priority": 50,
      "dueDate": null,
      "variables": [
        { "name": "jenisKlien", "value": "Existing" },
        { "name": "nilaiProjectRupiah", "value": 1500000000 },
        { "name": "hasilApprovalBod", "value": [] }
      ]
    }
  ],
  "total": 1,
  "start": 0,
  "sort": "id",
  "order": "asc",
  "size": 1
}
```

| Field respons | Keterangan |
|---|---|
| `data` | Array task — kosong (`[]`) kalau tidak ada task menunggu untuk `candidateGroup` ini, BUKAN error |
| `data[].id` | Dipakai sebagai `{taskId}` saat memanggil **Complete Task** (§4) |
| `data[].variables` | Hanya muncul kalau `includeProcessVariables=true` — lihat [§3.2](#32-variabel-proses-tambahan-includeprocessvariables) |
| `total` | Jumlah total task yang cocok filter (berguna juga sebagai teknik hitung cepat, lihat [§5.1](#51-hitung-task-aktif-per-grup)) |

Contoh implementasi yang terbukti jalan: lihat rujukan komposabel di akhir
[§3.2](#32-variabel-proses-tambahan-includeprocessvariables) di atas.

---

## 4. Endpoint Complete Task

| | |
|---|---|
| **Method** | `POST` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}` (`{taskId}` didapat dari notifikasi Endpoint A sebagai `taskId`, atau dari field `id` hasil PULL §3) |
| **Header** | `Content-Type: application/json`, `Authorization: Basic <base64 username:password>` (sama seperti §2/§3) |

Dipakai Aplikasi AWS untuk **menyelesaikan** satu task `UserTask_ApprovalBod`
setelah approver terkait menentukan keputusannya — ini langkah yang
membuat proses lanjut: kalau masih ada instance approver lain yang belum
selesai, proses menunggu; kalau ini instance approver **terakhir** yang
selesai, Flowable otomatis memicu Callback Hasil Approval (Endpoint B,
lihat [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)).
Dikonfirmasi terhadap spesifikasi resmi Flowable
(`operationId: executeTaskAction`, `POST /runtime/tasks/{taskId}` dengan
body `TaskActionRequest` — lihat `reference/flowable-swagger-process.json`).

### 4.1. Variabel yang wajib dikirim saat complete

| No. | Nama variabel | Scope | Tipe (`type`) | Nilai valid | Keterangan |
|---|---|---|---|---|---|
| 1 | `keputusanBod` | Task-local (**bukan** variabel proses) | `string` | `"TERIMA"`, `"TOLAK"`, `"REVISI"` | Dibaca oleh `flowable:taskListener` event `complete` pada `UserTask_ApprovalBod`, otomatis dipetakan & ditambahkan ke variabel proses `hasilApprovalBod` — lihat pemetaan di bawah |

**Pemetaan nilai** (dilakukan otomatis oleh script listener BPMN, Aplikasi
AWS tidak perlu melakukan pemetaan ini sendiri):

| Nilai `keputusanBod` yang dikirim | Hasil yang tercatat di `hasilApprovalBod` |
|---|---|
| `"TERIMA"` | `"DITERIMA"` |
| `"TOLAK"` | `"DITOLAK"` |
| `"REVISI"` | `"PERLU REVISI"` (perhatikan spasi, bukan "DIREVISI") |

Studio ini menyediakan 3 tombol aksi (Setujui/Tolak/Minta Revisi) di UI-nya
sendiri yang di baliknya mengirim `keputusanBod` persis dengan salah satu
dari 3 nilai di atas — kalau Aplikasi AWS membangun UI approval-nya
sendiri, kirim salah satu dari 3 nilai persis ini (case-sensitive), bukan
label tombol atau nilai lain.

### 4.2. Contoh request & respons

```
POST {VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}
Content-Type: application/json
Authorization: Basic <base64 username:password>
```

```json
{
  "action": "complete",
  "variables": [
    { "name": "keputusanBod", "value": "TERIMA", "type": "string" }
  ]
}
```

| Kode respons | Arti |
|---|---|
| `200 OK` | Task berhasil di-complete (tidak ada body respons yang perlu dibaca — cukup cek status code) |
| `400 Bad Request` | Body request tidak valid (mis. `action` salah/hilang) |
| `404 Not Found` | `{taskId}` tidak ditemukan (mis. task sudah di-complete approver lain atau ID salah) |
| `409 Conflict` | Task sedang diubah bersamaan (concurrency) — lihat [§6.1 poin 1](#61-risiko) |

**Belum diverifikasi ke server produksi**: bentuk request/respons di atas
mengikuti konvensi resmi `TaskActionRequest`/`RestVariable` Flowable
(dikonfirmasi terhadap `reference/flowable-swagger-process.json`), sama
seperti `TaskCompleteVariablePayload` yang sudah lama ditulis di
`src/types/flowable.ts` — tapi belum pernah dicoba langsung ke server
Flowable produksi dari proyek ini. Kalau server menolak/mengabaikan
`variables` pada action `complete`, atau meminta `type` yang berbeda untuk
suatu nilai, catat pesan error persisnya (ikuti pola pemecahan masalah yang
sama dipakai untuk memverifikasi fitur lain di proyek ini).

Contoh implementasi yang terbukti jalan (pola request, bukan pola
`variables` yang di atas — lihat catatan "belum diverifikasi"):
`src/composables/useFlowableTasks.ts` (fungsi `complete()`).

---

## 5. Endpoint Monitoring & Operasional Tambahan

Kelima endpoint di bawah ini **opsional** — Flowable bawaan juga, tidak
perlu dibangun, tapi tidak wajib dipanggil supaya proses "Approval Berita
Acara" bisa berjalan (berbeda dari §2/§3/§4 yang wajib). Dipakai kalau
Aplikasi AWS butuh kemampuan operasional tambahan: menghitung task aktif
per grup (mis. untuk badge notifikasi jumlah approval menunggu), melihat
detail satu task langsung dari `taskId`, mengecek apakah satu process
instance masih berjalan atau sudah selesai, menarik variabel prosesnya
langsung, atau melihat riwayat audit lengkap satu instance dari awal
sampai akhir.

### 5.1. Hitung task aktif per grup

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup={key}&size=0` |
| **Header** | `Authorization: Basic <base64 username:password>` (sama seperti §3) |

Variasi dari endpoint PULL (§3) — `size=0` membuat Flowable tidak
mengembalikan daftar task-nya, cukup jumlah totalnya saja lewat field
`total` di respons (`DataResponseTaskResponse`, dikonfirmasi terhadap
`reference/flowable-swagger-process.json`).

**Contoh request nyata** (hitung task menunggu untuk approver grup `"2"`):

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup=2&size=0
Authorization: Basic <base64 username:password>
```

**Contoh respons nyata** (`200 OK` — `data` sengaja kosong karena `size=0`,
hanya `total` yang relevan):

```json
{ "data": [], "total": 5, "start": 0, "sort": "id", "order": "asc", "size": 0 }
```

Berguna kalau Aplikasi AWS hanya butuh **angka** (mis. "5 approval
menunggu Anda" di badge notifikasi UI) tanpa perlu menarik & membuang
daftar task lengkapnya — lebih ringan daripada memanggil §3 penuh lalu
menghitung `.length` di sisi Aplikasi AWS. Parameter query lain di §3.1
(`processInstanceId`, `dueAfter`/`dueBefore`, dst.) bisa digabung di sini
juga untuk menghitung subset tertentu.

Contoh implementasi yang terbukti jalan (pola `size=0` + baca `total`):
`src/composables/useDashboardSummary.ts` (fungsi `countFetch()`) —
dipakai di situ per `processDefinitionKey`, bukan per `candidateGroup`,
tapi teknik query-nya identik.

### 5.2. Detail satu task

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}` |
| **Header** | `Authorization: Basic <base64 username:password>` (sama seperti §3) |

Mengambil detail satu task langsung dari `{taskId}` — berguna kalau
Aplikasi AWS sudah tahu `taskId` (mis. dari payload notifikasi Endpoint A
di [Bagian II](#bagian-ii--endpoint-yang-harus-dibangun-aplikasi-aws)) dan ingin
detail lengkapnya tanpa query ulang lewat §3 dengan filter `candidateGroup`.

**Contoh request nyata:**

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/tasks/7a3f9c2e-1234-4abc-9def-0123456789ab
Authorization: Basic <base64 username:password>
```

**Contoh respons nyata** (`200 OK` — field yang sama seperti satu item
`data` di respons §3, lihat `FlowableTask` di `src/types/flowable.ts`):

```json
{
  "id": "7a3f9c2e-1234-4abc-9def-0123456789ab",
  "name": "Approval BOD",
  "processInstanceId": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "assignee": null,
  "owner": null,
  "delegationState": null,
  "priority": 50,
  "dueDate": null
}
```

| Kode respons | Arti |
|---|---|
| `200 OK` | Task ditemukan — body seperti contoh di atas |
| `404 Not Found` | Task tidak ditemukan (mis. sudah di-complete lewat §4, atau `taskId` salah) |

### 5.3. Cek status process instance (berjalan/selesai)

| | |
|---|---|
| **Method** | `GET` |
| **URL (by ID)** | `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/{processInstanceId}` |
| **URL (by Business Key)** | `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances?businessKey={key}` |
| **Header** | `Authorization: Basic <base64 username:password>` (sama seperti §3) |

Dipakai untuk mengetahui apakah satu process instance **masih berjalan**
atau **sudah selesai**, tanpa menunggu Callback Hasil Approval (Endpoint B
di file sebelah):

| No. | Langkah | Keterangan |
|---|---|---|
| 1 | Panggil `GET /runtime/process-instances/{id}` (atau `?businessKey=...`) | `200` + body → **masih berjalan**. `404` (untuk query by ID) atau `data: []` kosong (untuk query by businessKey) → **tidak sedang berjalan**, lanjut ke langkah 2 |
| 2 | Kalau tidak ditemukan di langkah 1, panggil `GET /history/historic-process-instances?processInstanceId={id}` (atau `?businessKey={key}`) | Ditemukan → instance **sudah selesai** (`endTime` terisi di respons, lihat `HistoricProcessInstance` di `src/types/flowable.ts`). Tidak ditemukan di sini juga → ID/Business Key salah, atau instance belum pernah dibuat |

**Contoh request & respons — langkah 1 (masih berjalan):**

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

```json
{
  "id": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "businessKey": "BA-2026-00123",
  "processDefinitionId": "approvalBeritaAcaraProcess:1:158",
  "activityId": "UserTask_ApprovalBod",
  "suspended": false
}
```

`404 Not Found` pada request di atas (atau `?businessKey=BA-2026-00123`
dengan `data: []`) berarti instance tidak sedang berjalan — lanjut ke
langkah 2.

**Contoh request & respons — langkah 2 (fallback ke histori, kalau sudah
selesai):**

```
GET {VITE_FLOWABLE_BASE_URL}/history/historic-process-instances?processInstanceId=4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

```json
{
  "data": [
    {
      "id": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
      "businessKey": "BA-2026-00123",
      "processDefinitionId": "approvalBeritaAcaraProcess:1:158",
      "startTime": "2026-09-04T09:00:00.000+0000",
      "endTime": "2026-09-05T14:30:00.000+0000",
      "durationInMillis": 106200000
    }
  ],
  "total": 1
}
```

`endTime` terisi (bukan `null`) memastikan instance ini benar-benar sudah
**selesai**, bukan sekadar tidak ditemukan karena ID salah.

Contoh implementasi yang terbukti jalan (pola 2 langkah persis di atas,
termasuk fallback ke histori): `src/composables/useProcessTracking.ts`
(fungsi `search()`).

### 5.4. Ambil variabel proses langsung

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/{processInstanceId}/variables` |
| **Header** | `Authorization: Basic <base64 username:password>` (sama seperti §3) |

Mengambil semua variabel proses (`jenisKlien`, `nilaiProjectRupiah`,
`kategoriNilai`, `approvalBodLabel`, `hasilApprovalBod`, dst. — lihat
[§2.1](#21-variabel-proses-yang-wajib-dikirim) & [§3.2](#32-variabel-proses-tambahan-includeprocessvariables))
langsung dari satu process instance, tanpa lewat query task di §3. Berguna
kalau Aplikasi AWS sudah tahu `processInstanceId` (instance masih berjalan
— untuk instance yang **sudah selesai**, pakai riwayat aktivitas di §5.5
atau tunggu Callback Hasil Approval) dan hanya perlu nilai variabelnya,
bukan daftar task.

**Contoh request nyata:**

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd/variables
Authorization: Basic <base64 username:password>
```

**Contoh respons nyata** (`200 OK`, array `{name, value}` per variabel —
lihat `ProcessInstanceVariable` di `src/types/flowable.ts`):

```json
[
  { "name": "jenisKlien", "value": "Existing" },
  { "name": "nilaiProjectRupiah", "value": 1500000000 },
  { "name": "kategoriNilai", "value": "Menengah" },
  { "name": "approvalBodLabel", "value": "2 BOD (dinamis)" },
  { "name": "dynamicApproverGroups", "value": "1,2,3" },
  {
    "name": "hasilApprovalBod",
    "value": [
      { "group": "1", "keputusan": "DITERIMA" },
      { "group": "2", "keputusan": "DITOLAK" }
    ]
  }
]
```

`hasilApprovalBod` di atas terisi sebagian (baru 2 dari 3 approver) karena
process instance ini masih berjalan — lihat [§2.1](#21-variabel-proses-yang-wajib-dikirim)
untuk arti tiap variabel dan [§3 Bagian II](#3-endpoint-b--callback-hasil-approval)
untuk bentuk `hasilApprovalBod` lengkap setelah semua approver selesai.

Contoh implementasi yang terbukti jalan: `src/composables/useProcessTracking.ts`
(fungsi `fetchVariables()`).

### 5.5. Riwayat aktivitas proses (audit trail)

| | |
|---|---|
| **Method** | `GET` |
| **URL** | `{VITE_FLOWABLE_BASE_URL}/history/historic-activity-instances?processInstanceId={id}` |
| **Header** | `Authorization: Basic <base64 username:password>` (sama seperti §3) |

Mengambil seluruh langkah yang sudah dilalui satu process instance secara
berurutan, dari `StartEvent_1` sampai `EndEvent_1` — siapa mengerjakan apa,
kapan, dan berapa lama. Berguna untuk audit/compliance: menunjukkan
riwayat lengkap satu pengajuan approval dari awal sampai akhir, termasuk
approver mana yang memproses task kapan lewat Complete Task (§4).

**Contoh request nyata:**

```
GET {VITE_FLOWABLE_BASE_URL}/history/historic-activity-instances?processInstanceId=4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

**Contoh respons nyata** (`200 OK`, dipangkas jadi 3 langkah pertama untuk
ringkas — instance nyata akan berisi seluruh langkah sampai `EndEvent_1`):

```json
{
  "data": [
    {
      "activityId": "StartEvent_1",
      "activityName": "Permintaan Approval Berita Acara",
      "activityType": "startEvent",
      "assignee": null,
      "startTime": "2026-09-04T09:00:00.000+0000",
      "endTime": "2026-09-04T09:00:00.100+0000",
      "durationInMillis": 100
    },
    {
      "activityId": "UserTask_ApprovalBod",
      "activityName": "Approval BOD",
      "activityType": "userTask",
      "assignee": "budi.approver",
      "startTime": "2026-09-04T09:00:01.000+0000",
      "endTime": "2026-09-04T11:15:00.000+0000",
      "durationInMillis": 8099000
    }
  ],
  "total": 2
}
```

| Field respons | Keterangan |
|---|---|
| `activityId` / `activityName` | ID & nama elemen BPMN (mis. `UserTask_ApprovalBod` / "Approval BOD") |
| `activityType` | Jenis elemen (`userTask`, `serviceTask`, `startEvent`, `endEvent`, dst.) |
| `assignee` | Approver yang mengerjakan (untuk `userTask`) |
| `startTime` / `endTime` | Kapan langkah ini mulai & selesai |
| `durationInMillis` | Durasi langkah ini dalam milidetik |

Lihat `HistoricActivityInstance` di `src/types/flowable.ts` untuk bentuk
lengkapnya. Contoh implementasi yang terbukti jalan (termasuk
pemformatan durasi & label jenis aktivitas ke Bahasa Indonesia):
`src/composables/useAuditTrail.ts`.

---

## 6. Catatan Implementasi & Risiko Terkait

### 6.1. Risiko

| No. | Risiko | Detail & mitigasi |
|---|---|---|
| 1 | Concurrency saat 2+ approver menyelesaikan task hampir bersamaan | Kalau dua atau lebih task `UserTask_ApprovalBod` paralel di-complete nyaris bersamaan lewat **Complete Task** (§4), spesifikasi resmi Flowable mengonfirmasi kemungkinan respons `409 Conflict` ("task was updated simultaneously") — tapi belum ada pengujian eksplisit di server produksi soal seberapa sering ini terjadi dalam praktik. Kalau Aplikasi AWS menerima `409` saat memanggil Complete Task, catat pesan error lengkapnya dan pertimbangkan retry dengan backoff singkat di sisi Aplikasi AWS sendiri (endpoint ini tidak menyediakan retry otomatis) |
| 2 | Start Instance gagal total kalau `jenisKlien` tidak persis salah satu dari 3 nilai yang didukung | Decision DMN internal (§2.4) memakai `decisionTaskThrowErrorOnNoHits="true"` — kalau `jenisKlien` typo/beda kapitalisasi (mis. `"baru"` huruf kecil atau `"New"`), decision tidak menemukan rule yang cocok dan **seluruh panggilan Start Instance gagal** (dijalankan sinkron, bukan gagal belakangan). Ini BUKAN risiko hipotetis — ini konsekuensi langsung dari desain `hitPolicy="UNIQUE"` di `examples/keputusan-approval-bod.dmn`, yang rule-nya sudah mencakup semua kombinasi valid `jenisKlien`×`kategoriNilai`. Mitigasi: validasi `jenisKlien` di sisi Aplikasi AWS SEBELUM memanggil Start Instance (persis `"Baru"`/`"Existing"`/`"Tender"`, case-sensitive) |
| 3 | Risiko endpoint yang **harus dibangun** Aplikasi AWS sendiri | `ignoreException` belum diverifikasi, panggilan HTTP dari Task Listener belum diuji, latensi sinkron Endpoint A, tidak ada retry dari Flowable — lihat [§4 Bagian II](#4-risiko--hal-yang-belum-teruji-di-server-produksi) (tidak diulang di sini supaya tidak ada dua sumber kebenaran untuk hal yang sama) |

### 6.2. Checklist implementasi & pengujian

Checklist ini digabung di satu tempat (bukan dipecah per file) supaya tim
Aplikasi AWS tidak perlu membuka dua dokumen untuk menyusun rencana kerja.
Semua item di bawah ini **wajib** kecuali disebutkan opsional:

| No. | Langkah | Rujukan |
|---|---|---|
| 1 | Bangun **Endpoint A** — Notifikasi Task Baru | [§2 Bagian II](#2-endpoint-a--notifikasi-task-baru) |
| 2 | Bangun **Endpoint B** — Callback Hasil Approval | [§3 Bagian II](#3-endpoint-b--callback-hasil-approval) |
| 3 | Konfirmasi nama field `ignoreException` valid di server Flowable Anda | [§4 Bagian II](#4-risiko--hal-yang-belum-teruji-di-server-produksi), poin 1 |
| 4 | Konfirmasi panggilan HTTP dari dalam Task Listener (Endpoint A) diizinkan JVM server Flowable Anda | Rujukan sama dengan No. 3 di atas, poin 2 |
| 5 | Implementasikan mekanisme **PULL/polling** dengan interval sesuai SLA Aplikasi AWS sendiri — jangan mengandalkan PUSH saja | [§3](#3-endpoint-pull-polling-task) di dokumen ini |
| 6 | Implementasikan pemanggilan **Complete Task** setelah approver menentukan keputusan, termasuk penanganan `409 Conflict` (retry singkat) | [§4](#4-endpoint-complete-task) di dokumen ini |
| 7 | Setelah Endpoint A/B siap & disepakati, ganti URL placeholder di BPMN | [§5 Bagian II](#5-cara-mengganti-url-placeholder-setelah-kontrak-tersedia) |
| 8 | Uji end-to-end: Start Instance → task baru terbuat & Endpoint A terpanggil (atau terlewat, ketahuan lewat PULL) → Complete Task per approver (perhatikan risiko concurrency di [§6.1](#61-risiko) kalau pengujian melibatkan banyak approver sekaligus) → Endpoint B terpanggil dengan `hasilApprovalBod` lengkap setelah semua approver selesai | [§2](#2-endpoint-start-instance), [§3](#3-endpoint-pull-polling-task) & [§4](#4-endpoint-complete-task) di dokumen ini |
| 9 | Validasi `jenisKlien` di sisi Aplikasi AWS SEBELUM memanggil Start Instance — harus **persis** `"Baru"`, `"Existing"`, atau `"Tender"` (case-sensitive), bukan sekadar "ada isinya" | [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) & [§6.1](#61-risiko) poin 2 |
| 10 | Pastikan `examples/approval-berita-acara.bpmn` DAN `examples/keputusan-approval-bod.dmn` sudah ter-deploy ke server Flowable sebelum go-live/pengujian pertama | [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) poin 5 |
| 11 *(opsional)* | Implementasikan endpoint monitoring/operasional tambahan sesuai kebutuhan (badge hitung task, detail task, cek status instance, ambil variabel langsung, audit trail) | [§5](#5-endpoint-monitoring--operasional-tambahan) di dokumen ini |

---

# BAGIAN II — Endpoint yang HARUS DIBANGUN Aplikasi AWS

*(Flowable memanggil endpoint-endpoint ini — harus dibangun & di-hosting*
*oleh Aplikasi AWS. Detail lengkap: [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md).)*

Checklist implementasi & pengujian lengkap (membangun kedua endpoint,
mengganti URL, uji end-to-end) ada di
[§6 Bagian I](#6-catatan-implementasi--risiko-terkait)
bersama item checklist PULL & Complete Task — checklist digabung di satu
tempat supaya tim Aplikasi AWS tidak perlu buka dua file untuk menyusun rencana
kerja lengkap.

---

## 1. Ringkasan — dua endpoint yang harus dibangun

Proses "Approval Berita Acara" memicu **2 HTTP callback PUSH** ke Aplikasi
AWS (Agreement Workflow System) — selanjutnya disebut Aplikasi AWS —
lewat mekanisme Flowable `flowable:taskListener`/`flowable:type="http"`.
Kedua endpoint di bawah ini **harus dibangun & di-hosting oleh Aplikasi
AWS** — Flowable hanya memanggilnya (klien), tidak menyediakannya:

| Endpoint | Dipicu dari | Kapan |
|---|---|---|
| [§2 Endpoint A](#2-endpoint-a--notifikasi-task-baru) — Notifikasi Task Baru | Task Listener `UserTask_ApprovalBod` | Setiap task Approval BOD baru dibuat (berkali-kali per instance, paralel) |
| [§3 Endpoint B](#3-endpoint-b--callback-hasil-approval) — Callback Hasil Approval | Service Task `ServiceTask_CallbackHasil` | Sekali per instance, setelah SEMUA approval selesai |

Push ini **sengaja dijalankan dual-channel** bersama PULL (kesepakatan
RINGKASAN-PEMBAHASAN.md Update 3): push memberi kecepatan (real-time),
PULL (dijelaskan lengkap di
[§3 Bagian I](#3-endpoint-pull-polling-task))
jadi jaring pengaman kalau push gagal terkirim (jaringan putus, endpoint
Aplikasi AWS sedang down, dsb.) — jadi Aplikasi AWS **tidak boleh
mengandalkan push sebagai satu-satunya sumber kebenaran**. Mekanisme
pemicu push ini sudah diimplementasikan penuh di sisi BPMN; yang masih
PLACEHOLDER cuma URL tujuannya — itulah "2 kontrak API" yang harus
dibangun.

```
┌────────────────┐                                       ┌────────────────┐
│  Aplikasi AWS  │◀───── PUSH: Endpoint A (§2) ──────────│    Flowable    │
│                │       notifikasi task baru             │  (proses ini)  │
│                │◀───── PUSH: Endpoint B (§3) ──────────│                │
│                │       callback hasil approval           │                │
└────────────────┘                                       └────────────────┘
```

---

## 2. Endpoint A — Notifikasi Task Baru

| | |
|---|---|
| **Dipicu dari** | `flowable:taskListener` event `create` pada `UserTask_ApprovalBod` (baris ~361–385 di `examples/approval-berita-acara.bpmn`) |
| **Kapan terpicu** | Setiap kali SATU instance task Approval BOD dibuat — untuk multi-instance paralel, ini terpicu **berkali-kali per process instance** (satu kali per approver/candidate group dalam `dynamicApproverGroups`) |
| **Method** | `POST` |
| **URL** | PLACEHOLDER saat ini: `https://aplikasi-utama.contoh/api/notifikasi/task-baru` — **ganti dengan URL asli endpoint Aplikasi AWS** |
| **Header** | `Content-Type: application/json` |
| **Timeout klien** | 5 detik connect + 5 detik read (di-hardcode di script listener) |
| **Respons diharapkan** | **Tidak diperiksa sama sekali** oleh Flowable — script listener memanggil `conn.getResponseCode()` semata-mata untuk memaksa request selesai terkirim (flush), nilainya tidak dibaca/dicek. Disarankan tetap mengembalikan `2xx` secepatnya (endpoint ini dipanggil sinkron dari dalam transaksi `taskListener` — lihat [§4](#4-risiko--hal-yang-belum-teruji-di-server-produksi) soal implikasi latensinya) dan tidak perlu balikan body apa pun |

### 2.1. Skema body request

Semua field string:

```json
{
  "taskId": "string",
  "processInstanceId": "string",
  "candidateGroup": "string",
  "taskName": "string"
}
```

Contoh nyata:

```json
{
  "taskId": "7a3f9c2e-1234-4abc-9def-0123456789ab",
  "processInstanceId": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "candidateGroup": "2",
  "taskName": "Approval BOD"
}
```

| No. | Field | Keterangan |
|---|---|---|
| 1 | `taskId` | ID task Flowable ini, dipakai Aplikasi AWS untuk query detail lengkap lewat PULL kalau perlu (lihat [§3 Bagian I](#3-endpoint-pull-polling-task)) — payload ini **sengaja minim**, bukan detail lengkap |
| 2 | `processInstanceId` | Untuk mengaitkan notifikasi ke process instance yang sama |
| 3 | `candidateGroup` | Key BOD approver individu untuk instance task ini (satu elemen dari `dynamicApproverGroups` yang dikirim saat Start Instance, lihat [§2 Bagian I](#2-endpoint-start-instance)), dipakai untuk menentukan siapa yang perlu diberi tahu |
| 4 | `taskName` | Selalu `"Approval BOD"` untuk task ini (nama task di BPMN), disertakan supaya payload tetap jelas kalau nanti ada task lain yang memakai pola listener yang sama |

### 2.2. Sifat penting yang wajib dipahami

| No. | Sifat | Detail |
|---|---|---|
| 1 | Best-effort, bukan jalur terjamin | Panggilan ini dibungkus `try/catch` di sisi BPMN — **kalau endpoint ini gagal/timeout/error apa pun, kegagalannya SENGAJA ditelan** dan tidak menggagalkan pembuatan task |
| 2 | PULL/polling wajib sebagai jaring pengaman | Karena sifat best-effort di atas, Aplikasi AWS **wajib** tetap punya mekanisme PULL/polling (lihat [§3 Bagian I](#3-endpoint-pull-polling-task)) — jangan mengasumsikan setiap task baru pasti menghasilkan panggilan ke endpoint ini |
| 3 | Tidak ada retry dari Flowable | Kalau panggilan endpoint ini gagal, Flowable tidak mengulanginya sama sekali |

---

## 3. Endpoint B — Callback Hasil Approval

| | |
|---|---|
| **Dipicu dari** | `ServiceTask_CallbackHasil` (`flowable:type="http"`, baris ~434–461) |
| **Kapan terpicu** | Sekali per process instance, setelah **SEMUA** instance paralel `UserTask_ApprovalBod` selesai (approve/tolak/revisi) — ini langkah terakhir sebelum proses berakhir (`EndEvent_1`) |
| **Method** | `POST` |
| **URL** | PLACEHOLDER saat ini: `https://aplikasi-utama.contoh/api/callback/hasil-approval` — **ganti dengan URL asli endpoint Aplikasi AWS** |
| **Header** | `Content-Type: application/json` |
| **Respons diharapkan** | Field `responseVariableName` di service task ini diset ke `resultCallbackResponse` — artinya **body respons HTTP disimpan sebagai variabel proses** di sisi Flowable. Namun task ini adalah langkah TERAKHIR sebelum `EndEvent_1` — proses berakhir tepat sesudahnya, jadi saat ini **tidak ada logic BPMN apa pun yang membaca `resultCallbackResponse`**. Aplikasi AWS bebas mengembalikan body apa pun (disarankan tetap `2xx` + body JSON ringkas berisi status penerimaan, untuk memudahkan audit lewat riwayat variabel proses kalau suatu saat perlu ditelusuri) |

### 3.1. Skema body request

```json
{
  "processInstanceId": "string",
  "hasilApprovalBod": [
    { "group": "string", "keputusan": "DITERIMA | DITOLAK | PERLU REVISI" }
  ]
}
```

Contoh nyata (Komite 3 BOD, satu tolak satu revisi satu terima):

```json
{
  "processInstanceId": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "hasilApprovalBod": [
    { "group": "1", "keputusan": "DITERIMA" },
    { "group": "2", "keputusan": "DITOLAK" },
    { "group": "3", "keputusan": "PERLU REVISI" }
  ]
}
```

| No. | Field | Keterangan |
|---|---|---|
| 1 | `processInstanceId` | ID process instance yang mau ditutup hasil approval-nya |
| 2 | `hasilApprovalBod` | Berisi **satu entri per approver** yang berpartisipasi di instance ini (jumlah & urutan entri mengikuti `dynamicApproverGroups` yang dikirim saat Start Instance, tapi urutan penyelesaian task bisa beda karena approval paralel — jangan asumsikan urutan array = urutan `dynamicApproverGroups`, cocokkan lewat field `group`) |
| 3 | `keputusan` (di dalam tiap entri `hasilApprovalBod`) | **Hanya salah satu dari 3 nilai string persis ini** (perhatikan spasi di `"PERLU REVISI"`): `"DITERIMA"`, `"DITOLAK"`, `"PERLU REVISI"`. Ini adalah hasil pemetaan dari 3 tombol aksi Studio (`TERIMA`/`TOLAK`/`REVISI`) — **bukan** nilai mentah tombolnya |

Tidak ada field kategori nilai/jenis klien di payload ini — kalau Aplikasi
AWS butuh konteks itu, ambil lewat PULL (`includeProcessVariables=true`,
lihat [§3 Bagian I](#3-endpoint-pull-polling-task))
memakai `processInstanceId` di atas sebagai kunci.

### 3.2. Sifat penting yang wajib dipahami

| No. | Sifat | Detail |
|---|---|---|
| 1 | `ignoreException` belum diverifikasi | Field `ignoreException` diset `true` pada task ini — **niatnya** supaya kegagalan callback (timeout, 4xx/5xx, dsb.) tidak menggagalkan seluruh process instance. **Tapi nama field ini BELUM DIVERIFIKASI valid** di server produksi (beda dari `requestMethod`/`requestUrl`/`requestBody`/`requestHeaders`/`responseVariableName` yang formatnya sudah terbukti jalan lewat `test-http-task.bpmn`) — lihat [§4](#4-risiko--hal-yang-belum-teruji-di-server-produksi) poin terkait sebelum mengandalkan perilaku "gagal tapi proses tetap lanjut" ini |
| 2 | Channel PUSH pelengkap, bukan satu-satunya sumber | Sama seperti Endpoint A, callback ini adalah channel **PUSH pelengkap** — hasil approval akhir tetap bisa diambil lewat PULL kapan pun (variabel proses `hasilApprovalBod` sudah lengkap begitu semua task selesai, bisa dibaca sebelum callback ini terkirim sekalipun) |

---

## 4. Risiko & hal yang belum teruji di server produksi

Daftar ini diringkas dari komentar KOREKSI KELIMA poin (d)/(e) di kepala
`examples/approval-berita-acara.bpmn` — dibaca sebelum go-live. Semua poin
di sini spesifik untuk Endpoint A/B di atas — risiko yang berkaitan dengan
endpoint yang SUDAH ADA (mis. concurrency saat "complete task", lihat juga
kontrak endpoint Complete Task itu sendiri) ada di
[§6 Bagian I](#6-catatan-implementasi--risiko-terkait):

| No. | Risiko | Detail & tindak lanjut |
|---|---|---|
| 1 | `ignoreException` (Endpoint B) belum dikonfirmasi nama field yang valid | Di server Flowable Anda. Kalau Start Instance/eksekusi `ServiceTask_CallbackHasil` gagal dengan pesan field tidak dikenal, catat nama field yang disebut di pesan error itu (ikuti pola pemecahan masalah yang sama dipakai untuk memverifikasi `test-http-task.bpmn`) |
| 2 | Panggilan HTTP dari dalam Task Listener (Endpoint A) belum diuji di server produksi | Task listener memakai `java.net.URL`/`HttpURLConnection` langsung (bukan `flowable:type="http"` — mekanisme itu setahu tim ini cuma berlaku untuk `serviceTask`, bukan task listener) — perlu diverifikasi dulu apakah JVM server Flowable Anda mengizinkan panggilan jaringan keluar dari dalam script task listener (tergantung sandbox/security manager JVM, kalau ada) |
| 3 | Endpoint A dipanggil SINKRON di dalam transaksi penyelesaian task | Task listener event `create` berjalan sebagai bagian dari command yang sama yang membuat task. Endpoint Aplikasi AWS yang lambat merespons berarti memperlambat pembuatan task itu sendiri (timeout 5 detik membatasi ini, tapi 5 detik tetap signifikan kalau terjadi berkali-kali untuk multi-instance dengan banyak approver sekaligus) — endpoint ini **wajib** dibangun cepat (idealnya <1 detik) dan idealnya asinkron/fire-and-forget di sisi Aplikasi AWS sendiri (terima request, langsung `202`/`200`, proses lebih lanjut di background) |
| 4 | Endpoint A/B tidak pernah di-retry oleh Flowable kalau gagal | Ditelan (`try/catch` untuk A) atau (niatnya, tergantung poin 1) `ignoreException` untuk B. Rekonsiliasi lewat PULL (lihat [§3 Bagian I](#3-endpoint-pull-polling-task)) adalah satu-satunya jaring pengaman resmi — jangan bangun mekanisme retry di sisi BPMN untuk ini, itu di luar cakupan proses ini (lihat juga catatan di komentar kepala BPMN soal ini) |

---

## 5. Cara mengganti URL placeholder setelah kontrak tersedia

Studio ini (aplikasi BPMN/DMN Studio Vue) sekarang punya fitur Panel
Properti yang bisa langsung mengedit kedua lokasi placeholder ini **tanpa
edit XML manual**:

| No. | Langkah | Rujukan |
|---|---|---|
| 1 | Buka `examples/approval-berita-acara.bpmn` di Editor Diagram | — |
| 2 | **Endpoint A** (notifikasi task baru): klik elemen `UserTask_ApprovalBod` ("Approval BOD") di kanvas → di Panel Properti, bagian "Flowable — Task Listener" → cari baris dengan Event `create` → field isi script/URL listener ini bisa diedit langsung di sana | Catatan Teknis butir 27 di README.md |
| 3 | **Endpoint B** (callback hasil approval): klik elemen `ServiceTask_CallbackHasil` (nama tampilan di kanvas saat ini masih "Callback Hasil ke Aplikasi Utama (PLACEHOLDER)" — belum ikut diubah ke "Aplikasi AWS" karena perubahan istilah di dokumen ini sengaja dibatasi ke dokumen kontrak API, tidak menyentuh BPMN; ganti manual di Panel Properti kalau mau disamakan, murni kosmetik) di kanvas → di Panel Properti akan muncul field `Request URL` (dan field HTTP lain: Request Method/Body/Headers/Response Variable Name/Abaikan Exception) — ganti `Request URL` dengan URL asli Endpoint B | Catatan Teknis butir 26 di README.md |
| 4 | Setelah kedua URL diganti, kembali ke kanvas — outline merah/kuning & badge peringatan pada kedua elemen ini (kalau sebelumnya muncul karena domain contoh) akan otomatis hilang, dan Dialog Deploy tidak lagi menampilkan peringatan domain contoh untuk kedua elemen ini | Catatan Teknis butir 28 (outline/badge) & 25 (Dialog Deploy) di README.md |
| 5 | Deploy diagram (key proses yang sama akan otomatis jadi versi baru, tidak menimpa versi lama — instance yang sedang berjalan di versi lama tetap memakai URL placeholder sampai selesai; hanya instance BARU yang memakai URL asli) | — |

Nama elemen di atas (`UserTask_ApprovalBod`, `ServiceTask_CallbackHasil`)
adalah `id` di XML — kalau bingung mencari elemen mana yang mana di kanvas,
cari nama tampilan **"Approval BOD"** dan **"Callback Hasil ke Aplikasi
Utama (PLACEHOLDER)"** (label `(PLACEHOLDER)` ini juga sebaiknya dihapus
dari nama tampilan elemen setelah kontrak diisi, murni kosmetik).

