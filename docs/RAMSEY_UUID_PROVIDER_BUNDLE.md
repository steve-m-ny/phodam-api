# Creating a Provider Bundle for ramsey/uuid

This guide demonstrates how to create a ProviderBundle for the `ramsey/uuid` library that provides providers for `UuidInterface` and named providers for different UUID versions.

## Overview

The `ramsey/uuid` library provides support for generating UUIDs of various versions:
- **UUID v1**: Time-based UUID
- **UUID v2**: DCE Security UUID (rarely used)
- **UUID v3**: Name-based UUID using MD5 hashing
- **UUID v4**: Random UUID (most common)
- **UUID v5**: Name-based UUID using SHA-1 hashing
- **UUID v6**: Reordered time-based UUID
- **UUID v7**: Unix Epoch time-based UUID

This guide will show you how to create providers for `UuidInterface` and named providers for each version.

## Prerequisites

- PHP 8.4+
- `phodam/phodam-api` package
- `ramsey/uuid` package (^4.0 or ^5.0)

## Table of Contents

1. [Project Setup](#project-setup)
2. [Creating Individual Providers](#creating-individual-providers)
3. [Creating the Provider Bundle](#creating-the-provider-bundle)
4. [Complete Example](#complete-example)
5. [Usage Examples](#usage-examples)
6. [Advanced Configuration](#advanced-configuration)

## Project Setup

### Composer Configuration

First, set up your `composer.json`:

```json
{
    "name": "your-vendor/phodam-ramsey-uuid",
    "description": "Phodam providers for ramsey/uuid",
    "type": "library",
    "require": {
        "php": "^8.4",
        "phodam/phodam-api": "^2.0",
        "ramsey/uuid": "^4.0|^5.0"
    },
    "autoload": {
        "psr-4": {
            "YourVendor\\PhodamRamseyUuid\\": "src/"
        }
    }
}
```

### Directory Structure

```
phodam-ramsey-uuid/
├── src/
│   └── Provider/
│       ├── UuidInterfaceProvider.php
│       ├── UuidV1Provider.php
│       ├── UuidV3Provider.php
│       ├── UuidV4Provider.php
│       ├── UuidV5Provider.php
│       ├── UuidV6Provider.php
│       ├── UuidV7Provider.php
│       └── RamseyUuidProviderBundle.php
├── composer.json
└── README.md
```

## Creating Individual Providers

### Base UuidInterface Provider

Create a default provider for `UuidInterface` that generates UUID v4 (the most common random UUID):

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Default provider for UuidInterface.
 * Generates UUID v4 (random) by default.
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class)]
class UuidInterfaceProvider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        
        // Allow override of the entire UUID
        if (isset($overrides['uuid'])) {
            if ($overrides['uuid'] instanceof UuidInterface) {
                return $overrides['uuid'];
            }
            if (is_string($overrides['uuid'])) {
                return Uuid::fromString($overrides['uuid']);
            }
        }
        
        // Default to UUID v4 (random)
        return Uuid::uuid4();
    }
}
```

### UUID v1 Provider (Time-based)

UUID v1 is based on the current time and MAC address:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Provider for UUID v1 (time-based).
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class, name: 'v1')]
class UuidV1Provider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        // UUID v1 can optionally take a node and clock sequence
        $node = $overrides['node'] ?? $config['node'] ?? null;
        $clockSeq = $overrides['clockSeq'] ?? $config['clockSeq'] ?? null;
        
        if ($node !== null && $clockSeq !== null) {
            return Uuid::uuid1($node, $clockSeq);
        } elseif ($node !== null) {
            return Uuid::uuid1($node);
        }
        
        return Uuid::uuid1();
    }
}
```

### UUID v3 Provider (Name-based with MD5)

UUID v3 requires a namespace UUID and a name:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Provider for UUID v3 (name-based with MD5).
 * Requires a namespace UUID and a name.
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class, name: 'v3')]
class UuidV3Provider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        // Get namespace and name from overrides or config
        $namespace = $overrides['namespace'] ?? $config['namespace'] ?? null;
        $name = $overrides['name'] ?? $config['name'] ?? 'default-name';
        
        if ($namespace === null) {
            // Default to DNS namespace if not provided
            $namespace = Uuid::NAMESPACE_DNS;
        }
        
        // Convert namespace to UuidInterface if it's a string
        if (is_string($namespace)) {
            $namespace = Uuid::fromString($namespace);
        } elseif (!$namespace instanceof UuidInterface) {
            throw new \InvalidArgumentException(
                'Namespace must be a UuidInterface instance or a valid UUID string'
            );
        }
        
        return Uuid::uuid3($namespace, $name);
    }
}
```

### UUID v4 Provider (Random)

UUID v4 is the most common random UUID:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Provider for UUID v4 (random).
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class, name: 'v4')]
class UuidV4Provider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        
        // Allow override of the entire UUID
        if (isset($overrides['uuid'])) {
            if ($overrides['uuid'] instanceof UuidInterface) {
                return $overrides['uuid'];
            }
            if (is_string($overrides['uuid'])) {
                return Uuid::fromString($overrides['uuid']);
            }
        }
        
        return Uuid::uuid4();
    }
}
```

### UUID v5 Provider (Name-based with SHA-1)

UUID v5 is similar to v3 but uses SHA-1 instead of MD5:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Provider for UUID v5 (name-based with SHA-1).
 * Requires a namespace UUID and a name.
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class, name: 'v5')]
class UuidV5Provider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        // Get namespace and name from overrides or config
        $namespace = $overrides['namespace'] ?? $config['namespace'] ?? null;
        $name = $overrides['name'] ?? $config['name'] ?? 'default-name';
        
        if ($namespace === null) {
            // Default to DNS namespace if not provided
            $namespace = Uuid::NAMESPACE_DNS;
        }
        
        // Convert namespace to UuidInterface if it's a string
        if (is_string($namespace)) {
            $namespace = Uuid::fromString($namespace);
        } elseif (!$namespace instanceof UuidInterface) {
            throw new \InvalidArgumentException(
                'Namespace must be a UuidInterface instance or a valid UUID string'
            );
        }
        
        return Uuid::uuid5($namespace, $name);
    }
}
```

### UUID v6 Provider (Reordered Time-based)

UUID v6 is a reordered version of UUID v1:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Provider for UUID v6 (reordered time-based).
 * Note: UUID v6 support may require ramsey/uuid ^5.0
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class, name: 'v6')]
class UuidV6Provider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        // UUID v6 can optionally take a node and clock sequence
        $node = $overrides['node'] ?? $config['node'] ?? null;
        $clockSeq = $overrides['clockSeq'] ?? $config['clockSeq'] ?? null;
        
        if ($node !== null && $clockSeq !== null) {
            return Uuid::uuid6($node, $clockSeq);
        } elseif ($node !== null) {
            return Uuid::uuid6($node);
        }
        
        return Uuid::uuid6();
    }
}
```

### UUID v7 Provider (Unix Epoch Time-based)

UUID v7 is based on Unix epoch time:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\PhodamProvider;
use Phodam\Provider\TypedProviderInterface;
use Phodam\Provider\ProviderContextInterface;
use Ramsey\Uuid\Uuid;
use Ramsey\Uuid\UuidInterface;

/**
 * Provider for UUID v7 (Unix epoch time-based).
 * Note: UUID v7 support may require ramsey/uuid ^5.0
 *
 * @implements TypedProviderInterface<UuidInterface>
 */
#[PhodamProvider(type: UuidInterface::class, name: 'v7')]
class UuidV7Provider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        // UUID v7 doesn't take any parameters
        return Uuid::uuid7();
    }
}
```

## Creating the Provider Bundle

Now create the bundle that registers all providers:

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\ProviderBundleInterface;
use Phodam\Types\TypeDefinition;

class RamseyUuidProviderBundle implements ProviderBundleInterface
{
    public function getProviders(): array
    {
        return [
            UuidInterfaceProvider::class,  // Default provider (v4)
            UuidV1Provider::class,         // Named provider 'v1'
            UuidV3Provider::class,         // Named provider 'v3'
            UuidV4Provider::class,         // Named provider 'v4'
            UuidV5Provider::class,         // Named provider 'v5'
            UuidV6Provider::class,         // Named provider 'v6'
            UuidV7Provider::class,         // Named provider 'v7'
        ];
    }

    public function getTypeDefinitions(): array
    {
        return [];
    }
}
```

The bundle returns an array of provider class names. Phodam will automatically scan these classes for `PhodamProvider` attributes and register them based on the type and name specified in each attribute.

## Complete Example

Here's a complete, production-ready example with all files:

### src/Provider/RamseyUuidProviderBundle.php

```php
<?php

declare(strict_types=1);

namespace YourVendor\PhodamRamseyUuid\Provider;

use Phodam\Provider\ProviderBundleInterface;
use Phodam\Types\TypeDefinition;

class RamseyUuidProviderBundle implements ProviderBundleInterface
{
    public function getProviders(): array
    {
        return [
            UuidInterfaceProvider::class,
            UuidV1Provider::class,
            UuidV3Provider::class,
            UuidV4Provider::class,
            UuidV5Provider::class,
            UuidV6Provider::class,
            UuidV7Provider::class,
        ];
    }

    public function getTypeDefinitions(): array
    {
        return [];
    }
}
```

## Usage Examples

### Basic Usage

```php
<?php

use Phodam\Phodam;
use Phodam\PhodamSchema;
use YourVendor\PhodamRamseyUuid\Provider\RamseyUuidProviderBundle;
use Ramsey\Uuid\UuidInterface;

// Create Phodam instance and schema
$schema = PhodamSchema::withDefaults();
$phodam = $schema->getPhodam();

// Register the bundle
$schema->registerBundle(RamseyUuidProviderBundle::class);

// Use the default provider (generates v4)
$uuid = $phodam->create(UuidInterface::class);
echo $uuid->toString(); // e.g., "550e8400-e29b-41d4-a716-446655440000"

// Use named providers for specific versions
$uuidV1 = $phodam->create(UuidInterface::class, 'v1');
$uuidV4 = $phodam->create(UuidInterface::class, 'v4');
$uuidV5 = $phodam->create(UuidInterface::class, 'v5');
```

### Using Overrides

```php
// Override with a specific UUID string
$customUuid = $phodam->create(
    UuidInterface::class,
    null,
    ['uuid' => '123e4567-e89b-12d3-a456-426614174000']
);

// Override with a UuidInterface instance
$existingUuid = Uuid::uuid4();
$sameUuid = $phodam->create(
    UuidInterface::class,
    null,
    ['uuid' => $existingUuid]
);
```

### Using Configuration for v3 and v5

UUID v3 and v5 require a namespace and name:

```php
use Ramsey\Uuid\Uuid;

// Using overrides
$uuidV3 = $phodam->create(
    UuidInterface::class,
    'v3',
    [
        'namespace' => Uuid::NAMESPACE_DNS,
        'name' => 'example.com'
    ]
);

// Using config
$uuidV5 = $phodam->create(
    UuidInterface::class,
    'v5',
    null, // no overrides
    [
        'namespace' => Uuid::NAMESPACE_URL,
        'name' => 'https://example.com'
    ]
);
```

### Using Configuration for v1 and v6

UUID v1 and v6 can optionally take a node and clock sequence:

```php
// UUID v1 with node and clock sequence
$uuidV1 = $phodam->create(
    UuidInterface::class,
    'v1',
    [
        'node' => 0x123456789abc,
        'clockSeq' => 0x1234
    ]
);

// UUID v6 with config
$uuidV6 = $phodam->create(
    UuidInterface::class,
    'v6',
    null,
    [
        'node' => 0x123456789abc
    ]
);
```

### In Tests

```php
<?php

namespace YourApp\Tests;

use Phodam\Phodam;
use Phodam\PhodamSchema;
use YourVendor\PhodamRamseyUuid\Provider\RamseyUuidProviderBundle;
use Ramsey\Uuid\UuidInterface;

class UserTest extends TestCase
{
    private Phodam $phodam;
    
    protected function setUp(): void
    {
        parent::setUp();
        
        $schema = PhodamSchema::withDefaults();
        $phodam = $schema->getPhodam();
        $schema->registerBundle(RamseyUuidProviderBundle::class);
    }
    
    public function testUserCreation(): void
    {
        $user = new User();
        $user->setId($this->phodam->create(UuidInterface::class));
        $user->setName('John Doe');
        
        $this->assertInstanceOf(UuidInterface::class, $user->getId());
    }
    
    public function testUserWithSpecificUuidVersion(): void
    {
        // Use UUID v7 for time-ordered IDs
        $userId = $this->phodam->create(UuidInterface::class, 'v7');
        $user = new User();
        $user->setId($userId);
        
        $this->assertEquals(7, $userId->getVersion());
    }
}
```

## Advanced Configuration

### Custom Default Provider

If you want to change the default provider from v4 to another version, you can modify the `UuidInterfaceProvider` to use a different version, or create a new default provider:

```php
#[PhodamProvider(type: UuidInterface::class)]
class UuidInterfaceProvider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        // Make v7 the default instead of v4
        return Uuid::uuid7();
    }
}
```

### Provider with Custom Namespace Defaults

You can create a provider that uses custom default namespaces:

```php
#[PhodamProvider(type: UuidInterface::class, name: 'v5-custom')]
class UuidV5ProviderWithDefaults implements TypedProviderInterface
{
    private UuidInterface $defaultNamespace;
    
    public function __construct(?UuidInterface $defaultNamespace = null)
    {
        $this->defaultNamespace = $defaultNamespace ?? Uuid::NAMESPACE_DNS;
    }
    
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        $namespace = $overrides['namespace'] 
            ?? $config['namespace'] 
            ?? $this->defaultNamespace;
        
        $name = $overrides['name'] ?? $config['name'] ?? 'default-name';
        
        if (is_string($namespace)) {
            $namespace = Uuid::fromString($namespace);
        }
        
        return Uuid::uuid5($namespace, $name);
    }
}
```

### Error Handling

Add proper error handling for invalid configurations:

```php
use Phodam\Provider\CreationFailedException;

#[PhodamProvider(type: UuidInterface::class, name: 'v5-strict')]
class UuidV5StrictProvider implements TypedProviderInterface
{
    public function create(ProviderContextInterface $context): UuidInterface
    {
        $overrides = $context->getOverrides();
        $config = $context->getConfig();
        
        $namespace = $overrides['namespace'] ?? $config['namespace'] ?? null;
        $name = $overrides['name'] ?? $config['name'] ?? null;
        
        if ($namespace === null) {
            throw new CreationFailedException(
                'UUID v5 requires a namespace. Provide it via overrides or config.'
            );
        }
        
        if ($name === null) {
            throw new CreationFailedException(
                'UUID v5 requires a name. Provide it via overrides or config.'
            );
        }
        
        if (is_string($namespace)) {
            $namespace = Uuid::fromString($namespace);
        }
        
        return Uuid::uuid5($namespace, $name);
    }
}
```

## Summary

This guide has shown you how to:

1. **Create individual providers** for each UUID version (v1, v3, v4, v5, v6, v7)
2. **Create a default provider** for `UuidInterface` that generates v4 UUIDs
3. **Use PHP attributes** (`PhodamProvider`) to declare provider types and names
4. **Register named providers** for each version in a bundle
5. **Handle version-specific requirements** like namespaces for v3/v5
6. **Use overrides and configuration** to customize UUID generation

The bundle allows users to easily generate UUIDs in their tests:

```php
// Default (v4)
$uuid = $phodam->create(UuidInterface::class);

// Specific version
$uuid = $phodam->create(UuidInterface::class, 'v7');

// With configuration
$uuid = $phodam->create(UuidInterface::class, 'v5', null, [
    'namespace' => Uuid::NAMESPACE_DNS,
    'name' => 'example.com'
]);
```

This makes it easy to generate test data with UUIDs in your Phodam-powered tests!
