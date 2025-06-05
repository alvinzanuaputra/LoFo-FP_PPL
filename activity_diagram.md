# Sequence Diagrams - Lost & Found System

## UCD-01: Autentikasi dan Manajemen Pengguna

### SD-01: Registrasi Pengguna
```
title SD-01: Registrasi Pengguna

actor User
boundary "Registration Form" as RF
control "Registration Controller" as RC
entity "User Account" as UA
control "Email Service" as ES
database "User Database" as DB

User->RF: showRegistrationForm()
RF->RC: registerAccount()
RC->UA: validateData()
UA->DB: saveUser()
DB->UA: userSaved
UA->ES: sendWelcome()
ES->UA: emailSent
UA->RC: userCreated
RC->RF: success
RF->User: confirmation
```

### SD-02: Login Pengguna
```
title SD-02: Login Pengguna

actor User
boundary "Login Form" as LF
control "Auth Controller" as AC
entity "User Account" as UA
control "Session Manager" as SM
database "User Database" as DB

User->LF: showLoginForm()
LF->AC: enterCredentials()
AC->UA: validateLogin()
UA->DB: checkCredentials()
DB->UA: credentialsValid
UA->AC: loginValid
AC->SM: createSession()
SM->AC: sessionCreated
AC->LF: loginSuccess
LF->User: redirectToDashboard()
```

### SD-03: Login dengan Social Media
```
title SD-03: Login dengan Social Media

actor User
boundary "Login Form" as LF
control "Social Auth Provider" as SAP
control "Auth Controller" as AC
entity "User Account" as UA
control "Session Manager" as SM
database "User Database" as DB

User->LF: selectSocialLogin()
LF->SAP: redirectToProvider()
SAP->User: requestPermission()
User->SAP: grantPermission()
SAP->AC: returnAuthCode()
AC->SAP: exchangeToken()
SAP->AC: userProfile
AC->UA: findOrCreateUser()
UA->DB: saveUserData()
DB->UA: userReady
UA->SM: createSession()
SM->UA: sessionCreated
UA->AC: userAuthenticated
AC->LF: loginSuccess
LF->User: redirectToDashboard()
```

### SD-04: Manajemen Profil
```
title SD-04: Manajemen Profil

actor User
boundary "Profile Form" as PF
control "Profile Controller" as PC
entity "User Account" as UA
database "User Database" as DB

User->PF: showProfileForm()
PF->PC: editProfile()
PC->UA: updateProfile()
UA->DB: validateData()
DB->UA: profileSaved
UA->PC: updateSuccess
PC->PF: success
PF->User: profileUpdated()
```

### SD-05: Logout
```
title SD-05: Logout Pengguna

actor User
boundary Dashboard
control "Auth Controller" as AC
control "Session Manager" as SM
database "Activity Database" as AD

User->Dashboard: clickLogout()
Dashboard->AC: logout()
AC->SM: destroySession()
SM->AD: logActivity()
AD->SM: activityLogged
SM->AC: sessionDestroyed
AC->Dashboard: logoutSuccess
Dashboard->User: redirectToLogin()
```

## UCD-02: Manajemen Barang

### SD-06: Posting Barang Hilang
```
title SD-06: Posting Barang Hilang

actor User
boundary "Post Form" as PF
control "Upload Service" as US
control "Item Controller" as IC
database "Item Database" as ID
control "Notification Service" as NS

User->PF: showPostForm()
PF->US: createLostItem()
US->IC: uploadPhoto()
IC->ID: validateItem()
ID->IC: saveItem()
IC->NS: notifyPotentialMatches()
NS->IC: notificationSent
IC->US: itemSaved
US->PF: postSuccess
PF->User: itemPosted()
```

### SD-07: Posting Barang Temuan
```
title SD-07: Posting Barang Temuan

actor User
boundary "Post Form" as PF
control "Upload Service" as US
control "Item Controller" as IC
database "Item Database" as ID
control "Match Service" as MS
control "Notification Service" as NS

User->PF: showPostForm()
PF->US: createFoundItem()
US->IC: uploadPhoto()
IC->ID: validateItem()
ID->IC: saveItem()
IC->MS: findPotentialMatches()
MS->ID: searchSimilarItems()
ID->MS: matchingItems
MS->NS: notifyItemOwners()
NS->MS: notificationsSent
MS->IC: matchingComplete
IC->US: postSuccess
US->PF: itemPosted()
PF->User: postComplete()
```

### SD-08: Edit Postingan Barang
```
title SD-08: Edit Postingan Barang

actor User
boundary "Item Detail" as ID
boundary "Edit Form" as EF
control "Upload Service" as US
control "Item Controller" as IC
database "Item Database" as IDB

User->ID: editItem()
ID->EF: loadItemData()
EF->US: uploadNewPhoto()
US->EF: photoUpdated
EF->IC: updateItem()
IC->IDB: saveChanges()
IDB->IC: itemUpdated
IC->EF: updateSuccess
EF->User: itemEdited()
```

### SD-09: Hapus Postingan Barang
```
title SD-09: Hapus Postingan Barang

actor User
boundary "Item Detail" as ID
control "Item Controller" as IC
database "Item Database" as IDB
control "File Service" as FS

User->ID: deleteItem()
ID->IC: confirmDelete()
IC->IDB: deleteItem()
IDB->IC: itemDeleted
IC->FS: deletePhotos()
FS->IC: photosDeleted
IC->ID: deleteSuccess
ID->User: itemRemoved()
```

### SD-10: Pencarian Barang
```
title SD-10: Pencarian Barang

actor User
boundary "Search Form" as SF
control "Search Controller" as SC
database "Item Database" as ID
control "Filter Service" as FS

User->SF: enterSearchCriteria()
SF->SC: performSearch()
SC->ID: queryItems()
ID->SC: searchResults
SC->FS: applyFilters()
FS->SC: filteredResults
SC->SF: displayResults
SF->User: showSearchResults()
```

### SD-11: View Detail Barang
```
title SD-11: View Detail Barang

actor User
boundary "Item List" as IL
control "Item Controller" as IC
database "Item Database" as ID
control "View Counter" as VC
boundary "Item Detail" as IDetail

User->IL: selectItem()
IL->IC: getItemDetail()
IC->ID: fetchItemData()
ID->IC: itemData
IC->VC: incrementViews()
VC->IC: viewCounted
IC->IDetail: displayItem
IDetail->User: showItemDetail()
```

## UCD-03: Komunikasi

### SD-12: Inisiasi Chat
```
title SD-12: Inisiasi Chat

actor User
boundary "Item Detail" as ID
control "Chat Controller" as CC
database "Chat Database" as CD
control "Chat Service" as CS
boundary "Chat Interface" as CI

User->ID: contactOwner()
ID->CC: initializeChat()
CC->CD: createChatRoom()
CD->CC: chatRoomCreated
CC->CS: setupWebSocket()
CS->CC: connectionReady
CC->CI: openChat
CI->User: chatReady()
```

### SD-13: Mengirim Pesan
```
title SD-13: Mengirim Pesan Real-time

actor User
boundary "Chat Interface" as CI
control "Chat Service" as CS
database "Message Database" as MD
control "WebSocket Server" as WSS
boundary "Recipient Chat" as RC
control "Push Service" as PS

User->CI: typeMessage()
CI->CS: sendMessage()
CS->MD: saveMessage()
MD->CS: messageSaved
CS->WSS: broadcastMessage()
WSS->RC: deliverMessage()
RC->WSS: messageReceived
WSS->PS: sendNotification()
PS->WSS: notificationSent
CS->CI: messageDelivered
CI->User: messageConfirmed()
```

### SD-14: Menerima Pesan
```
title SD-14: Menerima Pesan

control "WebSocket Server" as WSS
boundary "Chat Interface" as CI
control "Chat Service" as CS
database "Message Database" as MD
control "Notification Service" as NS
actor User

WSS->CI: newMessageReceived()
CI->CS: processMessage()
CS->MD: markAsRead()
MD->CS: statusUpdated
CS->NS: updateBadgeCount()
NS->CS: badgeUpdated
CS->CI: displayMessage
CI->User: showNewMessage()
```

### SD-15: History Percakapan
```
title SD-15: History Percakapan

actor User
boundary "Chat List" as CL
control "Chat Controller" as CC
database "Message Database" as MD
control "Chat Service" as CS

User->CL: viewChatHistory()
CL->CC: getChatHistory()
CC->MD: fetchMessages()
MD->CC: messageHistory
CC->CS: formatMessages()
CS->CC: formattedMessages
CC->CL: displayHistory
CL->User: showChatHistory()
```

## UCD-04: Notifikasi dan Update

### SD-16: Push Notification untuk Match
```
title SD-16: Push Notification untuk Kecocokan Barang

control "Match Service" as MS
database "Item Database" as ID
control "Notification Controller" as NC
control "Push Service" as PS
entity "User Device" as UD

MS->ID: findItemMatch()
ID->MS: matchFound
MS->NC: createMatchNotification()
NC->PS: sendPushNotification()
PS->UD: deliverNotification()
UD->PS: notificationDelivered
PS->NC: deliveryConfirmed
NC->ID: logNotification()
ID->NC: notificationLogged
```

### SD-17: Update Status Barang
```
title SD-17: Update Status Barang

actor User
boundary "Item Detail" as ID
control "Item Controller" as IC
database "Item Database" as IDB
control "Notification Service" as NS
control "Push Service" as PS

User->ID: updateStatus()
ID->IC: changeStatus()
IC->IDB: updateItemStatus()
IDB->IC: statusUpdated
IC->NS: notifyInterestedUsers()
NS->IDB: getInterestedUsers()
IDB->NS: userList
NS->PS: sendStatusUpdate()
PS->NS: notificationsSent
NS->IC: notificationsComplete
IC->ID: statusChanged
ID->User: confirmStatusUpdate()
```

### SD-18: History Aktivitas
```
title SD-18: History Aktivitas Pengguna

actor User
boundary "Profile Page" as PP
control "Activity Controller" as AC
database "Activity Database" as AD
control "Activity Service" as AS

User->PP: viewActivityHistory()
PP->AC: getUserActivity()
AC->AD: fetchUserActivities()
AD->AC: activityData
AC->AS: formatActivities()
AS->AC: formattedData
AC->PP: displayActivities
PP->User: showActivityHistory()
```

### SD-19: Notifikasi Pesan Baru
```
title SD-19: Notifikasi Pesan Baru

control "Chat Service" as CS
control "Notification Controller" as NC
database "User Database" as UD
control "Push Service" as PS
entity "User Device" as UDevice
boundary "Mobile App" as MA

CS->NC: newMessageAlert()
NC->UD: checkUserPreferences()
UD->NC: notificationSettings
NC->PS: sendMessageNotification()
PS->UDevice: deliverNotification()
UDevice->MA: displayNotification()
MA->UDevice: notificationShown
UDevice->PS: notificationRead
PS->UD: updateNotificationStatus()
UD->PS: statusUpdated
```

---

## Stereotypes yang Digunakan:

- **actor**: User/Pengguna (stick figure)
- **boundary**: Interface/Form/UI (lingkaran dengan garis)
- **control**: Controller/Service/Logic (lingkaran dengan panah)
- **entity**: Model/Domain Object (lingkaran dengan garis bawah)
- **database**: Database/Repository (silinder)

## Cara Menggunakan:

1. Buka https://sequencediagram.org/
2. Copy salah satu kode di atas
3. Paste ke editor
4. Klik "Build Diagram" untuk melihat hasilnya
5. Anda bisa export sebagai PNG atau SVG

Setiap diagram akan menampilkan simbol yang berbeda sesuai dengan peran masing-masing komponen dalam sistem. 