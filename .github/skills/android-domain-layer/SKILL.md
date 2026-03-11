---
name: android-domain-layer
description: Guidance on implementing the Domain Layer with use cases, repository interfaces, and domain models following clean architecture.
---

# Android Domain Layer

## Instructions

The Domain Layer is the innermost layer in clean architecture. It contains core business logic and has no dependencies on other layers. All other layers depend on the domain layer — never the reverse.

### 1. Domain Models
*   **Role**: Represent core business data used throughout the application. Independent from database entities and network DTOs.
*   **Rules**: Plain Kotlin data classes with no framework annotations (`@Entity`, `@SerializedName`, etc.) and no Android imports.
    ```kotlin
    // domain/model/News.kt
    data class News(
        val id: String,
        val title: String,
        val content: String
    )
    ```

### 2. Repository Interfaces
*   **Role**: Define the contract for data access. The domain layer owns the interface; the data layer provides the implementation.
*   **Rules**:
    *   Use **only domain models** in parameter and return types — never entities, DTOs, or data-layer types.
    *   Use `Flow` (from `kotlinx.coroutines`) for observable data streams and `suspend` for one-shot operations.
    *   **Never** use Android framework types in signatures: no `LiveData`, `MutableLiveData`, `Context`, `Uri`, `Intent`, `Bundle`, `Cursor`, or any `android.*` type. Only pure Kotlin and `kotlinx.coroutines` types are permitted.
    ```kotlin
    // domain/repository/NewsRepository.kt
    interface NewsRepository {
        fun getNewsStream(): Flow<List<News>>
        suspend fun refreshNews(): Result<Unit>
    }
    ```

### 3. Use Cases
*   **Role**: Encapsulate a single piece of business logic or application behavior (e.g., validation, combining data from multiple repositories, triggering repository operations).
*   **Convention**: One public `operator fun invoke()` per use case. Name use cases as actions: `GetNewsUseCase`, `RefreshNewsUseCase`.
*   **Framework independence**: Use case classes must **not** carry DI annotations such as `@Inject`. They are plain Kotlin classes with constructor parameters. The DI layer is responsible for providing them (e.g., via `@Provides` in a Hilt module).
*   **When to use**: Use cases are justified when there is logic to orchestrate (validation, combining sources, conditional flows). If a use case simply delegates to a single repository method with no added logic, it may be omitted — the ViewModel can call the repository directly.
    ```kotlin
    // domain/usecase/GetNewsUseCase.kt
    class GetNewsUseCase(
        private val newsRepository: NewsRepository
    ) {
        operator fun invoke(): Flow<List<News>> = newsRepository.getNewsStream()
    }

    // domain/usecase/RefreshNewsUseCase.kt
    class RefreshNewsUseCase(
        private val newsRepository: NewsRepository
    ) {
        suspend operator fun invoke(): Result<Unit> = newsRepository.refreshNews()
    }
    ```

### 4. Dependency Rule
*   The domain packages (`domain/**`) must **never** import from `data/` or `ui/` packages.
*   They must have **no Android framework dependencies** — no `Context`, `LiveData`, `Room`, `Retrofit`, `WorkManager`, `Uri`, `Intent`, or any `android.*` / `androidx.*` import.
*   They must have **no DI framework annotations** — no `@Inject`, `@Module`, `@Provides`, or similar. DI wiring belongs in a dedicated `di/` package.
*   Only pure Kotlin, the Kotlin standard library, and `kotlinx.coroutines` are allowed as dependencies within domain code.
*   Other layers depend on the domain layer by implementing its interfaces (repositories) or calling its use cases.

### 5. Package Structure
Organize domain code under a `domain/` package with clear sub-packages:
```
domain/
  model/          # Domain models (e.g., News, User)
  repository/     # Repository interfaces (e.g., NewsRepository)
  usecase/        # Use cases (e.g., GetNewsUseCase)
```
