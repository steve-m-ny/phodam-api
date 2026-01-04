# Phodam API

The API for Phodam - a PHP library for generating test data objects.

## Overview

Phodam is a flexible and extensible library for creating test data in PHP. It provides a simple API for generating objects and arrays with realistic test data, making it easier to write comprehensive unit tests.

## Documentation

- **[Interfaces and Creating Provider Bundles](docs/INTERFACES_AND_PROVIDER_BUNDLES.md)** - Learn about the core interfaces and how to create your own provider bundles
- **[Ramsey UUID Provider Bundle Guide](docs/Ramsey UUID Provider Bundle.md)** - Step-by-step guide to creating a provider bundle for ramsey/uuid

## Quick Start

```php
<?php

use Phodam\Phodam;
use Phodam\PhodamSchema;

$schema = PhodamSchema::withDefaults();
$phodam = $schema->getPhodam();

// Register a provider bundle
$schema->registerBundle(YourProviderBundle::class);

// Create an instance
$user = $phodam->create(User::class);
```

## Requirements

- PHP 8.4+

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
