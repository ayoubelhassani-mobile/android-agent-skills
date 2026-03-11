---
name: android-architecture
description: Expert guidance on setting up and maintaining a modern Android application architecture using Clean Architecture and Hilt. Use this when asked about project structure, module setup, or dependency injection.
---

# Android Modern Architecture & Modularization

## Instructions

When designing or refactoring an Android application, adhere to the **Guide to App Architecture** and **Clean Architecture** principles.

### 1. High-Level Layers
Structure the application into three primary layers. Dependencies must strictly flow **inwards** — outer layers depend on inner layers, never the reverse.

*   **UI Layer (Presentation)**:
    *   **Responsibility**: Displaying data and handling user interactions.
    *   **Components**: Activities, Fragments, Composables, ViewModels.
    *   **Dependencies**: Depends on the Domain Layer only. **Never** depends on the Data Layer directly.
*   **Domain Layer (Business Logic)**:
    *   **Responsibility**: Encapsulating business rules and coordinating application behavior.
    *   **Components**: Use Cases (e.g., `GetLatestNewsUseCase`), Domain Models (pure Kotlin data classes), Repository Interfaces.
    *   **Pure Kotlin**: Must NOT contain any Android framework dependencies (no `android.*` or `androidx.*` imports) and must NOT use DI annotations (`@Inject`, `@Module`, etc.). Domain classes are plain Kotlin; the DI layer provides them via `@Provides`.
    *   **Dependencies**: Has no dependencies on other layers. Defines repository interfaces that the Data Layer implements.
    *   See the `android-domain-layer` skill for detailed guidance.
*   **Data Layer**:
    *   **Responsibility**: Managing application data (fetching, caching, saving).
    *   **Components**: Repository Implementations, Data Sources (Retrofit APIs, Room DAOs), DTOs, Entities.
    *   **Dependencies**: Implements domain-defined repository interfaces. Depends on external libraries (Room, Retrofit, etc.).
    *   See the `android-data-layer` skill for detailed guidance.

### 2. Dependency Injection with Hilt
Use **Hilt** for all dependency injection.

*   **@HiltAndroidApp**: Annotate the `Application` class.
*   **@AndroidEntryPoint**: Annotate Activities and Fragments.
*   **@HiltViewModel**: Annotate ViewModels; use standard `constructor` injection.
*   **Domain layer exception**: Domain classes (use cases, domain models) must **not** carry `@Inject` or any DI annotation. Provide them in a Hilt module using `@Provides`.
*   **Modules**:
    *   Use `@Module` and `@InstallIn(SingletonComponent::class)` for app-wide singletons (e.g., Network, Database).
    *   Use `@Binds` in an abstract class to bind interface implementations (cleaner than `@Provides`).

### 3. Modularization Strategy
For production apps, use a multi-module strategy to improve build times and separation of concerns.

*   **:app**: The main entry point, connects features.
*   **:core:domain**: Domain models, repository interfaces, and use cases (Pure Kotlin — no Android or DI dependencies).
*   **:core:data**: Repository implementations, data sources, DTOs, entities, database, network.
*   **:core:ui**: Shared Composables, Theme, Resources.
*   **:feature:[name]**: Standalone feature modules containing their own UI and ViewModels. Depends on `:core:domain` and `:core:ui`.

### 4. Checklist for Implementation
- [ ] Ensure `Domain` layer has no Android dependencies and no DI annotations.
- [ ] Repositories should default to main-safe suspend functions (use `Dispatchers.IO` internally if needed).
- [ ] ViewModels should interact with the UI layer via `StateFlow` (see `android-viewmodel` skill).
- [ ] UI layer depends only on Domain — never on Data layer implementation details.
- [ ] Domain models, repository interfaces, and use cases all live in `:core:domain` (or the `domain/` package in single-module projects).