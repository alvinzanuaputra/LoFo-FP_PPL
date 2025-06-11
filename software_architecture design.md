# Arsitektur MVC Laravel untuk Aplikasi LoFo (Lost and Found)

## Overview Arsitektur MVC

Model-View-Controller (MVC) adalah pola desain arsitektur yang memisahkan aplikasi menjadi tiga komponen utama yang saling terhubung namun independen:

- **Model**: Mengelola data dan logika bisnis
- **View**: Menangani presentasi data dan antarmuka pengguna
- **Controller**: Mengatur alur aplikasi dan komunikasi antara Model dan View

## 1. MODEL LAYER

### 1.1 User Model
```php
// app/Models/User.php
class User extends Authenticatable
{
    protected $fillable = [
        'name', 'email', 'phone', 'avatar', 'email_verified_at'
    ];
    
    // Relationships
    public function lostItems() {
        return $this->hasMany(LostItem::class);
    }
    
    public function foundItems() {
        return $this->hasMany(FoundItem::class);
    }
    
    public function conversations() {
        return $this->hasMany(Conversation::class);
    }
    
    public function notifications() {
        return $this->hasMany(Notification::class);
    }
}
```



### 1.2 Item Models
```php
// app/Models/LostItem.php
class LostItem extends Model
{
    protected $fillable = [
        'user_id', 'category_id', 'title', 'description', 
        'location', 'date_lost', 'images', 'status'
    ];
    
    public function user() {
        return $this->belongsTo(User::class);
    }
    
    public function category() {
        return $this->belongsTo(Category::class);
    }
    
    public function matches() {
        return $this->hasMany(ItemMatch::class);
    }
}

// app/Models/FoundItem.php
class FoundItem extends Model
{
    protected $fillable = [
        'user_id', 'category_id', 'title', 'description', 
        'location', 'date_found', 'images', 'status'
    ];
    
    // Similar relationships as LostItem
}
```

### 1.3 Communication Models
```php
// app/Models/Conversation.php
class Conversation extends Model
{
    protected $fillable = ['user1_id', 'user2_id', 'item_id', 'item_type'];
    
    public function messages() {
        return $this->hasMany(Message::class);
    }
    
    public function participants() {
        return $this->belongsToMany(User::class, 'conversation_participants');
    }
}

// app/Models/Message.php
class Message extends Model
{
    protected $fillable = ['conversation_id', 'user_id', 'content', 'is_read'];
    
    public function conversation() {
        return $this->belongsTo(Conversation::class);
    }
    
    public function sender() {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

### 1.4 Supporting Models
```php
// app/Models/Category.php
class Category extends Model
{
    protected $fillable = ['name', 'icon', 'description'];
    
    public function lostItems() {
        return $this->hasMany(LostItem::class);
    }
    
    public function foundItems() {
        return $this->hasMany(FoundItem::class);
    }
}

// app/Models/Notification.php
class Notification extends Model
{
    protected $fillable = [
        'user_id', 'type', 'title', 'message', 'data', 'is_read'
    ];
    
    public function user() {
        return $this->belongsTo(User::class);
    }
}
```

## 2. CONTROLLER LAYER

### 2.1 Authentication Controllers
```php
// app/Http/Controllers/Auth/AuthController.php
class AuthController extends Controller
{
    public function showRegistrationForm() {
        return view('auth.register');
    }
    
    public function register(RegisterRequest $request) {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);
        
        Auth::login($user);
        return redirect()->route('dashboard');
    }
    
    public function showLoginForm() {
        return view('auth.login');
    }
    
    public function login(LoginRequest $request) {
        if (Auth::attempt($request->only('email', 'password'))) {
            return redirect()->route('dashboard');
        }
        
        return back()->withErrors(['email' => 'Invalid credentials']);
    }
    
    public function logout() {
        Auth::logout();
        return redirect()->route('login');
    }
}
```

### 2.2 Item Management Controllers
```php
// app/Http/Controllers/LostItemController.php
class LostItemController extends Controller
{
    public function index() {
        $lostItems = LostItem::with(['user', 'category'])
                            ->latest()
                            ->paginate(12);
        return view('lost-items.index', compact('lostItems'));
    }
    
    public function create() {
        $categories = Category::all();
        return view('lost-items.create', compact('categories'));
    }
    
    public function store(StoreLostItemRequest $request) {
        $lostItem = LostItem::create([
            'user_id' => auth()->id(),
            'category_id' => $request->category_id,
            'title' => $request->title,
            'description' => $request->description,
            'location' => $request->location,
            'date_lost' => $request->date_lost,
            'images' => $this->handleImageUpload($request),
        ]);
        
        // Trigger matching algorithm
        $this->findPotentialMatches($lostItem);
        
        return redirect()->route('lost-items.show', $lostItem)
                        ->with('success', 'Lost item posted successfully!');
    }
    
    public function show(LostItem $lostItem) {
        $lostItem->load(['user', 'category']);
        return view('lost-items.show', compact('lostItem'));
    }
    
    public function edit(LostItem $lostItem) {
        $this->authorize('update', $lostItem);
        $categories = Category::all();
        return view('lost-items.edit', compact('lostItem', 'categories'));
    }
    
    public function update(UpdateLostItemRequest $request, LostItem $lostItem) {
        $this->authorize('update', $lostItem);
        
        $lostItem->update($request->validated());
        
        return redirect()->route('lost-items.show', $lostItem)
                        ->with('success', 'Lost item updated successfully!');
    }
    
    public function destroy(LostItem $lostItem) {
        $this->authorize('delete', $lostItem);
        $lostItem->delete();
        
        return redirect()->route('lost-items.index')
                        ->with('success', 'Lost item deleted successfully!');
    }
    
    private function findPotentialMatches(LostItem $lostItem) {
        // Algorithm to find matching found items
        $potentialMatches = FoundItem::where('category_id', $lostItem->category_id)
                                   ->where('status', 'available')
                                   ->get();
        
        foreach ($potentialMatches as $foundItem) {
            // Create notification for both users
            Notification::create([
                'user_id' => $lostItem->user_id,
                'type' => 'potential_match',
                'title' => 'Potential Match Found!',
                'message' => "We found a potential match for your lost {$lostItem->title}",
                'data' => json_encode(['found_item_id' => $foundItem->id])
            ]);
        }
    }
}
```

### 2.3 Communication Controllers
```php
// app/Http/Controllers/ChatController.php
class ChatController extends Controller
{
    public function index() {
        $conversations = auth()->user()->conversations()
                                      ->with(['participants', 'messages' => function($q) {
                                          $q->latest()->limit(1);
                                      }])
                                      ->latest()
                                      ->get();
        
        return view('chat.index', compact('conversations'));
    }
    
    public function show(Conversation $conversation) {
        $this->authorize('view', $conversation);
        
        $messages = $conversation->messages()
                                ->with('sender')
                                ->orderBy('created_at')
                                ->get();
        
        // Mark messages as read
        $conversation->messages()
                    ->where('user_id', '!=', auth()->id())
                    ->where('is_read', false)
                    ->update(['is_read' => true]);
        
        return view('chat.show', compact('conversation', 'messages'));
    }
    
    public function store(StoreMessageRequest $request, Conversation $conversation) {
        $this->authorize('participate', $conversation);
        
        $message = Message::create([
            'conversation_id' => $conversation->id,
            'user_id' => auth()->id(),
            'content' => $request->content,
        ]);
        
        // Send real-time notification
        broadcast(new MessageSent($message))->toOthers();
        
        return response()->json($message->load('sender'));
    }
    
    public function initiate(Request $request) {
        $conversation = Conversation::create([
            'user1_id' => auth()->id(),
            'user2_id' => $request->user_id,
            'item_id' => $request->item_id,
            'item_type' => $request->item_type,
        ]);
        
        return redirect()->route('chat.show', $conversation);
    }
}
```

### 2.4 Dashboard and Search Controllers
```php
// app/Http/Controllers/DashboardController.php
class DashboardController extends Controller
{
    public function index() {
        $userLostItems = auth()->user()->lostItems()->latest()->limit(5)->get();
        $userFoundItems = auth()->user()->foundItems()->latest()->limit(5)->get();
        $recentMatches = $this->getRecentMatches();
        $notifications = auth()->user()->notifications()
                                      ->unread()
                                      ->latest()
                                      ->limit(10)
                                      ->get();
        
        return view('dashboard', compact(
            'userLostItems', 'userFoundItems', 'recentMatches', 'notifications'
        ));
    }
    
    private function getRecentMatches() {
        return ItemMatch::whereHas('lostItem', function($q) {
                           $q->where('user_id', auth()->id());
                       })
                       ->orWhereHas('foundItem', function($q) {
                           $q->where('user_id', auth()->id());
                       })
                       ->with(['lostItem', 'foundItem'])
                       ->latest()
                       ->limit(5)
                       ->get();
    }
}

// app/Http/Controllers/SearchController.php
class SearchController extends Controller
{
    public function index(Request $request) {
        $query = $request->get('q');
        $category = $request->get('category');
        $type = $request->get('type', 'all'); // lost, found, all
        
        $lostItems = collect();
        $foundItems = collect();
        
        if (in_array($type, ['lost', 'all'])) {
            $lostItems = LostItem::when($query, function($q) use ($query) {
                               $q->where('title', 'like', "%{$query}%")
                                 ->orWhere('description', 'like', "%{$query}%");
                           })
                           ->when($category, function($q) use ($category) {
                               $q->where('category_id', $category);
                           })
                           ->with(['user', 'category'])
                           ->latest()
                           ->get();
        }
        
        if (in_array($type, ['found', 'all'])) {
            $foundItems = FoundItem::when($query, function($q) use ($query) {
                                $q->where('title', 'like', "%{$query}%")
                                  ->orWhere('description', 'like', "%{$query}%");
                            })
                            ->when($category, function($q) use ($category) {
                                $q->where('category_id', $category);
                            })
                            ->with(['user', 'category'])
                            ->latest()
                            ->get();
        }
        
        $categories = Category::all();
        
        return view('search.results', compact(
            'lostItems', 'foundItems', 'categories', 'query', 'category', 'type'
        ));
    }
}
```

## 3. VIEW LAYER

### 3.1 Layout Structure
```blade
{{-- resources/views/layouts/app.blade.php --}}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@yield('title', 'LoFo - Lost and Found')</title>
    <link href="{{ asset('css/app.css') }}" rel="stylesheet">
    @stack('styles')
</head>
<body>
    @include('partials.navbar')
    
    <main class="container mx-auto px-4 py-8">
        @if(session('success'))
            <div class="alert alert-success mb-4">
                {{ session('success') }}
            </div>
        @endif
        
        @if(session('error'))
            <div class="alert alert-error mb-4">
                {{ session('error') }}
            </div>
        @endif
        
        @yield('content')
    </main>
    
    @include('partials.footer')
    
    <script src="{{ asset('js/app.js') }}"></script>
    @stack('scripts')
</body>
</html>
```

### 3.2 Dashboard View
```blade
{{-- resources/views/dashboard.blade.php --}}
@extends('layouts.app')

@section('title', 'Dashboard - LoFo')

@section('content')
<div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
    <!-- Quick Stats -->
    <div class="lg:col-span-3">
        <div class="grid grid-cols-1 md:grid-cols-4 gap-4 mb-8">
            <div class="bg-white p-6 rounded-lg shadow">
                <h3 class="text-lg font-semibold text-gray-600">My Lost Items</h3>
                <p class="text-3xl font-bold text-red-500">{{ $userLostItems->count() }}</p>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
                <h3 class="text-lg font-semibold text-gray-600">My Found Items</h3>
                <p class="text-3xl font-bold text-green-500">{{ $userFoundItems->count() }}</p>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
                <h3 class="text-lg font-semibold text-gray-600">Potential Matches</h3>
                <p class="text-3xl font-bold text-blue-500">{{ $recentMatches->count() }}</p>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
                <h3 class="text-lg font-semibold text-gray-600">Unread Notifications</h3>
                <p class="text-3xl font-bold text-yellow-500">{{ $notifications->count() }}</p>
            </div>
        </div>
    </div>
    
    <!-- Recent Lost Items -->
    <div class="bg-white p-6 rounded-lg shadow">
        <h2 class="text-xl font-bold mb-4">My Recent Lost Items</h2>
        @forelse($userLostItems as $item)
            <div class="flex items-center space-x-4 mb-4 p-3 border rounded">
                @if($item->images)
                    <img src="{{ Storage::url(json_decode($item->images)[0]) }}" 
                         alt="{{ $item->title }}" 
                         class="w-12 h-12 object-cover rounded">
                @endif
                <div class="flex-1">
                    <h4 class="font-semibold">{{ $item->title }}</h4>
                    <p class="text-sm text-gray-600">{{ Str::limit($item->description, 50) }}</p>
                    <p class="text-xs text-gray-400">{{ $item->created_at->diffForHumans() }}</p>
                </div>
                <a href="{{ route('lost-items.show', $item) }}" 
                   class="text-blue-500 hover:text-blue-700">View</a>
            </div>
        @empty
            <p class="text-gray-500">No lost items yet.</p>
        @endforelse
        
        <a href="{{ route('lost-items.create') }}" 
           class="block w-full text-center bg-red-500 text-white py-2 rounded hover:bg-red-600">
            Report Lost Item
        </a>
    </div>
    
    <!-- Recent Found Items -->
    <div class="bg-white p-6 rounded-lg shadow">
        <h2 class="text-xl font-bold mb-4">My Recent Found Items</h2>
        @forelse($userFoundItems as $item)
            <div class="flex items-center space-x-4 mb-4 p-3 border rounded">
                @if($item->images)
                    <img src="{{ Storage::url(json_decode($item->images)[0]) }}" 
                         alt="{{ $item->title }}" 
                         class="w-12 h-12 object-cover rounded">
                @endif
                <div class="flex-1">
                    <h4 class="font-semibold">{{ $item->title }}</h4>
                    <p class="text-sm text-gray-600">{{ Str::limit($item->description, 50) }}</p>
                    <p class="text-xs text-gray-400">{{ $item->created_at->diffForHumans() }}</p>
                </div>
                <a href="{{ route('found-items.show', $item) }}" 
                   class="text-blue-500 hover:text-blue-700">View</a>
            </div>
        @empty
            <p class="text-gray-500">No found items yet.</p>
        @endforelse
        
        <a href="{{ route('found-items.create') }}" 
           class="block w-full text-center bg-green-500 text-white py-2 rounded hover:bg-green-600">
            Report Found Item
        </a>
    </div>
    
    <!-- Notifications -->
    <div class="bg-white p-6 rounded-lg shadow">
        <h2 class="text-xl font-bold mb-4">Recent Notifications</h2>
        @forelse($notifications as $notification)
            <div class="mb-4 p-3 border-l-4 border-blue-500 bg-blue-50">
                <h4 class="font-semibold">{{ $notification->title }}</h4>
                <p class="text-sm text-gray-600">{{ $notification->message }}</p>
                <p class="text-xs text-gray-400">{{ $notification->created_at->diffForHumans() }}</p>
            </div>
        @empty
            <p class="text-gray-500">No new notifications.</p>
        @endforelse
        
        <a href="{{ route('notifications.index') }}" 
           class="block w-full text-center bg-blue-500 text-white py-2 rounded hover:bg-blue-600">
            View All Notifications
        </a>
    </div>
</div>
@endsection
```

### 3.3 Item Listing Views
```blade
{{-- resources/views/lost-items/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Lost Items - LoFo')

@section('content')
<div class="flex justify-between items-center mb-6">
    <h1 class="text-3xl font-bold">Lost Items</h1>
    <a href="{{ route('lost-items.create') }}" 
       class="bg-red-500 text-white px-6 py-2 rounded hover:bg-red-600">
        Report Lost Item
    </a>
</div>

<!-- Search and Filter -->
<div class="bg-white p-6 rounded-lg shadow mb-6">
    <form action="{{ route('search.index') }}" method="GET" class="flex gap-4">
        <input type="text" name="q" placeholder="Search lost items..." 
               value="{{ request('q') }}" 
               class="flex-1 px-4 py-2 border rounded">
        <select name="category" class="px-4 py-2 border rounded">
            <option value="">All Categories</option>
            @foreach(App\Models\Category::all() as $category)
                <option value="{{ $category->id }}" 
                        {{ request('category') == $category->id ? 'selected' : '' }}>
                    {{ $category->name }}
                </option>
            @endforeach
        </select>
        <input type="hidden" name="type" value="lost">
        <button type="submit" class="bg-blue-500 text-white px-6 py-2 rounded hover:bg-blue-600">
            Search
        </button>
    </form>
</div>

<!-- Items Grid -->
<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
    @forelse($lostItems as $item)
        <div class="bg-white rounded-lg shadow overflow-hidden">
            @if($item->images)
                <img src="{{ Storage::url(json_decode($item->images)[0]) }}" 
                     alt="{{ $item->title }}" 
                     class="w-full h-48 object-cover">
            @else
                <div class="w-full h-48 bg-gray-200 flex items-center justify-center">
                    <span class="text-gray-500">No Image</span>
                </div>
            @endif
            
            <div class="p-4">
                <div class="flex items-center justify-between mb-2">
                    <h3 class="font-semibold text-lg">{{ $item->title }}</h3>
                    <span class="bg-red-100 text-red-800 text-xs px-2 py-1 rounded">
                        {{ $item->category->name }}
                    </span>
                </div>
                
                <p class="text-gray-600 text-sm mb-3">
                    {{ Str::limit($item->description, 100) }}
                </p>
                
                <div class="text-xs text-gray-500 mb-3">
                    <p><i class="fas fa-map-marker-alt"></i> {{ $item->location }}</p>
                    <p><i class="fas fa-calendar"></i> Lost on {{ $item->date_lost->format('M d, Y') }}</p>
                    <p><i class="fas fa-user"></i> by {{ $item->user->name }}</p>
                </div>
                
                <div class="flex justify-between items-center">
                    <span class="text-xs text-gray-400">
                        {{ $item->created_at->diffForHumans() }}
                    </span>
                    <a href="{{ route('lost-items.show', $item) }}" 
                       class="bg-blue-500 text-white px-4 py-2 text-sm rounded hover:bg-blue-600">
                        View Details
                    </a>
                </div>
            </div>
        </div>
    @empty
        <div class="col-span-full text-center py-12">
            <p class="text-gray-500 text-lg">No lost items found.</p>
            <a href="{{ route('lost-items.create') }}" 
               class="inline-block mt-4 bg-red-500 text-white px-6 py-2 rounded hover:bg-red-600">
                Be the first to report a lost item
            </a>
        </div>
    @endforelse
</div>

<!-- Pagination -->
<div class="mt-8">
    {{ $lostItems->links() }}
</div>
@endsection
```

## 4. ROUTING STRUCTURE

```php
// routes/web.php
use App\Http\Controllers\Auth\AuthController;
use App\Http\Controllers\DashboardController;
use App\Http\Controllers\LostItemController;
use App\Http\Controllers\FoundItemController;
use App\Http\Controllers\ChatController;
use App\Http\Controllers\SearchController;
use App\Http\Controllers\NotificationController;

// Authentication Routes
Route::get('/login', [AuthController::class, 'showLoginForm'])->name('login');
Route::post('/login', [AuthController::class, 'login']);
Route::get('/register', [AuthController::class, 'showRegistrationForm'])->name('register');
Route::post('/register', [AuthController::class, 'register']);
Route::post('/logout', [AuthController::class, 'logout'])->name('logout');

// Public Routes
Route::get('/', [DashboardController::class, 'welcome'])->name('welcome');
Route::get('/search', [SearchController::class, 'index'])->name('search.index');
Route::get('/lost-items', [LostItemController::class, 'index'])->name('lost-items.index');
Route::get('/found-items', [FoundItemController::class, 'index'])->name('found-items.index');
Route::get('/lost-items/{lostItem}', [LostItemController::class, 'show'])->name('lost-items.show');
Route::get('/found-items/{foundItem}', [FoundItemController::class, 'show'])->name('found-items.show');

// Protected Routes
Route::middleware('auth')->group(function () {
    // Dashboard
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    
    // Lost Items Management
    Route::get('/lost-items/create', [LostItemController::class, 'create'])->name('lost-items.create');
    Route::post('/lost-items', [LostItemController::class, 'store'])->name('lost-items.store');
    Route::get('/lost-items/{lostItem}/edit', [LostItemController::class, 'edit'])->name('lost-items.edit');
    Route::put('/lost-items/{lostItem}', [LostItemController::class, 'update'])->name('lost-items.update');
    Route::delete('/lost-items/{lostItem}', [LostItemController::class, 'destroy'])->name('lost-items.destroy');
    
    // Found Items Management
    Route::get('/found-items/create', [FoundItemController::class, 'create'])->name('found-items.create');
    Route::post('/found-items', [FoundItemController::class, 'store'])->name('found-items.store');
    Route::get('/found-items/{foundItem}/edit', [FoundItemController::class, 'edit'])->name('found-items.edit');
    Route::put('/found-items/{foundItem}', [FoundItemController::class, 'update'])->name('found-items.update');
    Route::delete('/found-items/{foundItem}', [FoundItemController::class, 'destroy'])->name('found-items.destroy');
    
    // Chat System
    Route::get('/chat', [ChatController::class, 'index'])->name('chat.index');
    Route::get('/chat/{conversation}', [ChatController::class, 'show'])->name('chat.show');
    Route::post('/chat/{conversation}/messages', [ChatController::class, 'store'])->name('chat.messages.store');
    Route::post('/chat/initiate', [ChatController::class, 'initiate'])->name('chat.initiate');
    
    // Notifications
    Route::get('/notifications', [NotificationController::class, 'index'])->name('notifications.index');
    Route::put('/notifications/{notification}/read', [NotificationController::class, 'markAsRead'])->name('notifications.read');
    Route::delete('/notifications/{notification}', [NotificationController::class, 'destroy'])->name('notifications.destroy');
    
    // Profile Management
    Route::get('/profile', [ProfileController::class, 'show'])->name('profile.show');
    Route::get('/profile/edit', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::put('/profile', [ProfileController::class, 'update'])->name('profile.update');
});

// API Routes for AJAX requests
Route::prefix('api')->middleware('auth')->group(function () {
    Route::get('/notifications/unread', [NotificationController::class, 'unread']);
    Route::post('/items/{item}/mark-found', [ItemController::class, 'markAsFound']);
    Route::get('/conversations/{conversation}/messages', [ChatController::class, 'getMessages']);
});
```

## 5. KEUNGGULAN ARSITEKTUR MVC UNTUK LOFO

### 5.1 Separation of Concerns
- **Model**: Fokus pada manajemen data barang hilang/temuan, user, dan komunikasi
- **View**: Menangani tampilan yang responsif dan user-friendly
- **Controller**: Mengatur alur bisnis dan koordinasi antara komponen

### 5.2 Scalability
- Mudah menambah fitur baru (kategorisasi advanced, AI matching, dll)
- Dapat diintegrasikan dengan mobile app melalui API
- Support untuk multiple database dan caching

### 5.3 Maintainability
- Code terorganisir dengan baik
- Mudah untuk debugging dan testing
- Reusable components

### 5.4 Security
- Built-in authentication dan authorization
- CSRF protection
- Input validation dan sanitization
- Secure file upload handling

## 6. IMPLEMENTASI FITUR KHUSUS

### 6.1 Real-time Notifications
```php
// Broadcasting untuk notifikasi real-time
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\PrivateChannel;

class ItemMatchFound implements ShouldBroadcast
{
    public $user;
    public $match;
    
    public function broadcastOn()
    {
        return new PrivateChannel('user.' . $this->user->id);
    }
    
    public function broadcastWith()
    {
        return [
            'type' => 'item_match',
            'title' => 'Potential Match Found!',
            'message' => 'We found a potential match for your item',
            'match_id' => $this->match->id,
        ];
    }
}
```

### 6.2 Advanced Search dengan Elasticsearch
```php
// app/Services/SearchService.php
class SearchService
{
    public function searchItems($query, $filters = [])
    {
        $searchQuery = [
            'index' => 'items',
            'body' => [
                'query' => [
                    'bool' => [
                        'must' => [
                            'multi_match' => [
                                'query' => $query,
                                'fields' => ['title^2', 'description', 'location']
                            ]
                        ],
                        'filter' => $this->buildFilters($filters)
                    ]
                ],
                'sort' => [
                    '_score' => ['order' => 'desc'],
                    'created_at' => ['order' => 'desc']
                ]
            ]
        ];
        
        return $this->elasticsearch->search($searchQuery);
    }
    
    private function buildFilters($filters)
    {
        $filterQueries = [];
        
        if (isset($filters['category'])) {
            $filterQueries[] = ['term' => ['category_id' => $filters['category']]];
        }
        
        if (isset($filters['date_range'])) {
            $filterQueries[] = [
                'range' => [
                    'created_at' => [
                        'gte' => $filters['date_range']['from'],
                        'lte' => $filters['date_range']['to']
                    ]
                ]
            ];
        }
        
        if (isset($filters['location'])) {
            $filterQueries[] = [
                'geo_distance' => [
                    'distance' => $filters['radius'] ?? '10km',
                    'location' => $filters['location']
                ]
            ];
        }
        
        return $filterQueries;
    }
}
```

### 6.3 Image Processing Service
```php
// app/Services/ImageService.php
class ImageService
{
    public function processUpload($images, $type = 'item')
    {
        $processedImages = [];
        
        foreach ($images as $image) {
            // Validate image
            $this->validateImage($image);
            
            // Generate unique filename
            $filename = $this->generateFilename($image);
            
            // Create multiple sizes
            $sizes = $this->createImageSizes($image, $filename);
            
            // Store in cloud storage
            $storedPaths = $this->storeImages($sizes, $type);
            
            $processedImages[] = [
                'original' => $storedPaths['original'],
                'thumbnail' => $storedPaths['thumbnail'],
                'medium' => $storedPaths['medium'],
                'filename' => $filename
            ];
        }
        
        return $processedImages;
    }
    
    private function createImageSizes($image, $filename)
    {
        $img = Image::make($image);
        
        return [
            'original' => $img->encode('jpg', 85),
            'medium' => $img->resize(800, 600, function ($constraint) {
                $constraint->aspectRatio();
                $constraint->upsize();
            })->encode('jpg', 80),
            'thumbnail' => $img->resize(300, 300, function ($constraint) {
                $constraint->aspectRatio();
                $constraint->upsize();
            })->encode('jpg', 75)
        ];
    }
}
```

### 6.4 Matching Algorithm Service
```php
// app/Services/MatchingService.php
class MatchingService
{
    public function findMatches(LostItem $lostItem)
    {
        // Basic matching criteria
        $potentialMatches = FoundItem::where('category_id', $lostItem->category_id)
                                   ->where('status', 'available')
                                   ->get();
        
        $scoredMatches = [];
        
        foreach ($potentialMatches as $foundItem) {
            $score = $this->calculateMatchScore($lostItem, $foundItem);
            
            if ($score > 0.6) { // Threshold for potential match
                $scoredMatches[] = [
                    'found_item' => $foundItem,
                    'score' => $score,
                    'reasons' => $this->getMatchReasons($lostItem, $foundItem)
                ];
            }
        }
        
        // Sort by score
        usort($scoredMatches, function($a, $b) {
            return $b['score'] <=> $a['score'];
        });
        
        return $scoredMatches;
    }
    
    private function calculateMatchScore(LostItem $lost, FoundItem $found)
    {
        $score = 0;
        $weights = [
            'category' => 0.3,
            'description' => 0.25,
            'location' => 0.2,
            'date' => 0.15,
            'color' => 0.1
        ];
        
        // Category match
        if ($lost->category_id === $found->category_id) {
            $score += $weights['category'];
        }
        
        // Description similarity using Levenshtein distance
        $descSimilarity = $this->calculateTextSimilarity(
            $lost->description, 
            $found->description
        );
        $score += $weights['description'] * $descSimilarity;
        
        // Location proximity
        $locationScore = $this->calculateLocationScore(
            $lost->location, 
            $found->location
        );
        $score += $weights['location'] * $locationScore;
        
        // Date proximity
        $dateScore = $this->calculateDateScore(
            $lost->date_lost, 
            $found->date_found
        );
        $score += $weights['date'] * $dateScore;
        
        return $score;
    }
    
    private function calculateTextSimilarity($text1, $text2)
    {
        $text1 = strtolower(trim($text1));
        $text2 = strtolower(trim($text2));
        
        $maxLen = max(strlen($text1), strlen($text2));
        if ($maxLen === 0) return 1;
        
        $distance = levenshtein($text1, $text2);
        return 1 - ($distance / $maxLen);
    }
}
```

## 7. MIDDLEWARE DAN SECURITY

### 7.1 Custom Middleware
```php
// app/Http/Middleware/CheckItemOwnership.php
class CheckItemOwnership
{
    public function handle($request, Closure $next, $itemType)
    {
        $itemId = $request->route($itemType);
        $modelClass = 'App\\Models\\' . Str::studly($itemType);
        
        if (!class_exists($modelClass)) {
            abort(404);
        }
        
        $item = $modelClass::findOrFail($itemId);
        
        if ($item->user_id !== auth()->id()) {
            abort(403, 'You do not have permission to access this item.');
        }
        
        return $next($request);
    }
}

// app/Http/Middleware/TrackActivity.php
class TrackActivity
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        
        if (auth()->check()) {
            ActivityLog::create([
                'user_id' => auth()->id(),
                'action' => $request->method(),
                'url' => $request->fullUrl(),
                'ip_address' => $request->ip(),
                'user_agent' => $request->userAgent(),
                'created_at' => now()
            ]);
        }
        
        return $response;
    }
}
```

### 7.2 Form Requests untuk Validasi
```php
// app/Http/Requests/StoreLostItemRequest.php
class StoreLostItemRequest extends FormRequest
{
    public function authorize()
    {
        return auth()->check();
    }
    
    public function rules()
    {
        return [
            'title' => 'required|string|max:255',
            'description' => 'required|string|max:1000',
            'category_id' => 'required|exists:categories,id',
            'location' => 'required|string|max:255',
            'date_lost' => 'required|date|before_or_equal:today',
            'images.*' => 'image|mimes:jpeg,png,jpg,gif|max:2048',
            'reward' => 'nullable|numeric|min:0',
            'contact_info' => 'nullable|string|max:255'
        ];
    }
    
    public function messages()
    {
        return [
            'title.required' => 'Item title is required.',
            'description.required' => 'Please provide a detailed description.',
            'date_lost.before_or_equal' => 'Date lost cannot be in the future.',
            'images.*.image' => 'Each file must be a valid image.',
            'images.*.max' => 'Each image must not exceed 2MB.'
        ];
    }
}
```

## 8. SERVICE LAYER UNTUK BUSINESS LOGIC

### 8.1 Notification Service
```php
// app/Services/NotificationService.php
class NotificationService
{
    public function sendMatchNotification(User $user, $matchData)
    {
        // Create database notification
        $notification = Notification::create([
            'user_id' => $user->id,
            'type' => 'item_match',
            'title' => 'Potential Match Found!',
            'message' => "We found a potential match for your {$matchData['item_type']}",
            'data' => json_encode($matchData)
        ]);
        
        // Send push notification
        $this->sendPushNotification($user, $notification);
        
        // Send email notification
        $this->sendEmailNotification($user, $notification);
        
        // Broadcast real-time notification
        broadcast(new ItemMatchFound($user, $matchData))->toOthers();
    }
    
    public function sendChatNotification(User $recipient, Message $message)
    {
        $notification = Notification::create([
            'user_id' => $recipient->id,
            'type' => 'new_message',
            'title' => 'New Message',
            'message' => "You have a new message from {$message->sender->name}",
            'data' => json_encode([
                'conversation_id' => $message->conversation_id,
                'sender_id' => $message->user_id
            ])
        ]);
        
        broadcast(new NewMessageNotification($recipient, $message))->toOthers();
    }
}
```

### 8.2 Item Status Service
```php
// app/Services/ItemStatusService.php
class ItemStatusService
{
    public function markAsFound(LostItem $lostItem, FoundItem $foundItem = null)
    {
        DB::transaction(function () use ($lostItem, $foundItem) {
            // Update lost item status
            $lostItem->update([
                'status' => 'found',
                'found_at' => now(),
                'found_item_id' => $foundItem?->id
            ]);
            
            // Update found item if matched
            if ($foundItem) {
                $foundItem->update([
                    'status' => 'returned',
                    'returned_at' => now(),
                    'lost_item_id' => $lostItem->id
                ]);
            }
            
            // Create success notification
            $this->notificationService->sendItemFoundNotification(
                $lostItem->user, 
                $lostItem
            );
            
            // Update statistics
            $this->updateUserStatistics($lostItem->user);
        });
    }
    
    public function markAsReturned(FoundItem $foundItem, LostItem $lostItem)
    {
        DB::transaction(function () use ($foundItem, $lostItem) {
            $foundItem->update([
                'status' => 'returned',
                'returned_at' => now()
            ]);
            
            $lostItem->update([
                'status' => 'found',
                'found_at' => now()
            ]);
            
            // Send thank you notifications
            $this->notificationService->sendThankYouNotification(
                $foundItem->user, 
                $lostItem
            );
            
            // Award reputation points
            $this->reputationService->awardPoints(
                $foundItem->user, 
                'item_returned', 
                10
            );
        });
    }
}
```

## 9. API LAYER UNTUK MOBILE INTEGRATION

### 9.1 API Controllers
```php
// app/Http/Controllers/Api/LostItemApiController.php
class LostItemApiController extends Controller
{
    public function index(Request $request)
    {
        $lostItems = LostItem::with(['user', 'category'])
                            ->when($request->category, function($q) use ($request) {
                                $q->where('category_id', $request->category);
                            })
                            ->when($request->search, function($q) use ($request) {
                                $q->where('title', 'like', "%{$request->search}%")
                                  ->orWhere('description', 'like', "%{$request->search}%");
                            })
                            ->latest()
                            ->paginate(20);
        
        return LostItemResource::collection($lostItems);
    }
    
    public function store(StoreLostItemRequest $request)
    {
        $lostItem = LostItem::create([
            'user_id' => auth()->id(),
            'category_id' => $request->category_id,
            'title' => $request->title,
            'description' => $request->description,
            'location' => $request->location,
            'date_lost' => $request->date_lost,
            'images' => $this->imageService->processUpload($request->file('images')),
        ]);
        
        // Find potential matches
        $matches = $this->matchingService->findMatches($lostItem);
        
        if (count($matches) > 0) {
            $this->notificationService->sendMatchNotification(
                auth()->user(), 
                $matches
            );
        }
        
        return new LostItemResource($lostItem);
    }
}
```

### 9.2 API Resources
```php
// app/Http/Resources/LostItemResource.php
class LostItemResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'location' => $this->location,
            'date_lost' => $this->date_lost->format('Y-m-d'),
            'status' => $this->status,
            'images' => $this->images ? json_decode($this->images) : [],
            'category' => new CategoryResource($this->whenLoaded('category')),
            'user' => new UserResource($this->whenLoaded('user')),
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }
}
```

## 10. TESTING STRATEGY

### 10.1 Feature Tests
```php
// tests/Feature/LostItemTest.php
class LostItemTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_user_can_create_lost_item()
    {
        $user = User::factory()->create();
        $category = Category::factory()->create();
        
        $response = $this->actingAs($user)
                         ->post('/lost-items', [
                             'title' => 'Lost Wallet',
                             'description' => 'Black leather wallet',
                             'category_id' => $category->id,
                             'location' => 'Main Street',
                             'date_lost' => '2024-01-01'
                         ]);
        
        $response->assertRedirect();
        $this->assertDatabaseHas('lost_items', [
            'title' => 'Lost Wallet',
            'user_id' => $user->id
        ]);
    }
    
    public function test_matching_algorithm_works()
    {
        $user1 = User::factory()->create();
        $user2 = User::factory()->create();
        $category = Category::factory()->create();
        
        $lostItem = LostItem::factory()->create([
            'user_id' => $user1->id,
            'category_id' => $category->id,
            'title' => 'iPhone 12',
            'description' => 'Black iPhone 12 with blue case'
        ]);
        
        $foundItem = FoundItem::factory()->create([
            'user_id' => $user2->id,
            'category_id' => $category->id,
            'title' => 'Found Phone',
            'description' => 'iPhone 12 with blue protective case'
        ]);
        
        $matches = app(MatchingService::class)->findMatches($lostItem);
        
        $this->assertNotEmpty($matches);
        $this->assertEquals($foundItem->id, $matches[0]['found_item']->id);
    }
}
```

### 10.2 Unit Tests
```php
// tests/Unit/MatchingServiceTest.php
class MatchingServiceTest extends TestCase
{
    public function test_text_similarity_calculation()
    {
        $service = new MatchingService();
        
        $similarity = $service->calculateTextSimilarity(
            'Black leather wallet',
            'Black wallet made of leather'
        );
        
        $this->assertGreaterThan(0.7, $similarity);
    }
    
    public function test_location_scoring()
    {
        $service = new MatchingService();
        
        $score = $service->calculateLocationScore(
            'Main Street, Downtown',
            'Main St, Downtown Area'
        );
        
        $this->assertGreaterThan(0.5, $score);
    }
}
```

## 11. DEPLOYMENT DAN PRODUCTION CONSIDERATIONS

### 11.1 Environment Configuration
```env
# .env.production
APP_NAME="LoFo - Lost and Found"
APP_ENV=production
APP_KEY=base64:...
APP_DEBUG=false
APP_URL=https://lofo.app

DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=lofo_production
DB_USERNAME=lofo_user
DB_PASSWORD=...

# Caching
CACHE_DRIVER=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

# File Storage
FILESYSTEM_DISK=s3
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=ap-southeast-1
AWS_BUCKET=lofo-production

# Broadcasting
BROADCAST_DRIVER=pusher
PUSHER_APP_ID=...
PUSHER_APP_KEY=...
PUSHER_APP_SECRET=...
PUSHER_APP_CLUSTER=ap1

# Search
ELASTICSEARCH_HOST=localhost:9200

# Mail
MAIL_MAILER=smtp
MAIL_HOST=...
```

### 11.2 Performance Optimizations
```php
// app/Providers/AppServiceProvider.php
public function boot()
{
    // Query optimization
    Model::preventLazyLoading(!app()->isProduction());
    
    // Cache views in production
    if (app()->isProduction()) {
        View::composer('*', function ($view) {
            $view->with('user', auth()->user());
        });
    }
}

// Database indexing
Schema::table('lost_items', function (Blueprint $table) {
    $table->index(['category_id', 'status', 'created_at']);
    $table->index(['location']);
    $table->fullText(['title', 'description']);
});
```

## KESIMPULAN

Arsitektur MVC Laravel untuk aplikasi LoFo menyediakan:

1. **Struktur yang Terorganisir**: Pemisahan yang jelas antara logika bisnis, presentasi, dan kontrol alur
2. **Skalabilitas**: Mudah untuk menambah fitur baru dan mengintegrasikan dengan platform lain
3. **Keamanan**: Built-in security features Laravel untuk proteksi data dan user
4. **Performance**: Optimisasi database, caching, dan real-time features
5. **Maintainability**: Code yang mudah dipelihara dan dikembangkan tim

Implementasi ini mendukung semua kebutuhan fungsional dan non-fungsional yang telah didefinisikan dalam dokumen perancangan aplikasi LoFo, dengan fokus pada user experience yang baik dan sistem yang robust.