---
name: android-data-layer
description: Guidance on implementing the Data Layer using Repository pattern, Room (Local), and Retrofit (Remote) with offline-first synchronization.
---

# Android Data Layer & Offline-First

## Instructions

The Data Layer coordinates data from multiple sources. All data-layer code lives under `data/` and never depends on the domain layer's internals — it only implements domain-defined repository interfaces.

### 1. Repository Pattern
*   **Role**: Single Source of Truth (SSOT). The local database is always the source of truth.
*   **Contract**: The domain layer owns the repository interface (see `android-domain-layer` skill). The data layer provides the implementation.
*   **Implementation**:
    ```kotlin
    class NewsRepositoryImpl @Inject constructor(
        private val localDataSource: NewsLocalDataSource,
        private val remoteDataSource: NewsRemoteDataSource,
        @IoDispatcher private val ioDispatcher: CoroutineDispatcher
    ) : NewsRepository {

        override fun getNewsStream(): Flow<List<News>> =
            localDataSource.getAllNews()
                .map { entities -> entities.map { it.toDomain() } }
                .flowOn(ioDispatcher)

        override suspend fun refreshNews(): Result<Unit> = withContext(ioDispatcher) {
            runCatching {
                val remoteNews = remoteDataSource.fetchLatest()
                localDataSource.insertAll(remoteNews.map { it.toEntity() })
            }
        }
    }
    ```

### 2. Data Sources
Data sources are data-layer internal abstractions. Both the interface and implementation live in the data layer.

*   **LocalDataSource**: Wraps Room DAOs.
    ```kotlin
    interface NewsLocalDataSource {
        fun getAllNews(): Flow<List<NewsEntity>>
        suspend fun insertAll(entities: List<NewsEntity>)
    }

    class NewsLocalDataSourceImpl @Inject constructor(
        private val dao: NewsDao
    ) : NewsLocalDataSource {
        override fun getAllNews(): Flow<List<NewsEntity>> = dao.getAll()
        override suspend fun insertAll(entities: List<NewsEntity>) = dao.upsertAll(entities)
    }
    ```
*   **RemoteDataSource**: Wraps Retrofit API.
    ```kotlin
    interface NewsRemoteDataSource {
        suspend fun fetchLatest(): List<NewsDto>
    }

    class NewsRemoteDataSourceImpl @Inject constructor(
        private val api: NewsApi
    ) : NewsRemoteDataSource {
        override suspend fun fetchLatest(): List<NewsDto> = api.getLatestNews()
    }
    ```

### 3. Local Persistence (Room)
*   **Entities**: Define `@Entity` data classes. These are data-layer models, never exposed to domain.
*   **DAOs**: Return `Flow<T>` for observable reads. Use `@Upsert` for inserts to handle conflicts.
*   **Access**: DAOs must only be used inside a `LocalDataSource` implementation, never directly in repositories.

### 4. Remote Data (Retrofit)
*   **DTOs**: Define response data classes matching the API schema. Never expose DTOs outside the data layer.
*   **API interfaces**: Use `suspend` functions.
*   **Access**: Retrofit APIs must only be used inside a `RemoteDataSource` implementation, never directly in repositories.

### 5. Data Mapping
*   **Purpose**: Convert between Remote DTOs, Database Entities, and Domain models.
*   **Direction**: Remote data always flows through local storage: `Dto → Entity` (on sync), `Entity → Domain` (on read). Never map `Dto → Domain` directly — this violates SSOT.
*   **Implementation**: Use extension functions.
    ```kotlin
    fun NewsDto.toEntity(): NewsEntity {
        return NewsEntity(id = id, title = title, content = content)
    }

    fun NewsEntity.toDomain(): News {
        return News(id = id, title = title, content = content)
    }
    ```

### 6. Synchronization
*   **Read — Stale-While-Revalidate**: Emit local data immediately via `Flow`, trigger a background refresh that updates the local DB (which automatically re-emits through the `Flow`).
*   **Write — Outbox Pattern**: Save the change locally with a sync flag, then use `WorkManager` to push to server.
    ```kotlin
    // Entity with sync flag
    @Entity
    data class PendingAction(
        @PrimaryKey(autoGenerate = true) val id: Long = 0,
        val type: String,       // e.g. "CREATE_ARTICLE"
        val payload: String,    // JSON-serialized data
        val createdAt: Long = System.currentTimeMillis()
    )

    // WorkManager worker
    class SyncWorker @AssistedInject constructor(
        @Assisted context: Context,
        @Assisted params: WorkerParameters,
        private val pendingActionDao: PendingActionDao,
        private val remoteDataSource: NewsRemoteDataSource
    ) : CoroutineWorker(context, params) {
        override suspend fun doWork(): Result {
            val pending = pendingActionDao.getAll()
            for (action in pending) {
                try {
                    // Push to server based on action.type
                    remoteDataSource.push(action.payload)
                    pendingActionDao.delete(action)
                } catch (e: Exception) {
                    return Result.retry()
                }
            }
            return Result.success()
        }
    }
    ```

### 7. Threading
*   Inject `@IoDispatcher` (a `CoroutineDispatcher` qualifier) into repositories and data sources that perform disk or network I/O.
*   Use `flowOn(ioDispatcher)` for `Flow` streams and `withContext(ioDispatcher)` for suspend functions.
*   Room and Retrofit already switch threads internally, but the dispatcher annotation keeps the contract explicit and testable.

### 8. Dependency Injection
*   Bind interfaces to implementations in Hilt modules.
    ```kotlin
    @Module
    @InstallIn(SingletonComponent::class)
    abstract class DataModule {
        @Binds
        abstract fun bindNewsRepository(impl: NewsRepositoryImpl): NewsRepository

        @Binds
        abstract fun bindLocalDataSource(impl: NewsLocalDataSourceImpl): NewsLocalDataSource

        @Binds
        abstract fun bindRemoteDataSource(impl: NewsRemoteDataSourceImpl): NewsRemoteDataSource
    }
    ```