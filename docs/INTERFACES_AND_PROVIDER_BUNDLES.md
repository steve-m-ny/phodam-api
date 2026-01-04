# Interfaces and Creating Provider Bundles

This guide explains the core interfaces in Phodam and how to create your own provider bundles.

## Table of Contents

1. [Core Interfaces](#core-interfaces)
2. [Creating a Provider](#creating-a-provider)
3. [Creating a Provider Bundle](#creating-a-provider-bundle)
4. [Complete Example](#complete-example)
5. [Advanced Topics](#advanced-topics)

## Core Interfaces

### ProviderInterface

The base interface for all providers. It defines a single method:

```php
interface ProviderInterface
{
    /**
     * @param ProviderContextInterface $context the context in which to create a value
     * @return mixed
     * @throws Throwable
     */
    public function create(ProviderContextInterface $context);
}
```

### TypedProviderInterface

For type-safe providers, use `TypedProviderInterface` which extends `ProviderInterface` with template support:

```php
/**
 * @template T
 */
interface TypedProviderInterface extends ProviderInterface
{
    /**
     * @inheritDoc
     * @return T
     */
    public function create(ProviderContextInterface $context);
}
```

### ProviderContextInterface

The context object passed to providers contains:

- `getType(): string` - The type to be created
- `getOverrides(): array<string, mixed>` - Field overrides
- `hasOverride(string $field): bool` - Check if a field is overridden
- `getOverride(string $field): mixed` - Get override value for a field
- `getConfig(): array<string, mixed>` - Provider-specific configuration
- `getPhodam(): PhodamInterface` - Access to the Phodam instance for creating nested objects

### ProviderBundleInterface

The interface for bundling multiple providers together:

```php
interface ProviderBundleInterface
{
    /**
     * Returns an array of provider class names that should be registered.
     * These classes will be scanned for PhodamProvider/PhodamArrayProvider attributes.
     *
     * @return array<class-string<ProviderInterface>>
     */
    public function getProviders(): array;

    /**
     * Returns an array of type definitions that should be registered.
     *
     * @return array<TypeDefinition>
     */
    public function getTypeDefinitions(): array;
}
```

### PhodamSchemaInterface

The schema interface for registering bundles and providers:

```php
interface PhodamSchemaInterface
{
    /**
     * @param ProviderBundleInterface | class-string<ProviderBundleInterface> $bundleOrClass
     */
    public function registerBundle($bundleOrClass): void;

    /**
     * @param ProviderInterface | class-string<ProviderInterface> $providerOrClass
     */
    public function registerProvider($providerOrClass): void;

    /**
     * @param TypeDefinition $definition
     */
    public function registerTypeDefinition(TypeDefinition $definition): void;

    public function getPhodam(): PhodamInterface;
}
```

## Creating a Provider

### Using Attributes

Providers are registered using PHP attributes. There are two attribute types:

1. **`PhodamProvider`** - For type providers
2. **`PhodamArrayProvider`** - For array providers

### Basic Provider Example

Here's a simple provider that creates instances of a `User` class:

```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use MyLibrary\Model\User;

#[PhodamProvider(type: User::class)]
class UserProvider implements TypedProviderInterface
{
    /**
     * @implements TypedProviderInterface<User>
     */
    public function create(ProviderContextInterface $context): User
    {
        $overrides = $context->getOverrides();
        
        $user = new User();
        $user->setId($overrides['id'] ?? rand(1, 10000));
        $user->setName($overrides['name'] ?? 'John Doe');
        $user->setEmail($overrides['email'] ?? 'john.doe@example.com');
        
        return $user;
    }
}
```

### Named Provider Example

To create a named provider (allowing multiple providers for the same type), specify the `name` parameter:

```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use MyLibrary\Model\Product;

#[PhodamProvider(type: Product::class, name: 'premium')]
class PremiumProductProvider implements TypedProviderInterface
{
    /**
     * @implements TypedProviderInterface<Product>
     */
    public function create(ProviderContextInterface $context): Product
    {
        $overrides = $context->getOverrides();
        
        $product = new Product();
        $product->setName($overrides['name'] ?? 'Premium Product');
        $product->setPrice($overrides['price'] ?? rand(10000, 100000) / 100);
        $product->setPremium(true);
        
        return $product;
    }
}
```

### Provider with Nested Objects

Providers can use the Phodam instance to create nested objects:

```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use MyLibrary\Model\Order;
use MyLibrary\Model\User;

#[PhodamProvider(type: Order::class)]
class OrderProvider implements TypedProviderInterface
{
    /**
     * @implements TypedProviderInterface<Order>
     */
    public function create(ProviderContextInterface $context): Order
    {
        $overrides = $context->getOverrides();
        $phodam = $context->getPhodam();
        
        $order = new Order();
        $order->setId($overrides['id'] ?? rand(1, 10000));
        $order->setUser($overrides['user'] ?? $phodam->create(User::class));
        $order->setTotal($overrides['total'] ?? rand(1000, 100000) / 100);
        
        return $order;
    }
}
```

### Provider with Configuration

Providers can use the config array for provider-specific settings:

```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use MyLibrary\Model\Product;

#[PhodamProvider(type: Product::class)]
class ProductProvider implements TypedProviderInterface
{
    /**
     * @implements TypedProviderInterface<Product>
     */
    public function create(ProviderContextInterface $context): Product
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        // Use config to control behavior
        $minPrice = $config['minPrice'] ?? 1.00;
        $maxPrice = $config['maxPrice'] ?? 1000.00;
        
        $product = new Product();
        $product->setName($overrides['name'] ?? 'Product ' . rand(1, 1000));
        $product->setPrice($overrides['price'] ?? rand((int)($minPrice * 100), (int)($maxPrice * 100)) / 100);
        
        return $product;
    }
}
```

### Array Provider Example

Array providers must be named and use the `PhodamArrayProvider` attribute:

```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamArrayProvider;
use Phodam\Provider\ProviderInterface;
use Phodam\Provider\ProviderContextInterface;

#[PhodamArrayProvider(name: 'shoppingCart')]
class ShoppingCartProvider implements ProviderInterface
{
    public function create(ProviderContextInterface $context): array
    {
        $overrides = $context->getOverrides();
        $phodam = $context->getPhodam();
        
        return [
            'id' => $overrides['id'] ?? rand(1, 10000),
            'items' => $overrides['items'] ?? [],
            'total' => $overrides['total'] ?? 0.0,
            'created_at' => $overrides['created_at'] ?? date('Y-m-d H:i:s'),
        ];
    }
}
```

## Creating a Provider Bundle

A `ProviderBundle` registers multiple providers at once. Here's how to create one:

```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\ProviderBundleInterface;
use Phodam\Types\TypeDefinition;

class MyLibraryProviderBundle implements ProviderBundleInterface
{
    public function getProviders(): array
    {
        return [
            UserProvider::class,
            OrderProvider::class,
            ProductProvider::class,
            PremiumProductProvider::class,
            ShoppingCartProvider::class,
        ];
    }

    public function getTypeDefinitions(): array
    {
        // Return any type definitions you want to register
        return [];
    }
}
```

The bundle returns an array of provider class names. Phodam will automatically scan these classes for `PhodamProvider` or `PhodamArrayProvider` attributes and register them accordingly.

### Override Providers

You can mark providers as override providers using the `overriding` parameter in the attribute:

```php
#[PhodamProvider(type: User::class, overriding: true)]
class CustomUserProvider implements TypedProviderInterface
{
    // This provider will override any existing User provider
}
```

## Complete Example

Here's a complete example of a provider library:

### Directory Structure

```
my-library/
├── src/
│   ├── Model/
│   │   ├── User.php
│   │   └── Order.php
│   └── Provider/
│       ├── UserProvider.php
│       ├── OrderProvider.php
│       └── MyLibraryProviderBundle.php
├── composer.json
└── README.md
```

### Model Classes

**src/Model/User.php:**
```php
<?php

declare(strict_types=1);

namespace MyLibrary\Model;

class User
{
    private int $id;
    private string $name;
    private string $email;
    
    // Getters and setters...
    public function setId(int $id): void { $this->id = $id; }
    public function setName(string $name): void { $this->name = $name; }
    public function setEmail(string $email): void { $this->email = $email; }
    public function getId(): int { return $this->id; }
    public function getName(): string { return $this->name; }
    public function getEmail(): string { return $this->email; }
}
```

**src/Model/Order.php:**
```php
<?php

declare(strict_types=1);

namespace MyLibrary\Model;

class Order
{
    private int $id;
    private User $user;
    private float $total;
    
    // Getters and setters...
    public function setId(int $id): void { $this->id = $id; }
    public function setUser(User $user): void { $this->user = $user; }
    public function setTotal(float $total): void { $this->total = $total; }
    public function getId(): int { return $this->id; }
    public function getUser(): User { return $this->user; }
    public function getTotal(): float { return $this->total; }
}
```

### Providers

**src/Provider/UserProvider.php:**
```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use MyLibrary\Model\User;

#[PhodamProvider(type: User::class)]
class UserProvider implements TypedProviderInterface
{
    /**
     * @implements TypedProviderInterface<User>
     */
    public function create(ProviderContextInterface $context): User
    {
        $overrides = $context->getOverrides();
        
        $user = new User();
        $user->setId($overrides['id'] ?? rand(1, 10000));
        $user->setName($overrides['name'] ?? 'John Doe');
        $user->setEmail($overrides['email'] ?? 'john.doe@example.com');
        
        return $user;
    }
}
```

**src/Provider/OrderProvider.php:**
```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use MyLibrary\Model\Order;
use MyLibrary\Model\User;

#[PhodamProvider(type: Order::class)]
class OrderProvider implements TypedProviderInterface
{
    /**
     * @implements TypedProviderInterface<Order>
     */
    public function create(ProviderContextInterface $context): Order
    {
        $overrides = $context->getOverrides();
        $phodam = $context->getPhodam();
        
        $order = new Order();
        $order->setId($overrides['id'] ?? rand(1, 10000));
        $order->setUser($overrides['user'] ?? $phodam->create(User::class));
        $order->setTotal($overrides['total'] ?? rand(1000, 100000) / 100);
        
        return $order;
    }
}
```

### Provider Bundle

**src/Provider/MyLibraryProviderBundle.php:**
```php
<?php

declare(strict_types=1);

namespace MyLibrary\Provider;

use Phodam\Provider\ProviderBundleInterface;
use Phodam\Types\TypeDefinition;

class MyLibraryProviderBundle implements ProviderBundleInterface
{
    public function getProviders(): array
    {
        return [
            UserProvider::class,
            OrderProvider::class,
        ];
    }

    public function getTypeDefinitions(): array
    {
        return [];
    }
}
```

### Composer Configuration

**composer.json:**
```json
{
    "name": "my-library/phodam-providers",
    "description": "Phodam providers for MyLibrary models",
    "type": "library",
    "require": {
        "php": "^8.4",
        "phodam/phodam-api": "^2.0"
    },
    "autoload": {
        "psr-4": {
            "MyLibrary\\": "src/"
        }
    }
}
```

### Usage

Once your library is installed, users can register it like this:

```php
<?php

use Phodam\Phodam;
use Phodam\PhodamSchema;
use MyLibrary\Provider\MyLibraryProviderBundle;

$schema = PhodamSchema::withDefaults();
$phodam = $schema->getPhodam();

// Register your bundle
$schema->registerBundle(MyLibraryProviderBundle::class);
// Or with an instance:
// $schema->registerBundle(new MyLibraryProviderBundle());

// Now users can create instances
$user = $phodam->create(User::class);
$order = $phodam->create(Order::class);

// With overrides
$customUser = $phodam->create(User::class, null, ['name' => 'Jane Doe']);

// With named providers (if you registered them)
$premiumProduct = $phodam->create(Product::class, 'premium');
```

## Advanced Topics

### Handling Overrides

Always check for overrides in your providers:

```php
public function create(ProviderContextInterface $context): MyType
{
    $overrides = $context->getOverrides();
    
    // Check if a specific field is overridden
    if ($context->hasOverride('fieldName')) {
        $value = $context->getOverride('fieldName');
    } else {
        $value = $this->generateDefaultValue();
    }
}
```

### Using Configuration

Providers can accept configuration through the context:

```php
public function create(ProviderContextInterface $context): MyType
{
    $config = $context->getConfig();
    $minValue = $config['min'] ?? 0;
    $maxValue = $config['max'] ?? 100;
    
    // Use config values...
}
```

Usage:
```php
$phodam->create(MyType::class, null, null, ['min' => 10, 'max' => 50]);
```

### Error Handling

Providers can throw exceptions if they cannot create a value:

```php
use Phodam\Provider\CreationFailedException;

public function create(ProviderContextInterface $context): MyType
{
    if ($someCondition) {
        throw new CreationFailedException('Unable to create MyType: reason');
    }
    
    // Create and return...
}
```

### Type Definitions

You can register type definitions alongside providers:

```php
use Phodam\Types\TypeDefinition;
use Phodam\Types\FieldDefinition;

class MyProviderBundle implements ProviderBundleInterface
{
    public function getProviders(): array
    {
        return [];
    }

    public function getTypeDefinitions(): array
    {
        // string $type, ?string $name = null, bool $overriding = false, array $fields = []
        $definition = new TypeDefinition(
            Article::class,
            null,
            false,
            [
                'id' => new FieldDefinition('int'),
                'name' => new FieldDefinition('string')
                    ->setNullable(true),
                'tags' => new FieldDefinition('string')
                    ->setArray(true),
            ]
        );
        
        return [$definition];
    }
}
```

## Best Practices

1. **Use TypedProviderInterface** - Provides better type safety and IDE support
2. **Handle Overrides** - Always check for and respect override values
3. **Use Meaningful Defaults** - Generate realistic test data
4. **Document Your Providers** - Add PHPDoc comments explaining what each provider creates
5. **Keep Providers Focused** - Each provider should handle one type
6. **Use Named Providers** - When you need multiple variations of the same type
7. **Leverage Phodam** - Use `$context->getPhodam()` to create nested objects
8. **Test Your Providers** - Write unit tests for your providers

## Summary

Creating a provider library involves:

1. Creating provider classes that implement `ProviderInterface` or `TypedProviderInterface`
2. Adding `PhodamProvider` or `PhodamArrayProvider` attributes to your provider classes
3. Implementing the `create()` method that generates test data
4. Creating a bundle class that implements `ProviderBundleInterface`
5. Returning all provider class names in the bundle's `getProviders()` method
6. Distributing your library via Composer

Users can then easily register your entire library with a single call to `$schema->registerBundle(YourBundle::class)`.
