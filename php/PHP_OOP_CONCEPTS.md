# PHP OOP Core Concepts — With Real-Life Examples

---

## 1. Class & Object

**Real life**: A class is a **blueprint**. An object is a **real thing built from that blueprint**.

- Class = Architectural plan of a house
- Object = The actual house you live in

You can build many houses (objects) from the same plan (class).

```php
// Blueprint (Class)
class Car {
    public string $brand;
    public string $color;

    public function start() {
        return "$this->brand is starting...";
    }
}

// Real things (Objects)
$my_car = new Car();
$my_car->brand = 'Toyota';
$my_car->color = 'Red';

$your_car = new Car();
$your_car->brand = 'Honda';
$your_car->color = 'Blue';

echo $my_car->start();   // "Toyota is starting..."
echo $your_car->start(); // "Honda is starting..."
```

**Key point**: `Car` is the idea. `$my_car` and `$your_car` are real cars with their own data.

---

## 2. Properties & Methods

**Real life**: Properties are **what something has**. Methods are **what something does**.

- A person **has** a name, age, height → Properties
- A person **can** walk, talk, eat → Methods

```php
class Person {
    // Properties (what a person HAS)
    public string $name;
    public int $age;
    public string $job;

    // Methods (what a person DOES)
    public function introduce(): string {
        return "Hi, I'm $this->name, I'm $this->age years old.";
    }

    public function work(): string {
        return "$this->name is working as a $this->job.";
    }
}

$arnob = new Person();
$arnob->name = 'Arnob';
$arnob->age = 25;
$arnob->job = 'Developer';

echo $arnob->introduce(); // "Hi, I'm Arnob, I'm 25 years old."
echo $arnob->work();      // "Arnob is working as a Developer."
```

---

## 3. Constructor (`__construct`)

**Real life**: When you buy a new phone, it comes with default settings already set (brand, model, storage). You don't manually set each one after purchase — they're set **at the moment of creation**.

The constructor runs **automatically when you create an object**.

```php
class Phone {
    public string $brand;
    public string $model;
    public int $storage;

    // Runs automatically when you create a new Phone
    public function __construct( string $brand, string $model, int $storage ) {
        $this->brand   = $brand;
        $this->model   = $model;
        $this->storage = $storage;
    }

    public function specs(): string {
        return "$this->brand $this->model with {$this->storage}GB";
    }
}

// Constructor sets everything at creation time
$phone = new Phone( 'Apple', 'iPhone 16', 256 );
echo $phone->specs(); // "Apple iPhone 16 with 256GB"
```

**Without constructor**: You'd need to manually set each property after creating the object. Easy to forget one.

---

## 4. Encapsulation (public, private, protected)

**Real life**: Think of a **bank account**.

- `public` — Anyone can see your name → public information
- `private` — Only you know your PIN → private, nobody else can access
- `protected` — Your family members (children) can access your home → protected, accessible by child classes

```php
class BankAccount {
    public string $owner;           // Anyone can see
    private float $balance = 0;     // Only this class can access
    protected string $bank_name;    // This class + child classes can access

    public function __construct( string $owner, string $bank ) {
        $this->owner     = $owner;
        $this->bank_name = $bank;
    }

    // Public method — anyone can call
    public function deposit( float $amount ): void {
        $this->validate_amount( $amount ); // Can call private method internally
        $this->balance += $amount;
    }

    // Public method — controlled access to private data
    public function get_balance(): float {
        return $this->balance;
    }

    // Private method — only this class can use
    private function validate_amount( float $amount ): void {
        if ( $amount <= 0 ) {
            throw new \Exception( 'Amount must be positive.' );
        }
    }
}

$account = new BankAccount( 'Arnob', 'DBBL' );
$account->deposit( 5000 );

echo $account->owner;          // ✅ Works — public
echo $account->get_balance();  // ✅ Works — public method
echo $account->balance;        // ❌ ERROR — private, can't access directly
echo $account->validate_amount( 100 ); // ❌ ERROR — private method
```

**Why encapsulation matters**: You can't directly change the balance. You **must** use `deposit()` which validates the amount first. This prevents invalid data.

---

## 5. Inheritance

**Real life**: A **child inherits traits from a parent**.

- Parent: Animal → has name, can eat, can sleep
- Child: Dog → inherits everything + can bark
- Child: Cat → inherits everything + can meow

The child gets everything the parent has, and can add its own unique features.

```php
// Parent class
class Animal {
    public string $name;

    public function __construct( string $name ) {
        $this->name = $name;
    }

    public function eat(): string {
        return "$this->name is eating.";
    }

    public function sleep(): string {
        return "$this->name is sleeping.";
    }
}

// Child class — inherits eat() and sleep(), adds bark()
class Dog extends Animal {
    public function bark(): string {
        return "$this->name says: Woof!";
    }
}

// Child class — inherits eat() and sleep(), adds meow()
class Cat extends Animal {
    public function meow(): string {
        return "$this->name says: Meow!";
    }
}

$dog = new Dog( 'Rex' );
echo $dog->eat();   // "Rex is eating."    ← inherited from Animal
echo $dog->sleep(); // "Rex is sleeping."  ← inherited from Animal
echo $dog->bark();  // "Rex says: Woof!"   ← Dog's own method

$cat = new Cat( 'Whiskers' );
echo $cat->eat();   // "Whiskers is eating."   ← inherited
echo $cat->meow();  // "Whiskers says: Meow!"  ← Cat's own method
echo $cat->bark();  // ❌ ERROR — Cat can't bark!
```

**Key point**: Dog and Cat share common behavior from Animal. Each adds its own unique behavior.

---

## 6. Method Overriding

**Real life**: Your parent taught you to cook rice a certain way. But you learned a **better way** and now you do it differently. You **override** the parent's method.

```php
class Vehicle {
    public function fuel_type(): string {
        return 'Petrol';
    }

    public function describe(): string {
        return "This vehicle runs on " . $this->fuel_type();
    }
}

class ElectricCar extends Vehicle {
    // Override parent's method with different behavior
    public function fuel_type(): string {
        return 'Electricity';
    }
}

class HybridCar extends Vehicle {
    public function fuel_type(): string {
        return 'Petrol + Electricity';
    }
}

$petrol = new Vehicle();
echo $petrol->describe();    // "This vehicle runs on Petrol"

$electric = new ElectricCar();
echo $electric->describe();  // "This vehicle runs on Electricity"

$hybrid = new HybridCar();
echo $hybrid->describe();    // "This vehicle runs on Petrol + Electricity"
```

**Key point**: Child class replaces parent's behavior while keeping the same method name.

---

## 7. Abstract Class

**Real life**: A **recipe template**.

A recipe template says: "You need a base, a sauce, and a topping" — but doesn't specify **what** those are. Each chef fills in the details differently.

- Abstract class = Template with some rules but incomplete
- You **cannot** cook from a template directly (can't instantiate abstract class)
- Each chef (child class) **must** fill in the blanks (implement abstract methods)

```php
// Template — can't use directly
abstract class Pizza {
    // Concrete method — same for all pizzas
    public function prepare(): string {
        return "Preparing: " . $this->get_base() . " + " . $this->get_sauce() . " + " . $this->get_topping();
    }

    // Abstract methods — each pizza MUST define these
    abstract public function get_base(): string;
    abstract public function get_sauce(): string;
    abstract public function get_topping(): string;
}

class Margherita extends Pizza {
    public function get_base(): string {
        return 'Thin crust';
    }

    public function get_sauce(): string {
        return 'Tomato sauce';
    }

    public function get_topping(): string {
        return 'Mozzarella + Basil';
    }
}

class BBQChicken extends Pizza {
    public function get_base(): string {
        return 'Thick crust';
    }

    public function get_sauce(): string {
        return 'BBQ sauce';
    }

    public function get_topping(): string {
        return 'Chicken + Onion + Cheese';
    }
}

// $pizza = new Pizza();          // ❌ ERROR — can't create abstract class
$margherita = new Margherita();
echo $margherita->prepare();
// "Preparing: Thin crust + Tomato sauce + Mozzarella + Basil"

$bbq = new BBQChicken();
echo $bbq->prepare();
// "Preparing: Thick crust + BBQ sauce + Chicken + Onion + Cheese"
```

**Key point**: Abstract class defines **what must be done** (abstract methods) + **how common things work** (concrete methods). Child classes fill in the specifics.

### Your plugin example:

```php
abstract class Provider {
    // Common for all: caching, model retrieval
    public function generate_embedding( string $text ) {
        $cached = get_transient( $this->get_cache_key( $text ) );
        if ( $cached ) return $cached;
        return $this->make_api_request( $text ); // Each provider implements differently
    }

    // Each provider MUST implement: how to call their specific API
    abstract protected function make_api_request( string $text, string $model_id );
}

class OpenAI extends Provider {
    protected function make_api_request( string $text, string $model_id ) {
        // Call OpenAI API
    }
}

class Gemini extends Provider {
    protected function make_api_request( string $text, string $model_id ) {
        // Call Gemini API
    }
}
```

---

## 8. Interface

**Real life**: A **job requirement**.

A job posting says: "Must be able to code, design, and communicate." It doesn't say **how** — just that you **must** be able to do these things.

- Interface = Job requirement (what you must do)
- Class = The person who fulfills those requirements (how they do it)

```php
// Job requirement — says WHAT, not HOW
interface Payable {
    public function calculate_pay(): float;
    public function get_payment_method(): string;
}

// Full-time employee — fulfills the requirement their way
class FullTimeEmployee implements Payable {
    private float $monthly_salary;

    public function __construct( float $salary ) {
        $this->monthly_salary = $salary;
    }

    public function calculate_pay(): float {
        return $this->monthly_salary;
    }

    public function get_payment_method(): string {
        return 'Bank Transfer';
    }
}

// Freelancer — fulfills the same requirement differently
class Freelancer implements Payable {
    private float $hourly_rate;
    private int $hours_worked;

    public function __construct( float $rate, int $hours ) {
        $this->hourly_rate  = $rate;
        $this->hours_worked = $hours;
    }

    public function calculate_pay(): float {
        return $this->hourly_rate * $this->hours_worked;
    }

    public function get_payment_method(): string {
        return 'PayPal';
    }
}

// Works with ANY Payable — doesn't care if employee or freelancer
function process_payment( Payable $payee ): string {
    $amount = $payee->calculate_pay();
    $method = $payee->get_payment_method();
    return "Paying $$amount via $method";
}

$employee   = new FullTimeEmployee( 5000 );
$freelancer = new Freelancer( 50, 80 );

echo process_payment( $employee );   // "Paying $5000 via Bank Transfer"
echo process_payment( $freelancer ); // "Paying $4000 via PayPal"
```

**Key point**: Interface defines **what** must be done. Classes define **how**. Functions can work with anything that implements the interface.

### Interface vs Abstract Class

| Feature | Interface | Abstract Class |
|---------|-----------|----------------|
| Contains code? | No (only method signatures) | Yes (can have real code) |
| Multiple inheritance? | Yes (class can implement many) | No (can extend only one) |
| Properties? | No (only constants) | Yes |
| Use when | Define a contract only | Share common code + contract |

---

## 9. Polymorphism

**Real life**: The word "play" means different things to different people.

- A musician "plays" → plays an instrument
- A child "plays" → plays a game
- An actor "plays" → plays a role

**Same action name, different behavior** depending on who does it.

```php
abstract class Shape {
    abstract public function area(): float;

    public function describe(): string {
        return get_class( $this ) . " has area: " . $this->area();
    }
}

class Circle extends Shape {
    public function __construct( private float $radius ) {}

    public function area(): float {
        return M_PI * $this->radius ** 2;
    }
}

class Rectangle extends Shape {
    public function __construct( private float $width, private float $height ) {}

    public function area(): float {
        return $this->width * $this->height;
    }
}

class Triangle extends Shape {
    public function __construct( private float $base, private float $height ) {}

    public function area(): float {
        return 0.5 * $this->base * $this->height;
    }
}

// Polymorphism — same method call, different behavior
$shapes = [
    new Circle( 5 ),
    new Rectangle( 4, 6 ),
    new Triangle( 3, 8 ),
];

foreach ( $shapes as $shape ) {
    echo $shape->describe();
    // "Circle has area: 78.54"
    // "Rectangle has area: 24"
    // "Triangle has area: 12"
}
```

**Key point**: `$shape->area()` works on ANY shape. PHP figures out which `area()` to call based on the actual object type. You don't need `if circle then... else if rectangle then...`.

### Your plugin example:

```php
$providers = [ new OpenAI(), new Claude(), new Gemini() ];

foreach ( $providers as $provider ) {
    // Same method call — each provider handles it differently
    $embedding = $provider->generate_embedding( 'product description text' );
}
```

---

## 10. Traits

**Real life**: **Skills** that anyone can learn, regardless of their profession.

A doctor, a developer, and a chef can all learn to drive a car. Driving isn't specific to any profession — it's a reusable skill.

- Traits = Reusable skills that any class can "plug in"
- Solves PHP's lack of multiple inheritance

```php
// Reusable skill — any class can use
trait Loggable {
    public function log( string $message ): void {
        error_log( '[' . get_class( $this ) . '] ' . $message );
    }
}

trait Cacheable {
    public function cache( string $key, $value, int $expiry = 3600 ): void {
        set_transient( $key, $value, $expiry );
    }

    public function get_cache( string $key ) {
        return get_transient( $key );
    }
}

// Order uses both traits
class Order {
    use Loggable, Cacheable;

    public function place( int $product_id ): void {
        $this->log( "Placing order for product $product_id" );
        $this->cache( "order_$product_id", time() );
    }
}

// Notification uses Loggable only
class Notification {
    use Loggable;

    public function send( string $message ): void {
        $this->log( "Sending notification: $message" );
    }
}

$order = new Order();
$order->place( 42 );         // Logs + Caches
$order->log( 'test' );       // ✅ From Loggable trait
$order->cache( 'k', 'v' );   // ✅ From Cacheable trait
```

**Key point**: Traits let you share methods across unrelated classes. A class can use multiple traits (unlike inheritance where you can only extend one class).

---

## 11. Static Methods & Properties

**Real life**: A **factory** or **utility** that doesn't need a specific instance.

You don't need to create a "calculator object" to add 2 + 3. The math works regardless.

```php
class MathHelper {
    public static function add( float $a, float $b ): float {
        return $a + $b;
    }

    public static function percentage( float $amount, float $percent ): float {
        return ( $amount * $percent ) / 100;
    }
}

// No need to create an object — call directly on the class
echo MathHelper::add( 10, 20 );         // 30
echo MathHelper::percentage( 200, 15 ); // 30
```

### Singleton Pattern (used in your plugin):

```php
class AiSemanticSearch {
    private static ?AiSemanticSearch $instance = null;

    // Private constructor — can't create from outside
    private function __construct() {
        // Initialize plugin
    }

    // Static method — only way to get the instance
    public static function init(): AiSemanticSearch {
        if ( self::$instance === null ) {
            self::$instance = new self();
        }
        return self::$instance;
    }
}

// Only ONE instance ever exists
$plugin = AiSemanticSearch::init(); // Creates instance
$same   = AiSemanticSearch::init(); // Returns SAME instance
// $plugin === $same → true
```

**Key point**: Static methods belong to the **class**, not to any specific object.

---

## 12. Namespaces

**Real life**: **Addresses** prevent confusion.

There might be two people named "Arnob" in Bangladesh. But "Arnob at WeLabs, Dhaka" and "Arnob at Google, California" are clearly different people.

Namespaces prevent class name conflicts.

```php
// Without namespaces — CONFLICT!
// class User {} in your plugin
// class User {} in another plugin → ❌ Fatal error: Cannot redeclare

// With namespaces — no conflict
namespace WeLabs\AiSemanticSearch;

class User {
    // Your User class
}

namespace SomeOtherPlugin;

class User {
    // Their User class — no conflict!
}

// Using them:
$my_user    = new \WeLabs\AiSemanticSearch\User();
$their_user = new \SomeOtherPlugin\User();
```

### With `use` statement:

```php
namespace WeLabs\AiSemanticSearch;

use WeLabs\AiSemanticSearch\Services\Provider;
use WeLabs\AiSemanticSearch\Services\Providers\OpenAI;

$provider = new OpenAI(); // No need for full path
```

---

## 13. Magic Methods

**Real life**: **Automatic reactions** — like a doorbell ringing automatically when someone presses the button.

```php
class Product {
    private array $data = [];

    // Called when creating object
    public function __construct( private string $name ) {}

    // Called when accessing non-existent property
    public function __get( string $key ) {
        return $this->data[ $key ] ?? null;
    }

    // Called when setting non-existent property
    public function __set( string $key, $value ): void {
        $this->data[ $key ] = $value;
    }

    // Called when using object as string
    public function __toString(): string {
        return "Product: $this->name";
    }
}

$product = new Product( 'Laptop' );
$product->price = 999;          // Triggers __set
echo $product->price;            // Triggers __get → 999
echo $product;                   // Triggers __toString → "Product: Laptop"
```

### Common magic methods:

| Method | When it runs |
|--------|-------------|
| `__construct()` | Object is created |
| `__destruct()` | Object is destroyed |
| `__get($key)` | Accessing undefined property |
| `__set($key, $value)` | Setting undefined property |
| `__toString()` | Object used as string |
| `__call($method, $args)` | Calling undefined method |
| `__isset($key)` | Using `isset()` on undefined property |

---

## 14. Type Declarations

**Real life**: A form that requires specific input types — you can't enter your name in the phone number field.

```php
class Order {
    public function __construct(
        private int $id,
        private string $customer_name,
        private float $total,
        private bool $is_paid = false
    ) {}

    // Return type declarations
    public function get_total(): float {
        return $this->total;
    }

    // Nullable type — can return string OR null
    public function get_note(): ?string {
        return null;
    }

    // Union type (PHP 8.0+)
    public function get_status(): string|int {
        return $this->is_paid ? 'paid' : 0;
    }
}

$order = new Order( 1, 'Arnob', 99.99 );
$order = new Order( 'abc', 'Arnob', 99.99 ); // ❌ TypeError — id must be int
```

---

## Complete Example: Putting It All Together

A real-world example combining all concepts — an **Embedding Provider System** (like your plugin):

```php
<?php
namespace WeLabs\AiSemanticSearch;

// TRAIT — Reusable logging
trait Loggable {
    protected function log( string $message ): void {
        if ( defined( 'WP_DEBUG' ) && WP_DEBUG ) {
            error_log( '[' . static::class . '] ' . $message );
        }
    }
}

// ABSTRACT CLASS — Base provider with shared logic
abstract class Provider {
    use Loggable; // TRAIT usage

    // ENCAPSULATION — protected property
    protected string $option_prefix = 'ai_semantic_search_';

    // ABSTRACT METHODS — each provider must implement
    abstract public function get_id(): string;
    abstract public function get_title(): string;
    abstract protected function call_api( string $text ): array|false;

    // CONCRETE METHOD — shared logic (caching + validation)
    public function generate_embedding( string $text ): array|false {
        if ( empty( $text ) ) {
            return false;
        }

        // Check cache
        $cache_key = $this->option_prefix . $this->get_id() . '_' . md5( $text );
        $cached    = get_transient( $cache_key );

        if ( $cached !== false ) {
            $this->log( "Cache hit for: " . substr( $text, 0, 50 ) );
            return $cached;
        }

        // POLYMORPHISM — call_api() behaves differently per provider
        $embedding = $this->call_api( $text );

        if ( $embedding ) {
            set_transient( $cache_key, $embedding, MONTH_IN_SECONDS );
            $this->log( "Generated embedding via " . $this->get_title() );
        }

        return $embedding;
    }

    // ENCAPSULATION — controlled access to API key
    public function get_api_key(): string {
        return get_option( $this->option_prefix . $this->get_id() . '_api_key', '' );
    }
}

// INHERITANCE — OpenAI extends Provider
class OpenAI extends Provider {
    // METHOD OVERRIDE — provider-specific values
    public function get_id(): string {
        return 'openai';
    }

    public function get_title(): string {
        return 'OpenAI';
    }

    // IMPLEMENTATION — OpenAI-specific API call
    protected function call_api( string $text ): array|false {
        $response = wp_remote_post( 'https://api.openai.com/v1/embeddings', [
            'headers' => [
                'Authorization' => 'Bearer ' . $this->get_api_key(),
                'Content-Type'  => 'application/json',
            ],
            'body' => wp_json_encode( [
                'input' => $text,
                'model' => 'text-embedding-3-small',
            ] ),
        ] );

        if ( is_wp_error( $response ) ) {
            $this->log( 'API error: ' . $response->get_error_message() );
            return false;
        }

        $body = json_decode( wp_remote_retrieve_body( $response ), true );
        return $body['data'][0]['embedding'] ?? false;
    }
}

// INHERITANCE — Gemini extends Provider
class Gemini extends Provider {
    public function get_id(): string {
        return 'gemini';
    }

    public function get_title(): string {
        return 'Google Gemini';
    }

    protected function call_api( string $text ): array|false {
        // Gemini-specific API call — completely different from OpenAI
        $api_key  = $this->get_api_key();
        $response = wp_remote_post(
            "https://generativelanguage.googleapis.com/v1/models/text-embedding-004:embedContent?key=$api_key",
            [
                'headers' => [ 'Content-Type' => 'application/json' ],
                'body'    => wp_json_encode( [
                    'model'   => 'models/text-embedding-004',
                    'content' => [ 'parts' => [ [ 'text' => $text ] ] ],
                ] ),
            ]
        );

        if ( is_wp_error( $response ) ) {
            $this->log( 'API error: ' . $response->get_error_message() );
            return false;
        }

        $body = json_decode( wp_remote_retrieve_body( $response ), true );
        return $body['embedding']['values'] ?? false;
    }
}

// SINGLETON — Main plugin class
final class AiSemanticSearch {
    // STATIC PROPERTY
    private static ?self $instance = null;

    // ENCAPSULATION — private constructor
    private function __construct() {}

    // STATIC METHOD — singleton access
    public static function init(): self {
        if ( self::$instance === null ) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    // POLYMORPHISM — works with ANY provider
    public function search( string $query, Provider $provider ): array|false {
        return $provider->generate_embedding( $query );
    }
}

// USAGE — Polymorphism in action
$plugin = AiSemanticSearch::init();

$openai = new OpenAI();
$gemini = new Gemini();

// Same method, different providers — POLYMORPHISM
$result1 = $plugin->search( 'red running shoes', $openai );
$result2 = $plugin->search( 'red running shoes', $gemini );
```

---

## Quick Reference Summary

| Concept | What It Is | Real-Life Analogy |
|---------|-----------|-------------------|
| **Class** | Blueprint | House plan |
| **Object** | Instance of class | Actual house |
| **Property** | Data/attribute | Person's name, age |
| **Method** | Action/function | Person can walk, talk |
| **Constructor** | Auto-runs on creation | Phone comes with default settings |
| **Encapsulation** | Access control | Bank PIN is private |
| **Inheritance** | Child gets parent's features | Dog inherits from Animal |
| **Overriding** | Child replaces parent's method | Your own recipe vs parent's |
| **Abstract Class** | Template with blanks to fill | Recipe template |
| **Interface** | Contract/requirement | Job posting requirements |
| **Polymorphism** | Same method, different behavior | "Play" means different things |
| **Trait** | Reusable skill any class can use | Anyone can learn to drive |
| **Static** | Belongs to class, not object | Calculator (no instance needed) |
| **Namespace** | Address to avoid conflicts | "Arnob at WeLabs" vs "Arnob at Google" |
| **Magic Methods** | Auto-triggered methods | Doorbell rings when pressed |

---

## How They Connect in Your Plugin

```
Namespace: WeLabs\AiSemanticSearch
    │
    ├── AiSemanticSearch (Singleton, Static)
    │       └── Uses Provider (Polymorphism)
    │
    ├── Provider (Abstract Class, Trait: Loggable)
    │       ├── Shared logic: caching, validation (Encapsulation)
    │       ├── Abstract: call_api() (must implement)
    │       │
    │       ├── OpenAI extends Provider (Inheritance)
    │       │       └── Implements call_api() (Override)
    │       │
    │       ├── Claude extends Provider (Inheritance)
    │       │       └── Implements call_api() (Override)
    │       │
    │       └── Gemini extends Provider (Inheritance)
    │               └── Implements call_api() (Override)
    │
    └── Model (Abstract Class)
            ├── OpenAITextEmbedding3Small (Inheritance)
            ├── ClaudeEmbeddingV2 (Inheritance)
            └── GeminiTextEmbedding004 (Inheritance)
```

Each concept builds on the previous one. Together they create a **scalable, maintainable, and clean** architecture.
