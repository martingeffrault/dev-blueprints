# Flutter & Dart (2025)

> **Last updated**: January 2026
> **Versions covered**: Flutter 3.32–3.38+, Dart 3.8–3.10+
> **Purpose**: Cross-platform mobile, web, and desktop development with native performance

---

## Philosophy (2025-2026)

Flutter in 2025 has matured into a **stable, production-ready framework** with Impeller as the default rendering engine, Dart 3.x language features, and official architecture recommendations. The focus has shifted from rapid feature additions to stability, performance, and developer experience.

**Key philosophical shifts:**
- **Official MVVM architecture** — Flutter now recommends separation into UI and Data layers with ViewModels
- **Feature-first organization** — Structure by feature, not by file type
- **Riverpod as preferred state management** — Compile-time safe, testable, and context-free
- **Declarative navigation** — go_router for URL-based, deep-link friendly routing
- **Dart 3.x language features** — Records, sealed classes, pattern matching are mainstream
- **Impeller default** — New rendering engine eliminates shader jank on iOS and Android
- **Immutability first** — Use freezed and immutable patterns for predictable state
- **Code generation matured** — build_runner, freezed, and json_serializable are standard practice

---

## TL;DR

- Use Flutter 3.32+ with Dart 3.8+ for latest features
- Structure by feature, not by type (feature-first architecture)
- Use Riverpod 2.x for state management (compile-time safe, testable)
- Use go_router for declarative, type-safe navigation
- Use freezed for immutable data models with code generation
- Leverage Dart 3.x: records, sealed classes, pattern matching, class modifiers
- Use `const` constructors everywhere possible to reduce rebuilds
- Split large widgets into small, focused, single-responsibility widgets
- Use `ListView.builder` / `GridView.builder` for lazy loading lists
- Repository pattern for data layer abstraction
- Test with `flutter_test` and `mocktail` for mocking

---

## Best Practices

### Project Structure (Feature-First)

```
my_app/
├── lib/
│   ├── main.dart                    # App entry point
│   ├── app.dart                     # MaterialApp configuration
│   ├── router/                      # Navigation
│   │   ├── app_router.dart          # go_router configuration
│   │   └── routes.dart              # Route constants
│   ├── core/                        # Shared infrastructure
│   │   ├── network/
│   │   │   ├── api_client.dart      # HTTP client (dio)
│   │   │   └── api_exceptions.dart
│   │   ├── storage/
│   │   │   └── local_storage.dart
│   │   ├── constants/
│   │   │   ├── app_constants.dart
│   │   │   └── api_endpoints.dart
│   │   └── utils/
│   │       └── extensions.dart
│   ├── shared/                      # Shared UI components
│   │   ├── widgets/
│   │   │   ├── app_button.dart
│   │   │   ├── app_text_field.dart
│   │   │   └── loading_indicator.dart
│   │   └── theme/
│   │       ├── app_theme.dart
│   │       └── app_colors.dart
│   └── features/                    # Feature modules
│       ├── auth/
│       │   ├── data/
│       │   │   ├── repositories/
│       │   │   │   └── auth_repository.dart
│       │   │   ├── data_sources/
│       │   │   │   └── auth_remote_data_source.dart
│       │   │   └── models/
│       │   │       └── user_model.dart
│       │   ├── domain/
│       │   │   ├── entities/
│       │   │   │   └── user.dart
│       │   │   └── repositories/
│       │   │       └── auth_repository_interface.dart
│       │   ├── presentation/
│       │   │   ├── screens/
│       │   │   │   ├── login_screen.dart
│       │   │   │   └── register_screen.dart
│       │   │   ├── widgets/
│       │   │   │   └── login_form.dart
│       │   │   └── providers/
│       │   │       └── auth_provider.dart
│       │   └── auth.dart            # Barrel export
│       ├── home/
│       │   ├── data/
│       │   ├── domain/
│       │   ├── presentation/
│       │   └── home.dart
│       └── products/
│           ├── data/
│           ├── domain/
│           ├── presentation/
│           └── products.dart
├── test/
│   ├── features/
│   │   ├── auth/
│   │   └── products/
│   └── shared/
├── pubspec.yaml
└── analysis_options.yaml
```

### State Management with Riverpod 2.x

```dart
// lib/features/products/presentation/providers/products_provider.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../../domain/entities/product.dart';
import '../../data/repositories/products_repository.dart';

part 'products_provider.g.dart';

// ✅ AsyncNotifierProvider for async state with business logic
@riverpod
class ProductsNotifier extends _$ProductsNotifier {
  @override
  Future<List<Product>> build() async {
    // Auto-dispose when no longer used
    return ref.watch(productsRepositoryProvider).getProducts();
  }

  Future<void> refresh() async {
    state = const AsyncLoading();
    state = await AsyncValue.guard(() =>
      ref.read(productsRepositoryProvider).getProducts()
    );
  }

  Future<void> addProduct(Product product) async {
    await ref.read(productsRepositoryProvider).addProduct(product);
    ref.invalidateSelf(); // Refresh the list
  }
}

// ✅ Simple provider for synchronous values
@riverpod
ProductsRepository productsRepository(ProductsRepositoryRef ref) {
  final apiClient = ref.watch(apiClientProvider);
  return ProductsRepositoryImpl(apiClient);
}

// ✅ FutureProvider for simple async data
@riverpod
Future<Product> productDetail(ProductDetailRef ref, String id) async {
  return ref.watch(productsRepositoryProvider).getProductById(id);
}

// ✅ Family provider for parameterized data
@riverpod
Future<List<Product>> productsByCategory(
  ProductsByCategoryRef ref,
  String category,
) async {
  return ref.watch(productsRepositoryProvider).getProductsByCategory(category);
}
```

```dart
// lib/features/products/presentation/screens/products_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../providers/products_provider.dart';
import '../widgets/product_card.dart';

class ProductsScreen extends ConsumerWidget {
  const ProductsScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ✅ Use ref.watch in build methods
    final productsAsync = ref.watch(productsNotifierProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('Products'),
        actions: [
          IconButton(
            icon: const Icon(Icons.refresh),
            // ✅ Use ref.read for one-time actions
            onPressed: () => ref.read(productsNotifierProvider.notifier).refresh(),
          ),
        ],
      ),
      body: productsAsync.when(
        data: (products) => ListView.builder(
          itemCount: products.length,
          itemBuilder: (context, index) => ProductCard(product: products[index]),
        ),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Error: $error'),
              ElevatedButton(
                onPressed: () => ref.invalidate(productsNotifierProvider),
                child: const Text('Retry'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

### Navigation with go_router

```dart
// lib/router/app_router.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../features/auth/presentation/providers/auth_provider.dart';
import '../features/auth/presentation/screens/login_screen.dart';
import '../features/home/presentation/screens/home_screen.dart';
import '../features/products/presentation/screens/products_screen.dart';
import '../features/products/presentation/screens/product_detail_screen.dart';
import 'routes.dart';

// ✅ Provider for router to enable reactive navigation
final routerProvider = Provider<GoRouter>((ref) {
  final authState = ref.watch(authStateProvider);

  return GoRouter(
    initialLocation: Routes.home,
    debugLogDiagnostics: true,

    // ✅ Redirect based on auth state
    redirect: (context, state) {
      final isAuthenticated = authState.valueOrNull?.isAuthenticated ?? false;
      final isAuthRoute = state.matchedLocation.startsWith('/auth');

      if (!isAuthenticated && !isAuthRoute) {
        return Routes.login;
      }
      if (isAuthenticated && isAuthRoute) {
        return Routes.home;
      }
      return null;
    },

    routes: [
      // ✅ Shell route for persistent bottom navigation
      StatefulShellRoute.indexedStack(
        builder: (context, state, navigationShell) {
          return ScaffoldWithNavBar(navigationShell: navigationShell);
        },
        branches: [
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: Routes.home,
                name: 'home',
                builder: (context, state) => const HomeScreen(),
              ),
            ],
          ),
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: Routes.products,
                name: 'products',
                builder: (context, state) => const ProductsScreen(),
                routes: [
                  // ✅ Nested route with path parameter
                  GoRoute(
                    path: ':id',
                    name: 'product-detail',
                    builder: (context, state) {
                      final id = state.pathParameters['id']!;
                      return ProductDetailScreen(productId: id);
                    },
                  ),
                ],
              ),
            ],
          ),
          StatefulShellBranch(
            routes: [
              GoRoute(
                path: Routes.profile,
                name: 'profile',
                builder: (context, state) => const ProfileScreen(),
              ),
            ],
          ),
        ],
      ),

      // ✅ Auth routes outside shell
      GoRoute(
        path: Routes.login,
        name: 'login',
        builder: (context, state) => const LoginScreen(),
      ),
      GoRoute(
        path: Routes.register,
        name: 'register',
        builder: (context, state) => const RegisterScreen(),
      ),
    ],

    // ✅ Error handling
    errorBuilder: (context, state) => ErrorScreen(error: state.error),
  );
});

// lib/router/routes.dart
abstract class Routes {
  static const home = '/';
  static const products = '/products';
  static const profile = '/profile';
  static const login = '/auth/login';
  static const register = '/auth/register';

  // ✅ Type-safe route builders
  static String productDetail(String id) => '/products/$id';
}
```

```dart
// ✅ Type-safe navigation usage
class ProductCard extends StatelessWidget {
  final Product product;

  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => context.push(Routes.productDetail(product.id)),
      // Or use named route: context.pushNamed('product-detail', pathParameters: {'id': product.id})
      child: Card(
        child: Text(product.name),
      ),
    );
  }
}
```

### Data Layer and Repository Pattern

```dart
// lib/features/products/domain/entities/product.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'product.freezed.dart';

// ✅ Domain entity with freezed for immutability
@freezed
class Product with _$Product {
  const factory Product({
    required String id,
    required String name,
    required String description,
    required double price,
    required String imageUrl,
    required String category,
    @Default(0) int stockQuantity,
    DateTime? createdAt,
  }) = _Product;
}
```

```dart
// lib/features/products/data/models/product_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';
import '../../domain/entities/product.dart';

part 'product_model.freezed.dart';
part 'product_model.g.dart';

// ✅ Data model with JSON serialization
@freezed
class ProductModel with _$ProductModel {
  const ProductModel._();

  const factory ProductModel({
    required String id,
    required String name,
    required String description,
    required double price,
    @JsonKey(name: 'image_url') required String imageUrl,
    required String category,
    @JsonKey(name: 'stock_quantity') @Default(0) int stockQuantity,
    @JsonKey(name: 'created_at') DateTime? createdAt,
  }) = _ProductModel;

  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);

  // ✅ Mapper to domain entity
  Product toDomain() => Product(
    id: id,
    name: name,
    description: description,
    price: price,
    imageUrl: imageUrl,
    category: category,
    stockQuantity: stockQuantity,
    createdAt: createdAt,
  );

  // ✅ Mapper from domain entity
  factory ProductModel.fromDomain(Product product) => ProductModel(
    id: product.id,
    name: product.name,
    description: product.description,
    price: product.price,
    imageUrl: product.imageUrl,
    category: product.category,
    stockQuantity: product.stockQuantity,
    createdAt: product.createdAt,
  );
}
```

```dart
// lib/features/products/domain/repositories/products_repository_interface.dart
import '../entities/product.dart';

// ✅ Abstract repository interface in domain layer
abstract class IProductsRepository {
  Future<List<Product>> getProducts();
  Future<Product> getProductById(String id);
  Future<List<Product>> getProductsByCategory(String category);
  Future<void> addProduct(Product product);
  Future<void> updateProduct(Product product);
  Future<void> deleteProduct(String id);
}
```

```dart
// lib/features/products/data/repositories/products_repository.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../../domain/entities/product.dart';
import '../../domain/repositories/products_repository_interface.dart';
import '../data_sources/products_remote_data_source.dart';
import '../models/product_model.dart';

part 'products_repository.g.dart';

@riverpod
ProductsRepository productsRepository(ProductsRepositoryRef ref) {
  return ProductsRepository(ref.watch(productsRemoteDataSourceProvider));
}

// ✅ Repository implementation in data layer
class ProductsRepository implements IProductsRepository {
  final ProductsRemoteDataSource _remoteDataSource;

  ProductsRepository(this._remoteDataSource);

  @override
  Future<List<Product>> getProducts() async {
    final models = await _remoteDataSource.getProducts();
    return models.map((m) => m.toDomain()).toList();
  }

  @override
  Future<Product> getProductById(String id) async {
    final model = await _remoteDataSource.getProductById(id);
    return model.toDomain();
  }

  @override
  Future<List<Product>> getProductsByCategory(String category) async {
    final models = await _remoteDataSource.getProductsByCategory(category);
    return models.map((m) => m.toDomain()).toList();
  }

  @override
  Future<void> addProduct(Product product) async {
    final model = ProductModel.fromDomain(product);
    await _remoteDataSource.addProduct(model);
  }

  @override
  Future<void> updateProduct(Product product) async {
    final model = ProductModel.fromDomain(product);
    await _remoteDataSource.updateProduct(model);
  }

  @override
  Future<void> deleteProduct(String id) async {
    await _remoteDataSource.deleteProduct(id);
  }
}
```

```dart
// lib/features/products/data/data_sources/products_remote_data_source.dart
import 'package:riverpod_annotation/riverpod_annotation.dart';
import '../../../../core/network/api_client.dart';
import '../models/product_model.dart';

part 'products_remote_data_source.g.dart';

@riverpod
ProductsRemoteDataSource productsRemoteDataSource(ProductsRemoteDataSourceRef ref) {
  return ProductsRemoteDataSource(ref.watch(apiClientProvider));
}

class ProductsRemoteDataSource {
  final ApiClient _apiClient;

  ProductsRemoteDataSource(this._apiClient);

  Future<List<ProductModel>> getProducts() async {
    final response = await _apiClient.get('/products');
    return (response.data as List)
        .map((json) => ProductModel.fromJson(json))
        .toList();
  }

  Future<ProductModel> getProductById(String id) async {
    final response = await _apiClient.get('/products/$id');
    return ProductModel.fromJson(response.data);
  }

  Future<List<ProductModel>> getProductsByCategory(String category) async {
    final response = await _apiClient.get('/products', queryParameters: {
      'category': category,
    });
    return (response.data as List)
        .map((json) => ProductModel.fromJson(json))
        .toList();
  }

  Future<void> addProduct(ProductModel product) async {
    await _apiClient.post('/products', data: product.toJson());
  }

  Future<void> updateProduct(ProductModel product) async {
    await _apiClient.put('/products/${product.id}', data: product.toJson());
  }

  Future<void> deleteProduct(String id) async {
    await _apiClient.delete('/products/$id');
  }
}
```

### Widget Composition

```dart
// ✅ Small, focused, single-responsibility widgets

// lib/shared/widgets/app_button.dart
import 'package:flutter/material.dart';

class AppButton extends StatelessWidget {
  final String text;
  final VoidCallback? onPressed;
  final bool isLoading;
  final ButtonVariant variant;

  const AppButton({
    super.key,
    required this.text,
    this.onPressed,
    this.isLoading = false,
    this.variant = ButtonVariant.primary,
  });

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: double.infinity,
      height: 48,
      child: ElevatedButton(
        onPressed: isLoading ? null : onPressed,
        style: _getButtonStyle(context),
        child: isLoading
            ? const SizedBox(
                width: 20,
                height: 20,
                child: CircularProgressIndicator(strokeWidth: 2),
              )
            : Text(text),
      ),
    );
  }

  ButtonStyle _getButtonStyle(BuildContext context) {
    return switch (variant) {
      ButtonVariant.primary => ElevatedButton.styleFrom(
          backgroundColor: Theme.of(context).colorScheme.primary,
          foregroundColor: Theme.of(context).colorScheme.onPrimary,
        ),
      ButtonVariant.secondary => ElevatedButton.styleFrom(
          backgroundColor: Theme.of(context).colorScheme.secondary,
          foregroundColor: Theme.of(context).colorScheme.onSecondary,
        ),
      ButtonVariant.outline => ElevatedButton.styleFrom(
          backgroundColor: Colors.transparent,
          foregroundColor: Theme.of(context).colorScheme.primary,
          side: BorderSide(color: Theme.of(context).colorScheme.primary),
        ),
    };
  }
}

enum ButtonVariant { primary, secondary, outline }
```

```dart
// ✅ Composing complex UI from small widgets
// lib/features/products/presentation/widgets/product_card.dart
import 'package:flutter/material.dart';
import 'package:cached_network_image/cached_network_image.dart';
import '../../domain/entities/product.dart';

class ProductCard extends StatelessWidget {
  final Product product;
  final VoidCallback? onTap;
  final VoidCallback? onAddToCart;

  const ProductCard({
    super.key,
    required this.product,
    this.onTap,
    this.onAddToCart,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      clipBehavior: Clip.antiAlias,
      child: InkWell(
        onTap: onTap,
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // ✅ Extract to separate widget for reusability
            _ProductImage(imageUrl: product.imageUrl),
            Padding(
              padding: const EdgeInsets.all(12),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  _ProductTitle(name: product.name),
                  const SizedBox(height: 4),
                  _ProductPrice(price: product.price),
                  const SizedBox(height: 8),
                  _AddToCartButton(onPressed: onAddToCart),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

// ✅ Small, const-constructible widgets
class _ProductImage extends StatelessWidget {
  final String imageUrl;

  const _ProductImage({required this.imageUrl});

  @override
  Widget build(BuildContext context) {
    return AspectRatio(
      aspectRatio: 16 / 9,
      child: CachedNetworkImage(
        imageUrl: imageUrl,
        fit: BoxFit.cover,
        placeholder: (_, __) => const _ImagePlaceholder(),
        errorWidget: (_, __, ___) => const _ImageError(),
      ),
    );
  }
}

class _ImagePlaceholder extends StatelessWidget {
  const _ImagePlaceholder();

  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.grey[200],
      child: const Center(child: CircularProgressIndicator()),
    );
  }
}

class _ImageError extends StatelessWidget {
  const _ImageError();

  @override
  Widget build(BuildContext context) {
    return Container(
      color: Colors.grey[200],
      child: const Icon(Icons.error),
    );
  }
}

class _ProductTitle extends StatelessWidget {
  final String name;

  const _ProductTitle({required this.name});

  @override
  Widget build(BuildContext context) {
    return Text(
      name,
      style: Theme.of(context).textTheme.titleMedium,
      maxLines: 2,
      overflow: TextOverflow.ellipsis,
    );
  }
}

class _ProductPrice extends StatelessWidget {
  final double price;

  const _ProductPrice({required this.price});

  @override
  Widget build(BuildContext context) {
    return Text(
      '\$${price.toStringAsFixed(2)}',
      style: Theme.of(context).textTheme.titleLarge?.copyWith(
            color: Theme.of(context).colorScheme.primary,
            fontWeight: FontWeight.bold,
          ),
    );
  }
}

class _AddToCartButton extends StatelessWidget {
  final VoidCallback? onPressed;

  const _AddToCartButton({this.onPressed});

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: double.infinity,
      child: FilledButton.icon(
        onPressed: onPressed,
        icon: const Icon(Icons.add_shopping_cart, size: 18),
        label: const Text('Add to Cart'),
      ),
    );
  }
}
```

### Dart 3.x Language Features

```dart
// ✅ Records for multiple return values
(String name, int age) getUserInfo() {
  return ('John Doe', 30);
}

// Named fields in records
({String name, int age}) getUserInfoNamed() {
  return (name: 'John Doe', age: 30);
}

void useRecords() {
  // Destructuring
  final (name, age) = getUserInfo();
  final (:name, :age) = getUserInfoNamed(); // shorthand
}
```

```dart
// ✅ Sealed classes for exhaustive pattern matching
sealed class AuthState {}

class AuthInitial extends AuthState {}
class AuthLoading extends AuthState {}
class AuthAuthenticated extends AuthState {
  final User user;
  AuthAuthenticated(this.user);
}
class AuthUnauthenticated extends AuthState {}
class AuthError extends AuthState {
  final String message;
  AuthError(this.message);
}

// ✅ Exhaustive switch expression
Widget buildAuthUI(AuthState state) {
  return switch (state) {
    AuthInitial() => const SplashScreen(),
    AuthLoading() => const LoadingScreen(),
    AuthAuthenticated(:final user) => HomeScreen(user: user),
    AuthUnauthenticated() => const LoginScreen(),
    AuthError(:final message) => ErrorScreen(message: message),
  };
}
```

```dart
// ✅ Pattern matching with guards
String describeNumber(int n) => switch (n) {
  0 => 'zero',
  1 => 'one',
  int x when x < 0 => 'negative',
  int x when x < 10 => 'small positive',
  int x when x < 100 => 'medium',
  _ => 'large',
};

// ✅ Object destructuring in patterns
class Point {
  final int x;
  final int y;
  Point(this.x, this.y);
}

String describePoint(Point point) => switch (point) {
  Point(x: 0, y: 0) => 'origin',
  Point(x: 0, y: _) => 'on y-axis',
  Point(x: _, y: 0) => 'on x-axis',
  Point(x: var x, y: var y) when x == y => 'on diagonal',
  _ => 'somewhere else',
};
```

```dart
// ✅ Class modifiers (Dart 3.0+)

// Can only be extended, not implemented
base class Animal {
  void breathe() {}
}

// Cannot be extended or implemented outside library
final class Logger {
  void log(String message) {}
}

// Can only be extended/implemented in same file
sealed class Result<T> {}
class Success<T> extends Result<T> {
  final T value;
  Success(this.value);
}
class Failure<T> extends Result<T> {
  final Exception error;
  Failure(this.error);
}

// Interface only (cannot be extended)
interface class Serializable {
  Map<String, dynamic> toJson();
}

// Mixin class (can be used as both class and mixin)
mixin class Walker {
  void walk() {}
}
```

### Performance Optimization

```dart
// ✅ Use const constructors to prevent rebuilds
class MyWidget extends StatelessWidget {
  const MyWidget({super.key}); // const constructor

  @override
  Widget build(BuildContext context) {
    return const Column(
      children: [
        Text('Static text'),     // const
        SizedBox(height: 16),    // const
        Icon(Icons.check),       // const
      ],
    );
  }
}
```

```dart
// ✅ Use ListView.builder for lazy loading
class ProductListView extends StatelessWidget {
  final List<Product> products;

  const ProductListView({super.key, required this.products});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: products.length,
      // ✅ Provide itemExtent for fixed-height items
      itemExtent: 100,
      // Or use prototypeItem for variable heights with same-sized items
      itemBuilder: (context, index) {
        return ProductListTile(product: products[index]);
      },
    );
  }
}

// ✅ Use GridView.builder for grids
class ProductGridView extends StatelessWidget {
  final List<Product> products;

  const ProductGridView({super.key, required this.products});

  @override
  Widget build(BuildContext context) {
    return GridView.builder(
      gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
        crossAxisCount: 2,
        childAspectRatio: 0.75,
      ),
      itemCount: products.length,
      itemBuilder: (context, index) {
        return ProductCard(product: products[index]);
      },
    );
  }
}
```

```dart
// ✅ Avoid rebuilding expensive widgets with RepaintBoundary
class ExpensiveChart extends StatelessWidget {
  final List<DataPoint> data;

  const ExpensiveChart({super.key, required this.data});

  @override
  Widget build(BuildContext context) {
    return RepaintBoundary(
      child: CustomPaint(
        painter: ChartPainter(data),
        size: const Size(300, 200),
      ),
    );
  }
}
```

```dart
// ✅ Use MediaQuery.sizeOf instead of MediaQuery.of to reduce rebuilds
// Only rebuilds when size changes, not when other MediaQuery properties change
class ResponsiveWidget extends StatelessWidget {
  const ResponsiveWidget({super.key});

  @override
  Widget build(BuildContext context) {
    // ❌ Rebuilds on ANY MediaQuery change (keyboard, orientation, etc.)
    // final size = MediaQuery.of(context).size;

    // ✅ Only rebuilds when size changes
    final size = MediaQuery.sizeOf(context);

    // ✅ Only get what you need
    final padding = MediaQuery.paddingOf(context);
    final viewInsets = MediaQuery.viewInsetsOf(context);

    return Container(
      width: size.width * 0.8,
      padding: EdgeInsets.only(bottom: viewInsets.bottom),
      child: const Text('Responsive content'),
    );
  }
}
```

```dart
// ✅ Cache expensive computations
class ExpensiveWidget extends ConsumerWidget {
  const ExpensiveWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ✅ Riverpod caches the result automatically
    final expensiveData = ref.watch(expensiveComputationProvider);

    return expensiveData.when(
      data: (data) => DataView(data: data),
      loading: () => const CircularProgressIndicator(),
      error: (e, _) => Text('Error: $e'),
    );
  }
}

@riverpod
Future<ExpensiveResult> expensiveComputation(ExpensiveComputationRef ref) async {
  // This is cached and only recomputed when dependencies change
  final input = ref.watch(inputProvider);
  return computeExpensiveResult(input);
}
```

### Testing

```dart
// test/features/products/data/repositories/products_repository_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:my_app/features/products/data/repositories/products_repository.dart';
import 'package:my_app/features/products/data/data_sources/products_remote_data_source.dart';
import 'package:my_app/features/products/data/models/product_model.dart';

class MockProductsRemoteDataSource extends Mock
    implements ProductsRemoteDataSource {}

void main() {
  late ProductsRepository repository;
  late MockProductsRemoteDataSource mockDataSource;

  setUp(() {
    mockDataSource = MockProductsRemoteDataSource();
    repository = ProductsRepository(mockDataSource);
  });

  group('getProducts', () {
    final productModels = [
      const ProductModel(
        id: '1',
        name: 'Product 1',
        description: 'Description 1',
        price: 29.99,
        imageUrl: 'https://example.com/1.jpg',
        category: 'electronics',
      ),
    ];

    test('returns list of products from data source', () async {
      // Arrange
      when(() => mockDataSource.getProducts())
          .thenAnswer((_) async => productModels);

      // Act
      final result = await repository.getProducts();

      // Assert
      expect(result.length, 1);
      expect(result.first.name, 'Product 1');
      verify(() => mockDataSource.getProducts()).called(1);
    });

    test('throws exception when data source fails', () async {
      // Arrange
      when(() => mockDataSource.getProducts())
          .thenThrow(Exception('Network error'));

      // Act & Assert
      expect(
        () => repository.getProducts(),
        throwsA(isA<Exception>()),
      );
    });
  });
}
```

```dart
// test/features/products/presentation/screens/products_screen_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';
import 'package:my_app/features/products/domain/entities/product.dart';
import 'package:my_app/features/products/presentation/providers/products_provider.dart';
import 'package:my_app/features/products/presentation/screens/products_screen.dart';

void main() {
  testWidgets('displays loading indicator while fetching products',
      (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          productsNotifierProvider.overrideWith(
            () => MockProductsNotifier()..state = const AsyncLoading(),
          ),
        ],
        child: const MaterialApp(home: ProductsScreen()),
      ),
    );

    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });

  testWidgets('displays products when loaded', (tester) async {
    final products = [
      const Product(
        id: '1',
        name: 'Test Product',
        description: 'Description',
        price: 29.99,
        imageUrl: 'https://example.com/1.jpg',
        category: 'test',
      ),
    ];

    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          productsNotifierProvider.overrideWith(
            () => MockProductsNotifier()..state = AsyncData(products),
          ),
        ],
        child: const MaterialApp(home: ProductsScreen()),
      ),
    );

    expect(find.text('Test Product'), findsOneWidget);
  });

  testWidgets('displays error message on failure', (tester) async {
    await tester.pumpWidget(
      ProviderScope(
        overrides: [
          productsNotifierProvider.overrideWith(
            () => MockProductsNotifier()
              ..state = AsyncError('Failed to load', StackTrace.current),
          ),
        ],
        child: const MaterialApp(home: ProductsScreen()),
      ),
    );

    expect(find.text('Error: Failed to load'), findsOneWidget);
    expect(find.text('Retry'), findsOneWidget);
  });
}

class MockProductsNotifier extends ProductsNotifier {
  @override
  Future<List<Product>> build() async => [];
}
```

---

## Anti-Patterns

### Directly Mutating State

**Why it's bad**: State changes won't trigger rebuilds, leading to stale UI.

```dart
// ❌ DON'T — Mutating state directly
class CartNotifier extends StateNotifier<List<CartItem>> {
  CartNotifier() : super([]);

  void addItem(CartItem item) {
    state.add(item); // Mutating the list directly!
    state = state;   // This hack doesn't reliably work
  }
}

// ✅ DO — Create new state object
class CartNotifier extends StateNotifier<List<CartItem>> {
  CartNotifier() : super([]);

  void addItem(CartItem item) {
    state = [...state, item]; // New list with added item
  }
}

// ✅ BETTER — Use freezed with copyWith
@freezed
class CartState with _$CartState {
  const factory CartState({
    @Default([]) List<CartItem> items,
  }) = _CartState;
}

class CartNotifier extends StateNotifier<CartState> {
  CartNotifier() : super(const CartState());

  void addItem(CartItem item) {
    state = state.copyWith(items: [...state.items, item]);
  }
}
```

### Giant Widgets

**Why it's bad**: Hard to maintain, impossible to reuse, and causes unnecessary rebuilds.

```dart
// ❌ DON'T — Massive build method
class ProductScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Products')),
      body: Column(
        children: [
          Container(
            height: 200,
            child: Image.network('url'),
          ),
          Padding(
            padding: EdgeInsets.all(16),
            child: Column(
              children: [
                Text('Title', style: TextStyle(fontSize: 24)),
                SizedBox(height: 8),
                Text('Description'),
                SizedBox(height: 16),
                Row(
                  children: [
                    Text('\$29.99'),
                    Spacer(),
                    ElevatedButton(
                      onPressed: () {},
                      child: Text('Add to Cart'),
                    ),
                  ],
                ),
              ],
            ),
          ),
          // ... 200 more lines
        ],
      ),
    );
  }
}

// ✅ DO — Break into small, focused widgets
class ProductScreen extends StatelessWidget {
  final Product product;

  const ProductScreen({super.key, required this.product});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(product.name)),
      body: Column(
        children: [
          ProductImage(url: product.imageUrl),
          ProductDetails(product: product),
          ProductActions(product: product),
        ],
      ),
    );
  }
}
```

### Using BuildContext After Async Gaps

**Why it's bad**: Context may be invalid after async operations, causing crashes.

```dart
// ❌ DON'T — Using context after await
Future<void> submitForm() async {
  await api.submit(data);
  Navigator.of(context).pop(); // Context may be invalid!
  ScaffoldMessenger.of(context).showSnackBar(...); // May crash!
}

// ✅ DO — Check mounted or use ref
Future<void> submitForm() async {
  await api.submit(data);
  if (!mounted) return; // For StatefulWidget
  Navigator.of(context).pop();
}

// ✅ BETTER — Use Riverpod (context-free)
Future<void> submitForm(WidgetRef ref) async {
  await ref.read(apiProvider).submit(data);
  ref.read(routerProvider).pop(); // No context needed
}
```

### Blocking the Main Thread

**Why it's bad**: Causes UI jank and freezes.

```dart
// ❌ DON'T — Heavy computation on main thread
Widget build(BuildContext context) {
  final processedData = heavyComputation(rawData); // Blocks UI!
  return ListView.builder(...);
}

// ✅ DO — Use compute for heavy operations
Future<void> processData() async {
  final result = await compute(heavyComputation, rawData);
  setState(() => processedData = result);
}

// ✅ BETTER — Use Riverpod with async
@riverpod
Future<ProcessedData> processedData(ProcessedDataRef ref) async {
  final rawData = await ref.watch(rawDataProvider.future);
  return compute(heavyComputation, rawData);
}
```

### Inline Anonymous Functions in Build

**Why it's bad**: Creates new function instances on every rebuild, preventing optimizations.

```dart
// ❌ DON'T — Anonymous functions in build
Widget build(BuildContext context) {
  return ListView.builder(
    itemCount: items.length,
    itemBuilder: (context, index) { // New function every build
      return ListTile(
        title: Text(items[index].name),
        onTap: () { // New function every build
          Navigator.of(context).push(...);
        },
      );
    },
  );
}

// ✅ DO — Extract to methods or use const widgets
class ItemList extends StatelessWidget {
  final List<Item> items;

  const ItemList({super.key, required this.items});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: _buildItem,
    );
  }

  Widget _buildItem(BuildContext context, int index) {
    return ItemTile(
      item: items[index],
      onTap: () => _onItemTap(context, items[index]),
    );
  }

  void _onItemTap(BuildContext context, Item item) {
    context.push('/items/${item.id}');
  }
}
```

### Ignoring dispose()

**Why it's bad**: Memory leaks, resource exhaustion, and crashes.

```dart
// ❌ DON'T — Forgetting to dispose
class MyWidget extends StatefulWidget {
  @override
  _MyWidgetState createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  late StreamSubscription _subscription;
  late AnimationController _controller;
  late TextEditingController _textController;

  @override
  void initState() {
    super.initState();
    _subscription = stream.listen(handleData);
    _controller = AnimationController(vsync: this);
    _textController = TextEditingController();
  }

  // Missing dispose! Memory leak!
}

// ✅ DO — Always dispose resources
class _MyWidgetState extends State<MyWidget> {
  late StreamSubscription _subscription;
  late AnimationController _controller;
  late TextEditingController _textController;

  @override
  void initState() {
    super.initState();
    _subscription = stream.listen(handleData);
    _controller = AnimationController(vsync: this);
    _textController = TextEditingController();
  }

  @override
  void dispose() {
    _subscription.cancel();
    _controller.dispose();
    _textController.dispose();
    super.dispose();
  }
}

// ✅ BETTER — Use Riverpod (auto-dispose)
@riverpod
Stream<Data> dataStream(DataStreamRef ref) {
  final controller = StreamController<Data>();

  ref.onDispose(() {
    controller.close(); // Automatically called when provider is disposed
  });

  return controller.stream;
}
```

### Using setState for Complex State

**Why it's bad**: Leads to prop drilling, tight coupling, and hard-to-test code.

```dart
// ❌ DON'T — setState for shared/complex state
class MyApp extends StatefulWidget {
  @override
  _MyAppState createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  User? user;
  List<Product> cart = [];
  ThemeMode themeMode = ThemeMode.light;

  void addToCart(Product p) => setState(() => cart.add(p));
  void setUser(User u) => setState(() => user = u);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: HomePage(
        user: user,
        cart: cart,
        onAddToCart: addToCart,  // Prop drilling!
        onSetUser: setUser,
      ),
    );
  }
}

// ✅ DO — Use Riverpod for state management
@riverpod
class CartNotifier extends _$CartNotifier {
  @override
  List<Product> build() => [];

  void add(Product product) {
    state = [...state, product];
  }
}

class ProductCard extends ConsumerWidget {
  final Product product;

  const ProductCard({super.key, required this.product});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ElevatedButton(
      onPressed: () => ref.read(cartNotifierProvider.notifier).add(product),
      child: const Text('Add to Cart'),
    );
  }
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| Flutter 3.32 | May 2025 | Web hot reload, Cupertino squircles, Firebase AI integrations |
| Dart 3.8 | May 2025 | Performance improvements, analysis server enhancements |
| Flutter 3.35 | Aug 2025 | Accessibility improvements, analysis performance (2s for large projects), Wasm dry-run builds |
| Dart 3.9 | Aug 2025 | MCP Server for AI assistants, AOT-compiled analysis server (50% faster) |
| Flutter 3.38 | Nov 2025 | Impeller default on Android API 29+, GenUI SDK, iOS 26 support, 16KB page size |
| Dart 3.10 | Nov 2025 | Dot shorthands syntax, stable build hooks, native analyzer plugins API |
| Flutter 4.0 | 2026 (Expected) | Impeller 2.0 Rendering Engine, Dart 4.x integration, framework restructure |

### Key 2025-2026 Highlights

**Impeller Default on Android (Flutter 3.38)**
- Default rendering engine on iOS since 3.29, Android API 29+ with Vulkan since 3.38
- 50% faster frame rasterization, consistent 120 FPS on high-refresh displays
- Eliminates shader compilation jank with precompiled shaders

**Android 16KB Memory Page Size (CRITICAL)**
- **Google Play deadline: November 1, 2025** — Apps targeting Android 15 must support 16KB pages
- Apps not compliant will be rejected from the Play Store
- Flutter 3.38+ handles this automatically for new builds

**Dart 3.10: Dot Shorthands Syntax**
```dart
// Before
Color color = Color.blue;

// After (Dart 3.10+) - Dot shorthands
Color color = .blue;  // Type inferred from context
```

**Dart Macros Cancelled, Augmentations Instead**
- Dart team cancelled macros due to Hot Reload performance concerns
- Instead: `augment` keyword for splitting class definitions across files
- Foundation for better code generation tools without compilation overhead

**Stable Build Hooks (Dart 3.10)**
- Game-changer for packages compiling/bundling native code (C, C++, Rust, Go)
- Enables seamless integration of native dependencies

**Native Analyzer Plugins API (Dart 3.10)**
- Write custom static analysis rules integrated directly into the Dart analyzer
- Rules run in IDE just like built-in lints

**WebAssembly Progress**
- Wasm becoming default compilation target for Flutter web
- Flutter 3.35 introduced `--wasm-dry-run` flag for preparation
- Significant performance improvements over JavaScript compilation

**Dart & Flutter MCP Server (Dart 3.9)**
- Bridge between IDE, CLI, codebase, and AI assistants
- Enables AI-powered development workflows

**FFIgen and JNIgen Early Access**
- Codegen solutions to simplify native platform API access
- Aims to replace method channels with direct Dart-to-native bridging

**Web Hot Reload (Flutter 3.32)**
- Hot reload now works for web development
- Config file support for `flutter run` on web

**Material and Cupertino Package Separation (2026)**
- Widgets being moved to separate packages
- Core framework will support platform-agnostic widgets only
- Platform-specific widgets (Material, Cupertino) maintained separately
- Part of Flutter 4.0 restructure

**Riverpod 3.0**
- Code generation improvements
- Better async handling
- Enhanced testing support

---

## Quick Reference

### Commands

| Command | Purpose |
|---------|---------|
| `flutter create my_app` | Create new project |
| `flutter run` | Run app on connected device |
| `flutter run -d chrome` | Run on web |
| `flutter build apk` | Build Android APK |
| `flutter build ios` | Build iOS app |
| `flutter test` | Run tests |
| `flutter analyze` | Run static analysis |
| `dart run build_runner build` | Run code generation |
| `dart run build_runner watch` | Watch for code gen changes |
| `flutter pub add <package>` | Add dependency |
| `flutter pub get` | Get dependencies |

### Essential Packages (2025)

| Package | Purpose |
|---------|---------|
| `flutter_riverpod` | State management |
| `riverpod_annotation` | Riverpod code generation |
| `go_router` | Declarative navigation |
| `freezed` | Immutable data models |
| `json_serializable` | JSON serialization |
| `dio` | HTTP client |
| `cached_network_image` | Image caching |
| `flutter_hooks` | React-style hooks |
| `mocktail` | Mocking for tests |
| `very_good_analysis` | Lint rules |

### Provider Types (Riverpod 2.x)

| Type | Use Case |
|------|----------|
| `@riverpod` function | Simple sync/async values |
| `@riverpod` class extending `_$Name` | Complex state with methods |
| `FutureProvider` | One-time async data |
| `StreamProvider` | Reactive streams |
| `Provider` | Sync computed values |

### Dart 3.x Class Modifiers

| Modifier | Can Construct | Can Extend | Can Implement |
|----------|---------------|------------|---------------|
| `class` | Yes | Yes | Yes |
| `base class` | Yes | Yes | No |
| `interface class` | Yes | No | Yes |
| `final class` | Yes | No | No |
| `sealed class` | No | Yes* | Yes* |
| `abstract class` | No | Yes | Yes |
| `mixin class` | Yes | Yes | Yes (as mixin) |

*Only in same library

---

## Resources

- [Flutter Documentation](https://docs.flutter.dev/)
- [Dart Documentation](https://dart.dev/guides)
- [Flutter App Architecture Guide](https://docs.flutter.dev/app-architecture/guide)
- [Riverpod Documentation](https://riverpod.dev/)
- [go_router Package](https://pub.dev/packages/go_router)
- [freezed Package](https://pub.dev/packages/freezed)
- [Code with Andrea - Flutter Tutorials](https://codewithandrea.com/)
- [Flutter Performance Best Practices](https://docs.flutter.dev/perf/best-practices)
- [Dart Patterns and Records](https://dart.dev/language/patterns)
- [Flutter Widget Catalog](https://docs.flutter.dev/ui/widgets)
