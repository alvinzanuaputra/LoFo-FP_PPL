# LoFo-FP_PPL


Struktur Project :

```bash
lofo-project/
│
├── app/
│   ├── Console/
│   │   └── Kernel.php
│   ├── Exceptions/
│   │   └── Handler.php
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Auth/
│   │   │   │   └── AuthController.php
│   │   │   ├── Api/
│   │   │   │   ├── LostItemApiController.php
│   │   │   │   └── FoundItemApiController.php
│   │   │   ├── DashboardController.php
│   │   │   ├── LostItemController.php
│   │   │   ├── FoundItemController.php
│   │   │   ├── ChatController.php
│   │   │   ├── SearchController.php
│   │   │   └── NotificationController.php
│   │   ├── Middleware/
│   │   │   ├── CheckItemOwnership.php
│   │   │   └── TrackActivity.php
│   │   ├── Requests/
│   │   │   ├── StoreLostItemRequest.php
│   │   │   └── StoreFoundItemRequest.php
│   │   └── Resources/
│   │       ├── LostItemResource.php
│   │       └── UserResource.php
│   ├── Models/
│   │   ├── User.php
│   │   ├── LostItem.php
│   │   ├── FoundItem.php
│   │   ├── Category.php
│   │   ├── Conversation.php
│   │   ├── Message.php
│   │   └── Notification.php
│   ├── Services/
│   │   ├── ImageService.php
│   │   ├── MatchingService.php
│   │   ├── NotificationService.php
│   │   └── ItemStatusService.php
│   ├── Events/
│   │   ├── ItemMatchFound.php
│   │   └── NewMessageNotification.php
│   └── Providers/
│       ├── AppServiceProvider.php
│       ├── AuthServiceProvider.php
│       └── EventServiceProvider.php
│
├── config/
│   ├── app.php
│   ├── database.php
│   └── services.php
│
├── database/
│   ├── migrations/
│   │   ├── 2024_01_01_create_users_table.php
│   │   ├── 2024_01_01_create_lost_items_table.php
│   │   ├── 2024_01_01_create_found_items_table.php
│   │   ├── 2024_01_01_create_categories_table.php
│   │   ├── 2024_01_01_create_conversations_table.php
│   │   └── 2024_01_01_create_messages_table.php
│   ├── seeders/
│   │   ├── UserSeeder.php
│   │   ├── CategorySeeder.php
│   │   └── DemoDataSeeder.php
│   └── factories/
│       ├── UserFactory.php
│       ├── LostItemFactory.php
│       └── FoundItemFactory.php
│
├── public/
│   ├── css/
│   ├── js/
│   └── images/
│
├── resources/
│   ├── views/
│   │   ├── layouts/
│   │   │   └── app.blade.php
│   │   ├── auth/
│   │   │   ├── login.blade.php
│   │   │   └── register.blade.php
│   │   ├── dashboard/
│   │   │   └── index.blade.php
│   │   ├── lost-items/
│   │   │   ├── index.blade.php
│   │   │   ├── create.blade.php
│   │   │   └── show.blade.php
│   │   ├── found-items/
│   │   │   ├── index.blade.php
│   │   │   ├── create.blade.php
│   │   │   └── show.blade.php
│   │   ├── chat/
│   │   │   ├── index.blade.php
│   │   │   └── conversation.blade.php
│   │   └── notifications/
│   │       └── index.blade.php
│   └── lang/
│       └── id/
│           └── validation.php
│
├── routes/
│   ├── web.php
│   ├── api.php
│   └── channels.php
│
├── storage/
│   ├── app/
│   ├── framework/
│   └── logs/
│
├── tests/
│   ├── Feature/
│   │   ├── LostItemTest.php
│   │   └── AuthenticationTest.php
│   └── Unit/
│       ├── MatchingServiceTest.php
│       └── NotificationServiceTest.php
│
├── .env
├── .env.example
├── .gitignore
├── composer.json
├── package.json
└── README.md
```





```bash
lofo-project/
├── app/
│   ├── Console/
│   ├── Exceptions/
│   ├── Http/
│   ├── Models/
│   ├── Services/
│   ├── Events/
│   ├── Providers/
│   │
│   ├── Patterns/                       # Direktori baru untuk Design Patterns
│   │   ├── Creational/                 # Pola desain creational
│   │   │   ├── Factories/              # Factory Method Pattern
│   │   │   │   ├── BarangFactory.php
│   │   │   │   ├── BarangHilangFactory.php
│   │   │   │   └── BarangTemuanFactory.php
│   │   │   │
│   │   │   └── Singleton/              # Singleton Pattern
│   │   │       └── ManagerAuth.php
│   │   │
│   │   ├── Structural/                 # Pola desain structural
│   │   │   ├── Facade/                 # Facade Pattern
│   │   │   │   └── FacadeLoFo.php
│   │   │   │
│   │   │   └── Repository/             # Repository Pattern
│   │   │       ├── Contracts/
│   │   │       │   └── RepoBarang.php
│   │   │       └── Implementations/
│   │   │           └── RepoLokal.php
│   │   │
│   │   └── Behavioral/                 # Pola desain behavioral
│   │       ├── Strategy/               # Strategy Pattern
│   │       │   ├── Contracts/
│   │       │   │   └── StrategiAuth.php
│   │       │   └── Implementations/
│   │       │       ├── AuthEmail.php
│   │       │       └── AuthSocial.php
│   │       │
│   │       ├── Observer/               # Observer Pattern
│   │       │   ├── Observers/
│   │       │   │   └── BarangObserver.php
│   │       │   └── Listeners/
│   │       │       └── BarangCreatedListener.php
│   │       │
│   │       └── Command/                # Command Pattern
│   │           ├── Abstracts/
│   │           │   └── ChatCommand.php
│   │           └── Implementations/
│   │               └── SendMessageCommand.php
│   │
│   └── Contracts/                      # Antarmuka untuk dependency injection
│       ├── Repositories/
│       └── Services/
│
├── config/
│   ├── patterns.php                    # Konfigurasi khusus untuk design patterns
│   └── ...
│
└── ...
```






```bash 

# Untuk autentikasi endpoint 
POST /api/auth/register
POST /api/auth/login
POST /api/auth/logout
POST /api/auth/social/{provider}
GET  /api/auth/user

# Barang Management
GET    /api/barang                    # List all items
POST   /api/barang                    # Create new item
GET    /api/barang/{id}               # Get item details
PUT    /api/barang/{id}               # Update item
DELETE /api/barang/{id}               # Delete item
GET    /api/barang/search             # Search items
GET    /api/barang/category/{type}    # Get by category

# Chat System
GET    /api/chats                     # User's chats
POST   /api/chats                     # Create new chat
GET    /api/chats/{id}/messages       # Get messages
POST   /api/chats/{id}/messages       # Send message
	
# Notifikasi 
GET    /api/notifications             # Get user notifications
POST   /api/notifications/read/{id}   # Mark as read

```