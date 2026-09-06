# Kontrak API — Endpoint Flowable (Proses "Approval Berita Acara")

## Daftar Isi

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

---

## 1. Ringkasan — alur end-to-end

Untuk menjalankan & memantau proses "Approval Berita Acara" dari awal
sampai akhir, Aplikasi AWS memanggil 3 endpoint Flowable Process REST API
bawaan (bukan buatan proyek ini):

| No. | Endpoint | Dipanggil kapan |
|---|---|---|
| 1 | [§2 Start Instance](#2-endpoint-start-instance) | Sekali per pengajuan, untuk memulai process instance baru |
| 2 | [§3 PULL-Polling Task](#3-endpoint-pull-polling-task) | Berkala (interval sesuai kebutuhan Aplikasi AWS), sebagai jaring pengaman kalau channel PUSH (lihat [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) gagal terkirim, ATAU untuk menemukan task yang menunggu approval |
| 3 | [§4 Complete Task](#4-endpoint-complete-task) | Setiap kali SATU approver BOD sudah menentukan keputusannya (Setuju/Tolak/Minta Revisi) — dipanggil berkali-kali per instance, satu kali per approver |

**Urutan pemanggilan dari awal sampai akhir satu process instance:**

1. Aplikasi AWS memanggil **Start Instance** (§2) → process instance baru
   dibuat, task `UserTask_ApprovalBod` untuk tiap approver di
   `dynamicApproverGroups` langsung terbentuk paralel.
2. Aplikasi AWS mengetahui adanya task baru lewat **PUSH** (Endpoint A,
   lihat [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) atau
   lewat **PULL** (§3) sebagai jaring pengaman.
3. Setelah approver terkait menentukan keputusannya di Aplikasi AWS,
   Aplikasi AWS memanggil **Complete Task** (§4) untuk menyelesaikan task
   itu dengan keputusan `TERIMA`/`TOLAK`/`REVISI`. Langkah 2–3 berulang
   untuk **setiap** approver di `dynamicApproverGroups` (paralel, urutan
   penyelesaian bebas).
4. Setelah **SEMUA** approver selesai, Flowable otomatis memicu
   **Callback Hasil Approval** (Endpoint B, lihat
   [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) — proses
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
                  (Endpoint B, lihat KONTRAK-API-APLIKASI-AWS.md)
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
| `id` | Process instance ID — dipakai untuk semua query lanjutan (PULL, Complete Task, histori, dsb.) dan akan muncul di payload kedua endpoint PUSH (lihat [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) sebagai `processInstanceId` |
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
A/B, keduanya dijelaskan di [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md))
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
[KONTRAK-API-APLIKASI-AWS.md §2](./KONTRAK-API-APLIKASI-AWS.md#2-endpoint-a--notifikasi-task-baru)
dan [§3](./KONTRAK-API-APLIKASI-AWS.md#3-endpoint-b--callback-hasil-approval)):

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
lihat [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)).
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
di [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) dan ingin
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
untuk arti tiap variabel dan [KONTRAK-API-APLIKASI-AWS.md §3](./KONTRAK-API-APLIKASI-AWS.md#3-endpoint-b--callback-hasil-approval)
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
| 3 | Risiko endpoint yang **harus dibangun** Aplikasi AWS sendiri | `ignoreException` belum diverifikasi, panggilan HTTP dari Task Listener belum diuji, latensi sinkron Endpoint A, tidak ada retry dari Flowable — lihat [KONTRAK-API-APLIKASI-AWS.md §4](./KONTRAK-API-APLIKASI-AWS.md#4-risiko--hal-yang-belum-teruji-di-server-produksi) (tidak diulang di sini supaya tidak ada dua sumber kebenaran untuk hal yang sama) |

### 6.2. Checklist implementasi & pengujian

Checklist ini digabung di satu tempat (bukan dipecah per file) supaya tim
Aplikasi AWS tidak perlu membuka dua dokumen untuk menyusun rencana kerja.
Semua item di bawah ini **wajib** kecuali disebutkan opsional:

| No. | Langkah | Rujukan |
|---|---|---|
| 1 | Bangun **Endpoint A** — Notifikasi Task Baru | [KONTRAK-API-APLIKASI-AWS.md §2](./KONTRAK-API-APLIKASI-AWS.md#2-endpoint-a--notifikasi-task-baru) |
| 2 | Bangun **Endpoint B** — Callback Hasil Approval | [KONTRAK-API-APLIKASI-AWS.md §3](./KONTRAK-API-APLIKASI-AWS.md#3-endpoint-b--callback-hasil-approval) |
| 3 | Konfirmasi nama field `ignoreException` valid di server Flowable Anda | [KONTRAK-API-APLIKASI-AWS.md §4](./KONTRAK-API-APLIKASI-AWS.md#4-risiko--hal-yang-belum-teruji-di-server-produksi), poin 1 |
| 4 | Konfirmasi panggilan HTTP dari dalam Task Listener (Endpoint A) diizinkan JVM server Flowable Anda | Rujukan sama dengan No. 3 di atas, poin 2 |
| 5 | Implementasikan mekanisme **PULL/polling** dengan interval sesuai SLA Aplikasi AWS sendiri — jangan mengandalkan PUSH saja | [§3](#3-endpoint-pull-polling-task) di dokumen ini |
| 6 | Implementasikan pemanggilan **Complete Task** setelah approver menentukan keputusan, termasuk penanganan `409 Conflict` (retry singkat) | [§4](#4-endpoint-complete-task) di dokumen ini |
| 7 | Setelah Endpoint A/B siap & disepakati, ganti URL placeholder di BPMN | [KONTRAK-API-APLIKASI-AWS.md §5](./KONTRAK-API-APLIKASI-AWS.md#5-cara-mengganti-url-placeholder-setelah-kontrak-tersedia) |
| 8 | Uji end-to-end: Start Instance → task baru terbuat & Endpoint A terpanggil (atau terlewat, ketahuan lewat PULL) → Complete Task per approver (perhatikan risiko concurrency di [§6.1](#61-risiko) kalau pengujian melibatkan banyak approver sekaligus) → Endpoint B terpanggil dengan `hasilApprovalBod` lengkap setelah semua approver selesai | [§2](#2-endpoint-start-instance), [§3](#3-endpoint-pull-polling-task) & [§4](#4-endpoint-complete-task) di dokumen ini |
| 9 | Validasi `jenisKlien` di sisi Aplikasi AWS SEBELUM memanggil Start Instance — harus **persis** `"Baru"`, `"Existing"`, atau `"Tender"` (case-sensitive), bukan sekadar "ada isinya" | [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) & [§6.1](#61-risiko) poin 2 |
| 10 | Pastikan `examples/approval-berita-acara.bpmn` DAN `examples/keputusan-approval-bod.dmn` sudah ter-deploy ke server Flowable sebelum go-live/pengujian pertama | [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) poin 5 |
| 11 *(opsional)* | Implementasikan endpoint monitoring/operasional tambahan sesuai kebutuhan (badge hitung task, detail task, cek status instance, ambil variabel langsung, audit trail) | [§5](#5-endpoint-monitoring--operasional-tambahan) di dokumen ini |

