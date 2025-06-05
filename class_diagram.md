# Class Diagram - Lost & Found System

```plantuml
@startuml LoFo_Compact

class Pengguna {
    userId: String
    username: String
    email: String
    ---
    login()
    logout()
    updateProfile()
}

class ManagerAuth {
    -instance: ManagerAuth
    ---
    +getInstance()
    +authenticate()
}

interface StrategiAuth {
    authenticate()
}

class AuthEmail {
    authenticate()
}

abstract class Barang {
    itemId: String
    judul: String
    kategori: String
    ---
    getDetail()
    updateStatus()
}

class BarangHilang {
    tanggalHilang: Date
    ---
    laporHilang()
    markFound()
}

class BarangDitemukan {
    tanggalDitemukan: Date
    ---
    laporDitemukan()
    markReturned()
}

class PabrikBarang {
    +createItem()
}

class Chat {
    chatId: String
    user1: Pengguna
    user2: Pengguna
    ---
    kirimPesan()
    getHistory()
}

class Pesan {
    messageId: String
    konten: String
    sender: Pengguna
    timestamp: Date
}

class Lokasi {
    latitude: Double
    longitude: Double
    alamat: String
    ---
    hitungJarak()
}

interface RepoBarang {
    save()
    findById()
    findByCategory()
}

class RepoLokal {
    localDb: Database
    ---
    save()
    findById()
}

class FacadeLoFo {
    authManager: ManagerAuth
    itemRepo: RepoBarang
    ---
    authenticateUser()
    postItem()
    searchItems()
    startChat()
}

' Relationships
Pengguna ||--o{ Barang : "owns"
Pengguna ||--o{ Chat : "participates"
Chat ||--o{ Pesan : "contains"
Barang --> Lokasi : "has location"

' Inheritance
BarangHilang --|> Barang
BarangDitemukan --|> Barang
AuthEmail ..|> StrategiAuth
RepoLokal ..|> RepoBarang

' Associations
ManagerAuth --> StrategiAuth : "uses"
FacadeLoFo --> ManagerAuth : "uses"
FacadeLoFo --> RepoBarang : "uses"
PabrikBarang ..> Barang : "creates"

@enduml