
# Laravel Setup on Poridhi VS CODE

This guide will walk you through setting up PHP 8.1 and Laravel on Ubuntu 22.04, covering all necessary steps and dependencies.

---

## Prerequisites

---

### 1. Update the System

Start by updating and upgrading your system to ensure all packages are up-to-date:

```bash
sudo apt update && sudo apt upgrade -y

```
---

### 2. Install PHP and Required Extensions

Add the PHP repository and install PHP 8.1 along with the necessary extensions:

```bash
# Install prerequisites for adding a new repository
sudo apt install -y software-properties-common

# Add the PHP repository
sudo add-apt-repository ppa:ondrej/php
sudo apt update

# Install PHP 8.1 and required extensions
sudo apt install -y php8.1 php8.1-common php8.1-cli php8.1-curl php8.1-mbstring php8.1-xml php8.1-zip php8.1-mysql php8.1-gd unzip
```

---

### 3. Install Composer

Composer is the dependency manager for PHP, which we will use to install Laravel.

```bash
# Download Composer installer
curl -sS https://getcomposer.org/installer | php

# Move Composer to a global location
sudo mv composer.phar /usr/local/bin/composer

# Set executable permissions
sudo chmod +x /usr/local/bin/composer
```

---

### 4. Install Node.js and npm

Node.js and npm are required to manage frontend dependencies in Laravel.

```bash
# Add Node.js repository and install Node.js
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
```

---

## Laravel Setup

---

### 1. Install Laravel via Composer

Use Composer to install the Laravel installer globally:

```bash
composer global require laravel/installer

# Add Composer's global bin to the PATH (for future sessions, add this to ~/.bashrc)
echo 'export PATH="$PATH:$HOME/.config/composer/vendor/bin"' >> ~/.bashrc
source ~/.bashrc
```

---

### 2. Create a New Laravel Project

Create a new Laravel project. Replace `project-name` with your desired project name.

```bash
# Using the Laravel installer
laravel new project-name

# OR using Composer directly
composer create-project laravel/laravel project-name
```

---

### 3. Set Proper Permissions

To ensure Laravel can write to the necessary directories, adjust permissions:

```bash
cd project-name
sudo chown -R $USER:www-data storage
sudo chown -R $USER:www-data bootstrap/cache
chmod -R 775 storage
chmod -R 775 bootstrap/cache
```

---

### 4. Setup Environment Configuration

Copy the example environment file and generate an application key:

```bash
cp .env.example .env
php artisan key:generate
```

---

### 5. Install Node.js Dependencies and Compile Assets

Laravel uses npm to manage frontend dependencies. Install these dependencies and compile the assets:

```bash
npm install
npm run dev
```

---

### 6. Start the Development Server

You’re ready to run your Laravel application! Start the development server with:

```bash
php artisan serve
```

Visit [http://localhost:8000](http://localhost:8000) in your browser to see your Laravel application running.

---

## Laravel ToDo App

This guide will walk you through creating a simple ToDo app using Laravel with basic CRUD functionality.

---

### Prerequisites

Ensure you've set up Laravel by following the previous setup instructions. Configure your database in the `.env` file:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=your_database_name
DB_USERNAME=your_username
DB_PASSWORD=your_password
```

---

### 1. Create the `todos` Table Migration

Generate a migration for the `todos` table:

```bash
php artisan make:migration create_todos_table
```

Define the schema in the migration file at `database/migrations/[timestamp]_create_todos_table.php`:

```php
public function up()
{
    Schema::create('todos', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->text('description')->nullable();
        $table->boolean('completed')->default(false);
        $table->timestamps();
    });
}
```

---

### 2. Create the Todo Model

Generate a model for `Todo`:

```bash
php artisan make:model Todo
```

Define the model’s properties in `app/Models/Todo.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Todo extends Model
{
    protected $fillable = ['title', 'description', 'completed'];

    protected $casts = [
        'completed' => 'boolean'
    ];
}
```

---

### 3. Create the TodoController

Generate a controller with resource methods:

```bash
php artisan make:controller TodoController --resource
```

Implement CRUD operations in `app/Http/Controllers/TodoController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Todo;
use Illuminate\Http\Request;

class TodoController extends Controller
{
    public function index()
    {
        $todos = Todo::latest()->get();
        return view('todos.index', compact('todos'));
    }

    public function create()
    {
        return view('todos.create');
    }

    public function store(Request $request)
    {
        $request->validate([
            'title' => 'required|max:255',
            'description' => 'nullable'
        ]);

        Todo::create($request->all());
        return redirect()->route('todos.index')->with('success', 'Todo created successfully');
    }

    public function edit(Todo $todo)
    {
        return view('todos.edit', compact('todo'));
    }

    public function update(Request $request, Todo $todo)
    {
        $request->validate([
            'title' => 'required|max:255',
            'description' => 'nullable'
        ]);

        $todo->update($request->all());
        return redirect()->route('todos.index')->with('success', 'Todo updated successfully');
    }

    public function destroy(Todo $todo)
    {
        $todo->delete();
        return redirect()->route('todos.index')->with('success', 'Todo deleted successfully');
    }

    public function toggleComplete(Todo $todo)
    {
        $todo->completed = !$todo->completed;
        $todo->save();
        return redirect()->route('todos.index');
    }
}
```

---

### 4. Define Routes

Add routes in `routes/web.php`:

```php
<?php

use App\Http\Controllers\TodoController;

Route::resource('todos', TodoController::class);
Route::patch('todos/{todo}/toggle', [TodoController::class, 'toggleComplete'])->name('todos.toggle');
```

---

### 5. Create the Views

#### `resources/views/todos/index.blade.php`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Todo App</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100">
    <div class="container mx-auto px-4 py-8">
        <div class="mb-4 flex justify-between items-center">
            <h1 class="text-2xl font-bold">Todo List</h1>
            <a href="{{ route('todos.create') }}" class="bg-blue-500 text-white px-4 py-2 rounded">Add New Todo</a>
        </div>

        @if(session('success'))
            <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded mb-4">
                {{ session('success') }}
            </div>
        @endif

        <div class="bg-white rounded-lg shadow">
            @foreach($todos as $todo)
                <div class="border-b p-4 flex items-center justify-between">
                    <div class="flex items-center space-x-4">
                        <form action="{{ route('todos.toggle', $todo) }}" method="POST">
                            @csrf
                            @method('PATCH')
                            <input type="checkbox" onchange="this.form.submit()" {{ $todo->completed ? 'checked' : '' }} class="h-4 w-4">
                        </form>
                        <div>
                            <h3 class="font-medium {{ $todo->completed ? 'line-through text-gray-400' : '' }}">{{ $todo->title }}</h3>
                            <p class="text-gray-500">{{ $todo->description }}</p>
                        </div>
                    </div>
                    <div class="flex space-x-2">
                        <a href="{{ route('todos.edit', $todo) }}" class="text-blue-500 hover:text-blue-700">Edit</a>
                        <form action="{{ route('todos.destroy', $todo) }}" method="POST" onsubmit="return confirm('Are you sure?')">
                            @csrf
                            @method('DELETE')
                            <button class="text-red-500 hover:text-red-700">Delete</button>
                        </form>
                    </div>
                </div>
            @endforeach
        </div>
    </div>
</body>
</html>
```

---

### 6. Run Migrations

Run the

 migrations to create the database tables:

```bash
php artisan migrate
```

---

### 7. Test Your Application

Visit [http://localhost:8000/todos](http://localhost:8000/todos) to manage your Todo list.

---

That's it! You now have a functional Laravel ToDo application with CRUD operations.



