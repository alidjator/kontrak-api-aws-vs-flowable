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
     - 5.1.1. [Contoh request & respons](#511-contoh-request--respons)
   - 5.2. [Detail satu task](#52-detail-satu-task)
     - 5.2.1. [Contoh request & respons](#521-contoh-request--respons)
   - 5.3. [Cek status process instance (berjalan/selesai)](#53-cek-status-process-instance-berjalanselesai)
     - 5.3.1. [Mekanisme 2 langkah](#531-mekanisme-2-langkah)
     - 5.3.2. [Contoh request & respons](#532-contoh-request--respons)
   - 5.4. [Ambil variabel proses langsung](#54-ambil-variabel-proses-langsung)
     - 5.4.1. [Contoh request & respons](#541-contoh-request--respons)
   - 5.5. [Riwayat aktivitas proses (audit trail)](#55-riwayat-aktivitas-proses-audit-trail)
     - 5.5.1. [Contoh request & respons](#551-contoh-request--respons)
   - 5.6. [Aksi Lain pada Task (Claim, Delegate, Resolve)](#56-aksi-lain-pada-task-claim-delegate-resolve)
     - 5.6.1. [Contoh request & respons](#561-contoh-request--respons)
   - 5.7. [Batalkan Process Instance](#57-batalkan-process-instance)
     - 5.7.1. [Contoh request & respons](#571-contoh-request--respons)
   - 5.8. [Komentar/Alasan pada Task](#58-komentaralasan-pada-task)
     - 5.8.1. [Contoh request & respons](#581-contoh-request--respons)
   - 5.9. [Diagram Posisi Process Instance](#59-diagram-posisi-process-instance)
     - 5.9.1. [Contoh request & respons](#591-contoh-request--respons)
   - 5.10. [Monitoring Failed Job (Dead-Letter)](#510-monitoring-failed-job-dead-letter)
     - 5.10.1. [Contoh request & respons](#5101-contoh-request--respons)
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

**Method:** `POST`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances`

**Header:** `Content-Type: application/json`, `Authorization: Basic <base64 username:password>` (hanya kalau kredensial Flowable diisi — kalau kosong, header ini tidak dikirim sama sekali, lihat `useFlowableStore().authHeader`)

### 2.1. Variabel proses yang wajib dikirim

Dikirim lewat field `variables` (array `{name, value}`):

| No. | Nama variabel | Tipe nilai | Keterangan |
|---|---|---|---|
| 1 | `jenisKlien` | string | **Persis** salah satu dari `"Baru"`, `"Existing"`, `"Tender"` (case-sensitive) — dipakai sebagai input decision DMN `keputusanApprovalBod`, lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment). **Tidak lagi menentukan cabang gateway apa pun di BPMN** (`Gateway_JenisKlien` sudah dihapus total dari proses ini) — nilai selain 3 string persis ini membuat Start Instance gagal, lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) |
| 2 | `nilaiProjectRupiah` | number | Nilai project dalam Rupiah — menentukan cabang `Gateway_KategoriNilai` (`< 1 Miliar`, `1–2 Miliar`, `≥ 2 Miliar` — persis label `Flow_KategoriKecil`/`Menengah`/`Besar` di `examples/approval-berita-acara.bpmn`), yang lalu menghasilkan `kategoriNilai` ("Kecil"/"Menengah"/"Besar") sebagai input kedua ke decision DMN yang sama (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment)) |
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

Contoh berikut memakai 3 approver BOD dengan ID angka polos.

#### Request

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

#### Respons

`201 Created` — hanya field yang dipakai Studio ini/berguna untuk
Aplikasi AWS (respons asli Flowable punya lebih banyak field):

```json
{
  "id": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "url": "{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd",
  "businessKey": "BA-2026-00123"
}
```

| No. | Field respons | Keterangan |
|---|---|---|
| 1 | `id` | Process instance ID — dipakai untuk semua query lanjutan (PULL, Complete Task, histori, dsb.) dan akan muncul di payload kedua endpoint PUSH (lihat [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) sebagai `processInstanceId` |
| 2 | `businessKey` | Dikembalikan apa adanya seperti yang dikirim — berguna untuk mencocokkan balik ke record internal Aplikasi AWS tanpa perlu menyimpan mapping `id`↔`businessKey` terpisah, kalau `businessKey` sendiri sudah dipakai sebagai identifier utama di sisi Aplikasi AWS |

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `201 Created` | Process instance berhasil dibuat — body seperti contoh respons di atas |
| 2 | `400 Bad Request` | `processDefinitionKey` tidak ditemukan (proses belum ter-deploy, lihat [§2.4 poin 5](#24-keputusan-dmn-internal--prasyarat-deployment)), atau salah satu variabel di [§2.1](#21-variabel-proses-yang-wajib-dikirim) memiliki tipe/format tidak valid — dikonfirmasi terhadap `reference/flowable-swagger-process.json` (`operationId: createProcessInstance`) |

**Belum diverifikasi ke server produksi**: kode respons persis untuk
kegagalan decision DMN internal (rule tidak ditemukan karena `jenisKlien`
tidak valid — lihat
[§2.4 poin 4](#24-keputusan-dmn-internal--prasyarat-deployment) &
[§6.1 poin 2](#61-risiko)) tidak didokumentasikan eksplisit di
spesifikasi resmi Flowable untuk operasi ini — kegagalan itu terjadi di
dalam mesin proses saat mengeksekusi Business Rule Task, bukan validasi
REST layer biasa seperti 2 kode di atas. Kemungkinan besar tetap `4xx`,
tapi belum pernah dicoba langsung ke server produksi dari proyek ini.

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

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup={key}&includeProcessVariables=true`

**Header:** `Authorization: Basic <base64 username:password>` (hanya kalau kredensial diisi, sama seperti §2 — endpoint GET ini tidak butuh `Content-Type` karena tidak ada body request)

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

| No. | Variabel | Keterangan |
|---|---|---|
| 1 | `jenisKlien` | Nilai yang dikirim saat Start Instance ([§2.1](#21-variabel-proses-yang-wajib-dikirim)) |
| 2 | `nilaiProjectRupiah` | Nilai yang dikirim saat Start Instance ([§2.1](#21-variabel-proses-yang-wajib-dikirim)) |
| 3 | `kategoriNilai` | `"Kecil"`/`"Menengah"`/`"Besar"` — dihitung otomatis dari `nilaiProjectRupiah` oleh gateway di BPMN, BUKAN dikirim Aplikasi AWS (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment)) |
| 4 | `approvalBodLabel` | Label tampilan tingkat approval BOD (mis. `"2 BOD + Managing Director (dinamis)"`) — output decision DMN internal, murni untuk ditampilkan Aplikasi AWS, tidak dipakai logika BPMN apa pun (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment)) |
| 5 | `hasilApprovalBod` | Terisi sebagian atau lengkap, tergantung berapa banyak approver yang sudah menyelesaikan task-nya lewat **Complete Task** (§4) |

Contoh implementasi yang terbukti jalan (pola query string, auth header,
dan pola polling berkala dengan deteksi task baru): `src/composables/useFlowableTasks.ts`
(fungsi `search()`) dan `src/composables/useNotifyTasks.ts` (fungsi
`pollOnce()`, dipanggil berkala lewat `setInterval` — pola inilah yang
disarankan dicontoh untuk mekanisme polling Aplikasi AWS sendiri).
`src/composables/useDashboardSummary.ts` juga memakai pola auth/fetch yang
sama untuk endpoint hitung ringkasan lain (`/repository/process-definitions`,
dsb.) kalau Aplikasi AWS butuh contoh tambahan.

### 3.3. Contoh request & respons

Contoh berikut mencari task menunggu untuk approver grup `"2"`, sekalian
menyertakan variabel proses.

#### Request

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup=2&includeProcessVariables=true
Authorization: Basic <base64 username:password>
```

#### Respons

`200 OK`, `DataResponseTaskResponse` — hanya field yang dipakai Studio
ini/berguna untuk Aplikasi AWS, lihat `FlowableTask` di
`src/types/flowable.ts`; respons asli Flowable punya field lebih banyak
per task:

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

| No. | Field respons | Keterangan |
|---|---|---|
| 1 | `data` | Array task — kosong (`[]`) kalau tidak ada task menunggu untuk `candidateGroup` ini, BUKAN error |
| 2 | `data[].id` | Dipakai sebagai `{taskId}` saat memanggil **Complete Task** (§4) |
| 3 | `data[].variables` | Hanya muncul kalau `includeProcessVariables=true` — lihat [§3.2](#32-variabel-proses-tambahan-includeprocessvariables) |
| 4 | `total` | Jumlah total task yang cocok filter (berguna juga sebagai teknik hitung cepat, lihat [§5.1](#51-hitung-task-aktif-per-grup)) |

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Query berhasil — `data` berisi task yang cocok filter (array kosong `[]` kalau tidak ada, BUKAN error) |
| 2 | `404 Not Found` | Salah satu parameter query tidak valid (mis. format tanggal `dueAfter`/`dueBefore` salah), atau `delegationState` diisi nilai selain `pending`/`resolved` — dikonfirmasi terhadap `reference/flowable-swagger-process.json` (`operationId: listTasks`); perhatikan Flowable mendokumentasikan `404`, BUKAN `400`, untuk kesalahan parameter pada endpoint ini |

Contoh implementasi yang terbukti jalan: lihat rujukan komposabel di akhir
[§3.2](#32-variabel-proses-tambahan-includeprocessvariables) di atas.

---

## 4. Endpoint Complete Task

**Method:** `POST`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}` (`{taskId}` didapat dari notifikasi Endpoint A sebagai `taskId`, atau dari field `id` hasil PULL §3)

**Header:** `Content-Type: application/json`, `Authorization: Basic <base64 username:password>` (sama seperti §2/§3)

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

| No. | Nilai `keputusanBod` yang dikirim | Hasil yang tercatat di `hasilApprovalBod` |
|---|---|---|
| 1 | `"TERIMA"` | `"DITERIMA"` |
| 2 | `"TOLAK"` | `"DITOLAK"` |
| 3 | `"REVISI"` | `"PERLU REVISI"` (perhatikan spasi, bukan "DIREVISI") |

Studio ini menyediakan 3 tombol aksi (Setujui/Tolak/Minta Revisi) di UI-nya
sendiri yang di baliknya mengirim `keputusanBod` persis dengan salah satu
dari 3 nilai di atas — kalau Aplikasi AWS membangun UI approval-nya
sendiri, kirim salah satu dari 3 nilai persis ini (case-sensitive), bukan
label tombol atau nilai lain.

### 4.2. Contoh request & respons

Contoh berikut memakai keputusan `TERIMA`.

#### Request

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

#### Respons

Tidak ada body — lihat tabel kode respons di bawah untuk cara membaca
hasil panggilan ini.

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Task berhasil di-complete (tidak ada body respons yang perlu dibaca — cukup cek status code) |
| 2 | `400 Bad Request` | Body request tidak valid (mis. `action` salah/hilang) |
| 3 | `404 Not Found` | `{taskId}` tidak ditemukan (mis. task sudah di-complete approver lain atau ID salah) |
| 4 | `409 Conflict` | Task sedang diubah bersamaan (concurrency) — lihat [§6.1 poin 1](#61-risiko) |

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

Sepuluh endpoint di bawah ini **opsional** — Flowable bawaan juga, tidak
perlu dibangun, tapi tidak wajib dipanggil supaya proses "Approval Berita
Acara" bisa berjalan (berbeda dari §2/§3/§4 yang wajib).

**§5.1–§5.5** dipakai kalau Aplikasi AWS butuh kemampuan operasional
tambahan: menghitung task aktif per grup (mis. untuk badge notifikasi
jumlah approval menunggu), melihat detail satu task langsung dari
`taskId`, mengecek apakah satu process instance masih berjalan atau
sudah selesai, menarik variabel prosesnya langsung, atau melihat riwayat
audit lengkap satu instance dari awal sampai akhir — kelimanya **sudah
dipakai & terbukti jalan** di Studio ini sendiri (lihat rujukan
komposabel di tiap subbagian).

**§5.6–§5.10** adalah **usulan tambahan** berdasarkan kemampuan yang
sudah tersedia di spesifikasi resmi Flowable tapi belum pernah menjadi
bagian kontrak API untuk proses "Approval Berita Acara" ini secara
spesifik: mengklaim/mendelegasikan/resolve task (§5.6), membatalkan
process instance (§5.7), menambahkan komentar/alasan pada task (§5.8),
mengambil diagram posisi process instance (§5.9), dan memantau failed
job (§5.10). Tiga di antaranya (§5.6/§5.7/§5.8) sudah terbukti jalan di
Studio ini lewat fitur umum yang tidak terkait proses ini secara
spesifik (Task/Kelola Proses/Komentar); dua sisanya (§5.9/§5.10) belum
pernah dipakai proyek ini sama sekali — lihat catatan verifikasi di
masing-masing subbagian sebelum diadopsi sebagai fitur Aplikasi AWS.

### 5.1. Hitung task aktif per grup

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup={key}&size=0`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Variasi dari endpoint PULL (§3) — `size=0` membuat Flowable tidak
mengembalikan daftar task-nya, cukup jumlah totalnya saja lewat field
`total` di respons (`DataResponseTaskResponse`, dikonfirmasi terhadap
`reference/flowable-swagger-process.json`).

#### 5.1.1. Contoh request & respons

Contoh berikut menghitung task menunggu untuk approver grup `"2"`.

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/tasks?candidateGroup=2&size=0
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK` — `data` sengaja kosong karena `size=0`, hanya `total` yang
relevan:

```json
{ "data": [], "total": 5, "start": 0, "sort": "id", "order": "asc", "size": 0 }
```

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Query berhasil — `total` berisi jumlah task yang cocok filter (`data` sengaja kosong karena `size=0`, lihat catatan di atas) |
| 2 | `404 Not Found` | Parameter query tidak valid (mis. `candidateGroup` kosong/format salah) — endpoint yang sama dengan [§3](#3-endpoint-pull-polling-task), lihat catatan kode respons di [§3.3](#33-contoh-request--respons) |

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

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Mengambil detail satu task langsung dari `{taskId}` — berguna kalau
Aplikasi AWS sudah tahu `taskId` (mis. dari payload notifikasi Endpoint A
di [KONTRAK-API-APLIKASI-AWS.md](./KONTRAK-API-APLIKASI-AWS.md)) dan ingin
detail lengkapnya tanpa query ulang lewat §3 dengan filter `candidateGroup`.

#### 5.2.1. Contoh request & respons

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/tasks/7a3f9c2e-1234-4abc-9def-0123456789ab
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK` — field yang sama seperti satu item `data` di respons §3,
lihat `FlowableTask` di `src/types/flowable.ts`:

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

| No. | Field respons | Keterangan |
|---|---|---|
| 1 | `id` | ID task ini — sama dengan `{taskId}` yang diminta, dipakai juga sebagai `{taskId}` saat memanggil **Complete Task** (§4) |
| 2 | `name` | Nama task BPMN (mis. "Approval BOD") |
| 3 | `processInstanceId` | ID process instance pemilik task ini — dipakai untuk §5.3/§5.4/§5.5 |
| 4 | `assignee` | Approver yang sudah meng-klaim task ini, atau `null` kalau belum diklaim |
| 5 | `owner` | Pemilik task (biasanya `null` untuk alur candidate-group biasa) |
| 6 | `delegationState` | Status delegasi task (`null` kalau tidak sedang didelegasikan) |
| 7 | `priority` | Prioritas task (angka, default `50`) |
| 8 | `dueDate` | Batas waktu task, atau `null` kalau tidak diset |

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Task ditemukan — body seperti contoh di atas |
| 2 | `404 Not Found` | Task tidak ditemukan (mis. sudah di-complete lewat §4, atau `taskId` salah) |

### 5.3. Cek status process instance (berjalan/selesai)

**Method:** `GET`

**URL (by ID):** `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/{processInstanceId}`

**URL (by Business Key):** `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances?businessKey={key}`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Dipakai untuk mengetahui apakah satu process instance **masih berjalan**
atau **sudah selesai**, tanpa menunggu Callback Hasil Approval (Endpoint B
di file sebelah):

#### 5.3.1. Mekanisme 2 langkah

1. Panggil `GET /runtime/process-instances/{id}` (atau `?businessKey=...`)
   — `200` + body berarti instance **masih berjalan**; `404` (untuk
   query by ID) atau `data: []` kosong (untuk query by businessKey)
   berarti **tidak sedang berjalan**, lanjut ke langkah 2.
2. Kalau tidak ditemukan di langkah 1, panggil `GET
   /history/historic-process-instances?processInstanceId={id}` (atau
   `?businessKey={key}`) — kalau ditemukan, instance **sudah selesai**
   (`endTime` terisi di respons, lihat `HistoricProcessInstance` di
   `src/types/flowable.ts`); kalau tidak ditemukan di sini juga, berarti
   ID/Business Key salah, atau instance belum pernah dibuat.

#### 5.3.2. Contoh request & respons

**Langkah 1 (masih berjalan):**

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK` — instance masih berjalan, body seperti berikut:

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

**Langkah 2 (fallback ke histori, kalau sudah selesai):**

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/history/historic-process-instances?processInstanceId=4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK` dengan `endTime` terisi — instance sudah selesai, body seperti
berikut:

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

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` (langkah 1) | Instance **masih berjalan** — body seperti contoh langkah 1 |
| 2 | `404 Not Found` (langkah 1, by ID) / `data: []` (langkah 1, by Business Key) | Tidak sedang berjalan — lanjut ke langkah 2 |
| 3 | `200 OK` + `endTime` terisi (langkah 2) | Instance **sudah selesai** — body seperti contoh langkah 2 |
| 4 | `200 OK` + `total: 0` (langkah 2) | ID/Business Key salah, atau instance belum pernah dibuat |

Contoh implementasi yang terbukti jalan (pola 2 langkah persis di atas,
termasuk fallback ke histori): `src/composables/useProcessTracking.ts`
(fungsi `search()`).

### 5.4. Ambil variabel proses langsung

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/{processInstanceId}/variables`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Mengambil semua variabel proses (`jenisKlien`, `nilaiProjectRupiah`,
`kategoriNilai`, `approvalBodLabel`, `hasilApprovalBod`, dst. — lihat
[§2.1](#21-variabel-proses-yang-wajib-dikirim) & [§3.2](#32-variabel-proses-tambahan-includeprocessvariables))
langsung dari satu process instance, tanpa lewat query task di §3. Berguna
kalau Aplikasi AWS sudah tahu `processInstanceId` (instance masih berjalan
— untuk instance yang **sudah selesai**, pakai riwayat aktivitas di §5.5
atau tunggu Callback Hasil Approval) dan hanya perlu nilai variabelnya,
bukan daftar task.

#### 5.4.1. Contoh request & respons

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd/variables
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK`, array `{name, value}` per variabel — lihat
`ProcessInstanceVariable` di `src/types/flowable.ts`:

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

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Instance ditemukan — array variabel seperti contoh di atas |
| 2 | `400 Bad Request` | `{processInstanceId}` tidak ditemukan (mis. instance sudah selesai — pakai §5.5 atau Callback Hasil Approval — atau ID salah) — dikonfirmasi terhadap `reference/flowable-swagger-process.json` (`operationId: listProcessInstanceVariables`); perhatikan Flowable mendokumentasikan ini sebagai `400`, BUKAN `404`, meski deskripsi resminya sendiri menyebut "not found" |

`hasilApprovalBod` di atas terisi sebagian (baru 2 dari 3 approver) karena
process instance ini masih berjalan — lihat [§2.1](#21-variabel-proses-yang-wajib-dikirim)
untuk arti tiap variabel dan [KONTRAK-API-APLIKASI-AWS.md §3](./KONTRAK-API-APLIKASI-AWS.md#3-endpoint-b--callback-hasil-approval)
untuk bentuk `hasilApprovalBod` lengkap setelah semua approver selesai.

Contoh implementasi yang terbukti jalan: `src/composables/useProcessTracking.ts`
(fungsi `fetchVariables()`).

### 5.5. Riwayat aktivitas proses (audit trail)

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/history/historic-activity-instances?processInstanceId={id}`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Mengambil seluruh langkah yang sudah dilalui satu process instance secara
berurutan, dari `StartEvent_1` sampai `EndEvent_1` — siapa mengerjakan apa,
kapan, dan berapa lama. Berguna untuk audit/compliance: menunjukkan
riwayat lengkap satu pengajuan approval dari awal sampai akhir, termasuk
approver mana yang memproses task kapan lewat Complete Task (§4).

#### 5.5.1. Contoh request & respons

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/history/historic-activity-instances?processInstanceId=4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK`, dipangkas jadi 3 langkah pertama untuk ringkas — instance
nyata akan berisi seluruh langkah sampai `EndEvent_1`:

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

| No. | Field respons | Keterangan |
|---|---|---|
| 1 | `activityId` / `activityName` | ID & nama elemen BPMN (mis. `UserTask_ApprovalBod` / "Approval BOD") |
| 2 | `activityType` | Jenis elemen (`userTask`, `serviceTask`, `startEvent`, `endEvent`, dst.) |
| 3 | `assignee` | Approver yang mengerjakan (untuk `userTask`) |
| 4 | `startTime` / `endTime` | Kapan langkah ini mulai & selesai |
| 5 | `durationInMillis` | Durasi langkah ini dalam milidetik |

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Query berhasil — `data` berisi seluruh langkah (kosong `[]` kalau `{processInstanceId}` tidak pernah ada, BUKAN error) |
| 2 | `400 Bad Request` | Parameter query tidak valid (mis. format `processInstanceId` salah) — dikonfirmasi terhadap `reference/flowable-swagger-process.json` (`operationId: listHistoricActivityInstances`) |

Lihat `HistoricActivityInstance` di `src/types/flowable.ts` untuk bentuk
lengkapnya. Contoh implementasi yang terbukti jalan (termasuk
pemformatan durasi & label jenis aktivitas ke Bahasa Indonesia):
`src/composables/useAuditTrail.ts`.

### 5.6. Aksi Lain pada Task (Claim, Delegate, Resolve)

**Method:** `POST`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}` (endpoint yang SAMA dengan **Complete Task**, §4 — bedanya cuma nilai `action` di body)

**Header:** `Content-Type: application/json`, `Authorization: Basic <base64 username:password>` (sama seperti §4)

Selain `action: "complete"` (§4.1), skema resmi `TaskActionRequest`
Flowable mendukung 3 nilai `action` lain yang belum dipakai proses
"Approval Berita Acara" ini — dikonfirmasi terhadap
`reference/flowable-swagger-process.json` (`operationId:
executeTaskAction`, definisi `TaskActionRequest`):

| No. | Action | Skenario | Parameter tambahan |
|---|---|---|---|
| 1 | `claim` | Mencegah 2 staf yang sama-sama berhak atas satu key `dynamicApproverGroups` (mis. `"bod-1"` dipetakan ke lebih dari satu orang di sisi Aplikasi AWS sendiri — ingat, Flowable tidak pernah memvalidasi isi grup ini, lihat [§2.2](#22-format-dynamicapprovergroups)) memproses task yang sama secara bersamaan | `assignee` (wajib) — user yang meng-klaim |
| 2 | `delegate` | Approver berhalangan (cuti/sakit) — alihkan sementara ke backup; approver asli tetap tercatat sebagai `owner` task, backup jadi `assignee` sementara | `assignee` (wajib) — user tujuan delegasi |
| 3 | `resolve` | Backup yang menerima delegasi (poin 2) mengembalikan task ke approver asli setelah selesai ditinjau, TANPA ikut memutuskan `keputusanBod` — approver asli lanjut memutuskan lewat **Complete Task** (§4) biasa | Tidak ada — `assignee` otomatis kembali ke `owner` |

#### 5.6.1. Contoh request & respons

Contoh berikut mendelegasikan task ke approver backup.

##### Request

```
POST {VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}
Content-Type: application/json
Authorization: Basic <base64 username:password>
```

```json
{
  "action": "delegate",
  "assignee": "backup.approver"
}
```

##### Respons

Sama seperti Complete Task (§4) — tidak ada body, cukup cek status code:

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Aksi berhasil dijalankan |
| 2 | `400 Bad Request` | Body tidak valid (mis. `action` bukan salah satu dari `complete`/`claim`/`delegate`/`resolve`, atau `assignee` kosong padahal wajib untuk `claim`/`delegate`) |
| 3 | `404 Not Found` | `{taskId}` tidak ditemukan |
| 4 | `409 Conflict` | Khusus `claim`: task sudah diklaim user lain |

Ketiga aksi ini **sudah terbukti jalan** di Studio ini sendiri (fitur
Task umum, bukan bagian kontrak proses "Approval Berita Acara" ini) —
`src/composables/useFlowableTasks.ts` (fungsi `claim()`, `delegate()`,
`resolveDelegation()`) — tapi belum pernah didokumentasikan atau
diusulkan sebagai bagian kontrak API Aplikasi AWS untuk proses ini
secara spesifik. Perlu didiskusikan dulu dengan tim Aplikasi AWS kalau
mau diadopsi: siapa yang berhak memicu `claim`/`delegate` (requester?
approver sendiri? admin?), dan bagaimana UI Aplikasi AWS menampilkan
status "sedang didelegasikan" ke penggunanya.

### 5.7. Batalkan Process Instance

**Method:** `DELETE`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/{processInstanceId}` (parameter `deleteReason` opsional lewat query string, lihat contoh)

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3 — tidak ada body request)

Membatalkan/menghapus satu process instance yang **masih berjalan**
secara PERMANEN (mis. requester menyadari data pengajuan salah, atau
butuh dibatalkan sebelum semua approver selesai) — dikonfirmasi terhadap
`reference/flowable-swagger-process.json` (`operationId:
deleteProcessInstance`). **Beda dari suspend** (bagian dari fitur
"Kelola Proses" di Studio ini) yang cuma menjeda sementara dan bisa
diaktifkan lagi — endpoint ini menghapus PERMANEN, tidak bisa
dilanjutkan lagi setelahnya, dan SEMUA task yang masih terbuka di
instance ini ikut hilang. Parameter `deleteReason` opsional tapi
disarankan selalu diisi — tersimpan di riwayat/histori (lihat
[§5.5](#55-riwayat-aktivitas-proses-audit-trail)) supaya jelas kenapa
instance ini berhenti, bukan karena selesai normal.

#### 5.7.1. Contoh request & respons

##### Request

```
DELETE {VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd?deleteReason=Dibatalkan+oleh+requester+-+data+salah
Authorization: Basic <base64 username:password>
```

##### Respons

`204 No Content` — tidak ada body, cukup cek status code:

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `204 No Content` | Instance berhasil dibatalkan — semua task terbuka di instance ini ikut terhapus, tidak bisa di-complete lagi lewat Complete Task (§4) |
| 2 | `404 Not Found` | `{processInstanceId}` tidak ditemukan (mis. sudah selesai normal, atau ID salah) |

Endpoint ini **sudah terbukti jalan** di Studio ini sendiri lewat fitur
"Kelola Proses" — `src/composables/useProcessControl.ts` (fungsi
`cancelInstance()`) — tapi belum pernah diekspos/didokumentasikan
sebagai bagian kontrak API yang bisa dipanggil Aplikasi AWS untuk proses
"Approval Berita Acara" ini secara spesifik. Perlu didiskusikan dulu
dengan tim Aplikasi AWS sebelum diadopsi: siapa yang berhak memicu
pembatalan (requester sendiri? admin? kombinasi validasi status
approval saat itu, mis. tidak boleh dibatalkan kalau sudah ada approver
yang TERIMA?), dan apakah requester perlu diberi tahu (lewat mekanisme
di luar Flowable) kalau pembatalannya berhasil.

### 5.8. Komentar/Alasan pada Task

**Method:** `POST` (tambah komentar) / `GET` (lihat daftar komentar)

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}/comments`

**Header:** `Content-Type: application/json` (khusus `POST`), `Authorization: Basic <base64 username:password>` (sama seperti §4)

Menambahkan/melihat catatan bebas teks pada satu task — dikonfirmasi
terhadap `reference/flowable-swagger-process.json` (`operationId:
createTaskComments`/`listTaskComments`). Berguna untuk melengkapi
keputusan `keputusanBod` ([§4.1](#41-variabel-yang-wajib-dikirim-saat-complete))
yang saat ini **cuma enum** (`TERIMA`/`TOLAK`/`REVISI`) **tanpa field
alasan bebas teks** — untuk hasil `TOLAK`/`REVISI` khususnya, requester
biasanya perlu tahu alasannya. Komentar tetap tersimpan walau task-nya
sudah selesai (bisa diambil lagi lewat `GET
/history/historic-process-instances/{processInstanceId}/comments`
setelah instance selesai — endpoint terpisah, tidak dibahas detail di
sini).

#### 5.8.1. Contoh request & respons

Contoh berikut menambahkan alasan pada task sebelum di-complete dengan
keputusan `REVISI`.

##### Request

```
POST {VITE_FLOWABLE_BASE_URL}/runtime/tasks/{taskId}/comments
Content-Type: application/json
Authorization: Basic <base64 username:password>
```

```json
{
  "message": "Perlu revisi: lampiran Berita Acara belum ditandatangani pihak kedua.",
  "author": "budi.approver"
}
```

##### Respons

`201 Created` — komentar berhasil dibuat:

```json
{
  "id": "10",
  "author": "budi.approver",
  "message": "Perlu revisi: lampiran Berita Acara belum ditandatangani pihak kedua.",
  "time": "2026-09-04T10:30:00.000+0000",
  "taskId": "7a3f9c2e-1234-4abc-9def-0123456789ab"
}
```

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `201 Created` | Komentar berhasil dibuat — body seperti contoh di atas |
| 2 | `400 Bad Request` | Field `message` kosong/hilang dari body |
| 3 | `404 Not Found` | `{taskId}` tidak ditemukan |

Endpoint ini **sudah terbukti jalan** di Studio ini sendiri (fitur
"Komentar") — `src/composables/useTaskComments.ts` (fungsi
`addComment()`/`loadComments()`) — tapi belum pernah
diekspos/didokumentasikan sebagai bagian kontrak API Aplikasi AWS untuk
proses ini. Kalau diadopsi, urutan yang disarankan: Aplikasi AWS
memanggil endpoint ini SEBELUM memanggil Complete Task (§4) untuk
keputusan `TOLAK`/`REVISI` (sisipkan alasan dulu, baru selesaikan
task-nya) — bukan sesudahnya, karena task yang sudah di-complete tidak
bisa ditambah komentar baru lewat endpoint runtime ini (harus lewat
endpoint historic yang terpisah).

### 5.9. Diagram Posisi Process Instance

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/runtime/process-instances/{processInstanceId}/diagram`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Mengambil gambar diagram BPMN (format PNG, dikirim sebagai byte array)
dengan elemen yang sedang aktif ditandai — mirip fitur "Status Proses"
yang sudah ada di Studio ini, tapi Studio membuat visualisasinya sendiri
secara client-side (overlay di kanvas bpmn-js dari data
[§5.5](#55-riwayat-aktivitas-proses-audit-trail)/[§5.3](#53-cek-status-process-instance-berjalanselesai)),
BUKAN dari endpoint ini — dikonfirmasi terhadap
`reference/flowable-swagger-process.json` (`operationId:
getProcessInstanceDiagram`). Berguna kalau Aplikasi AWS ingin
menampilkan diagram progres visual ("Anda di sini") langsung di UI
mereka sendiri tanpa perlu membangun renderer BPMN sendiri seperti
Studio ini.

#### 5.9.1. Contoh request & respons

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/runtime/process-instances/4b8e1d0a-5678-4cde-8f01-23456789abcd/diagram
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK` — body berupa gambar PNG mentah (`Content-Type: image/png`),
BUKAN JSON, jadi tidak ada contoh body untuk endpoint ini:

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Diagram berhasil dibuat — body PNG mentah |
| 2 | `400 Bad Request` | Process definition-nya tidak punya informasi grafis (BPMN DI) — tidak mungkin terjadi untuk proses ini karena `examples/approval-berita-acara.bpmn` sudah punya diagram lengkap |
| 3 | `404 Not Found` | `{processInstanceId}` tidak ditemukan |

**Belum diverifikasi ke server produksi & belum ada implementasi di
Studio ini** — endpoint ini dikonfirmasi ADA di spesifikasi resmi
Flowable, tapi tidak pernah dipakai proyek ini sama sekali (Studio ini
sengaja memilih rendering client-side sendiri, lihat di atas). Kalau
Aplikasi AWS mau memakainya, perhatikan responsnya BUKAN JSON (byte
array gambar) — beda dari semua endpoint lain di dokumen ini — pastikan
sisi Aplikasi AWS menangani `Content-Type: image/png`, bukan mencoba
mem-parsing-nya sebagai JSON.

### 5.10. Monitoring Failed Job (Dead-Letter)

**Method:** `GET`

**URL:** `{VITE_FLOWABLE_BASE_URL}/management/deadletter-jobs?processInstanceId={id}`

**Header:** `Authorization: Basic <base64 username:password>` (sama seperti §3)

Menarik daftar "dead-letter job" — job asinkron yang GAGAL dieksekusi
berulang kali sampai kehabisan retry (mis. Task Listener HTTP ke
Endpoint A gagal terus-menerus, kalau konfigurasi listener-nya
asinkron) — dikonfirmasi terhadap `reference/flowable-swagger-process.json`
(`operationId: listDeadLetterJobs`). **Relevan langsung dengan risiko
yang sudah dicatat** di [§6.1 poin 3](#61-risiko) &
[KONTRAK-API-APLIKASI-AWS.md §4](./KONTRAK-API-APLIKASI-AWS.md#4-risiko--hal-yang-belum-teruji-di-server-produksi)
soal keandalan panggilan HTTP dari Task Listener yang belum diuji —
endpoint ini memberi cara memantau kalau ada panggilan yang gagal diam-diam,
tanpa perlu akses langsung ke database/server Flowable.

#### 5.10.1. Contoh request & respons

##### Request

```
GET {VITE_FLOWABLE_BASE_URL}/management/deadletter-jobs?processInstanceId=4b8e1d0a-5678-4cde-8f01-23456789abcd
Authorization: Basic <base64 username:password>
```

##### Respons

`200 OK` — array job yang gagal (kosong `[]` kalau tidak ada, BUKAN
error):

```json
{
  "data": [
    {
      "id": "8",
      "processInstanceId": "4b8e1d0a-5678-4cde-8f01-23456789abcd",
      "elementId": "Task_KirimNotifikasiEndpointA",
      "elementName": "Kirim Notifikasi ke Endpoint A",
      "exceptionMessage": "Connection refused"
    }
  ],
  "total": 1
}
```

| No. | Kode respons | Arti |
|---|---|---|
| 1 | `200 OK` | Query berhasil — `data` kosong (`[]`) kalau tidak ada job yang gagal, BUKAN error |
| 2 | `400 Bad Request` | Parameter query tidak valid, atau kombinasi `timersOnly`+`messagesOnly` dipakai bersamaan (keduanya saling eksklusif) |

**Belum diverifikasi ke server produksi & belum ada implementasi di
Studio ini** — endpoint ini dikonfirmasi ADA di spesifikasi resmi
Flowable, tapi tidak pernah dipakai proyek ini sama sekali, dan **belum
pernah dikonfirmasi apakah Task Listener HTTP proses ini benar-benar
berjalan asinkron** (kalau sinkron, kegagalannya TIDAK akan pernah
muncul di sini — job dead-letter hanya berlaku untuk eksekusi asinkron).
Perlu dikonfirmasi dulu konfigurasi listener-nya sebelum endpoint ini
benar-benar berguna untuk memantau risiko di [§6.1 poin 3](#61-risiko).

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

1. [ ] Bangun **Endpoint A** — Notifikasi Task Baru (lihat [KONTRAK-API-APLIKASI-AWS.md §2](./KONTRAK-API-APLIKASI-AWS.md#2-endpoint-a--notifikasi-task-baru))
2. [ ] Bangun **Endpoint B** — Callback Hasil Approval (lihat [KONTRAK-API-APLIKASI-AWS.md §3](./KONTRAK-API-APLIKASI-AWS.md#3-endpoint-b--callback-hasil-approval))
3. [ ] Konfirmasi nama field `ignoreException` valid di server Flowable Anda (lihat [KONTRAK-API-APLIKASI-AWS.md §4](./KONTRAK-API-APLIKASI-AWS.md#4-risiko--hal-yang-belum-teruji-di-server-produksi), poin 1)
4. [ ] Konfirmasi panggilan HTTP dari dalam Task Listener (Endpoint A) diizinkan JVM server Flowable Anda (rujukan sama dengan item 3 di atas, poin 2)
5. [ ] Implementasikan mekanisme **PULL/polling** dengan interval sesuai SLA Aplikasi AWS sendiri — jangan mengandalkan PUSH saja (lihat [§3](#3-endpoint-pull-polling-task) di dokumen ini)
6. [ ] Implementasikan pemanggilan **Complete Task** setelah approver menentukan keputusan, termasuk penanganan `409 Conflict` (retry singkat) (lihat [§4](#4-endpoint-complete-task) di dokumen ini)
7. [ ] Setelah Endpoint A/B siap & disepakati, ganti URL placeholder di BPMN (lihat [KONTRAK-API-APLIKASI-AWS.md §5](./KONTRAK-API-APLIKASI-AWS.md#5-cara-mengganti-url-placeholder-setelah-kontrak-tersedia))
8. [ ] Uji end-to-end: Start Instance → task baru terbuat & Endpoint A terpanggil (atau terlewat, ketahuan lewat PULL) → Complete Task per approver (perhatikan risiko concurrency di [§6.1](#61-risiko) kalau pengujian melibatkan banyak approver sekaligus) → Endpoint B terpanggil dengan `hasilApprovalBod` lengkap setelah semua approver selesai (lihat [§2](#2-endpoint-start-instance), [§3](#3-endpoint-pull-polling-task) & [§4](#4-endpoint-complete-task) di dokumen ini)
9. [ ] Validasi `jenisKlien` di sisi Aplikasi AWS SEBELUM memanggil Start Instance — harus **persis** `"Baru"`, `"Existing"`, atau `"Tender"` (case-sensitive), bukan sekadar "ada isinya" (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) & [§6.1](#61-risiko) poin 2)
10. [ ] Pastikan `examples/approval-berita-acara.bpmn` DAN `examples/keputusan-approval-bod.dmn` sudah ter-deploy ke server Flowable sebelum go-live/pengujian pertama (lihat [§2.4](#24-keputusan-dmn-internal--prasyarat-deployment) poin 5)
11. [ ] *(opsional)* Implementasikan endpoint monitoring/operasional tambahan sesuai kebutuhan (badge hitung task, detail task, cek status instance, ambil variabel langsung, audit trail) (lihat [§5](#5-endpoint-monitoring--operasional-tambahan) di dokumen ini)

