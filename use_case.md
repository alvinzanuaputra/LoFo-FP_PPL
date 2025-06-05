# Use Case Diagrams - Lost & Found System

## UCD-01: Autentikasi dan Manajemen Pengguna

```plantuml
left to right direction
actor "Pengguna" as user
actor "Admin" as admin
rectangle "Manajemen Pengguna" {
  usecase "Registrasi Pengguna Baru" as reg
  usecase "Login" as login
  usecase "Manajemen Profil Pengguna" as manageProfile
  usecase "Logout" as logout
  usecase "Validasi Email/Username" as validateEmail
  usecase "Validasi Kredensial" as validateCreds
  usecase "Manajemen Pengguna" as manageUsers
  user --> reg
  user --> login
  user --> manageProfile
  user --> logout
  reg ..> validateEmail : <<include>>
  login ..> validateCreds : <<include>>
  admin --> manageUsers
}
```

## UCD-02: Manajemen Barang

```plantuml
left to right direction
actor "Pengguna" as user
rectangle "Manajemen Barang" {
  usecase "Posting Barang Hilang" as postLost
  usecase "Posting Barang Temuan" as postFound
  usecase "Edit Postingan Barang" as editPost
  usecase "Hapus Postingan Barang" as deletePost
  usecase "Pencarian Barang" as searchItem
  usecase "Lihat Detail Barang" as viewDetail
  user --> postLost
  user --> postFound
  user --> editPost
  user --> deletePost
  user --> searchItem
  user --> viewDetail
}
```

## UCD-03: Komunikasi

```plantuml
left to right direction
actor "Pengguna" as user
rectangle "Komunikasi" {
  usecase "Sistem Chat Real-time" as chat
  usecase "Notifikasi Pesan Baru" as newMsgNotif
  usecase "Notifikasi Update Barang" as updateItemNotif
  usecase "History Percakapan" as chatHistory
  user --> chat
  user --> chatHistory
  "Sistem" --> newMsgNotif
  "Sistem" --> updateItemNotif
}
```

## UCD-04: Notifikasi dan Update

```plantuml
left to right direction
actor "Pengguna" as user
actor "Sistem Notifikasi" as system
rectangle "Notifikasi dan Update" {
  usecase "Push Notifikasi Kecocokan Barang" as pushNotif
  usecase "Update Status Barang (Ditemukan/Dikembalikan)" as updateStatus
  usecase "History Aktivitas Pengguna" as activityHistory
  system --> pushNotif
  user --> updateStatus
  user --> activityHistory
}
```