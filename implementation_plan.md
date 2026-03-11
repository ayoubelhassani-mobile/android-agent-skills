# Review & Harden the Android Domain Layer Skill for Clean Architecture

The `android-domain-layer` SKILL.md is mostly well-structured but contains a few violations and gaps relative to strict clean architecture. This plan addresses each issue.

## Identified Issues

| # | Issue | Severity |
|---|-------|----------|
| 1 | **`@Inject` on use case constructors** — This couples the domain layer to `javax.inject` / Hilt, which is a framework dependency. Strict clean architecture requires the domain layer to be framework-free. | High |
| 2 | **No explicit prohibition of `LiveData`** — Section 2 says "never entities or DTOs" but doesn't prohibit `LiveData`, `Uri`, or other Android types in repository signatures. `LiveData` is especially tempting. | Medium |
| 3 | **Missing domain-level error/exception guidance** — The skill shows `Result<Unit>` but never discusses how to model domain-specific errors (e.g., sealed classes for failure cases). Relying on `kotlin.Result` is fine, but domain exceptions should also be addressed. | Medium |
| 4 | **Missing guidance on keeping the Gradle module pure** — No mention of ensuring the domain module has **no** Android/Hilt/Room/Retrofit dependencies in its `build.gradle`. | Medium |
| 5 | **Package structure not specified** — No guidance on recommended package layout (`domain/model/`, `domain/repository/`, `domain/usecase/`). | Low |

## Proposed Changes

### Domain Layer Skill

#### [MODIFY] [SKILL.md](file:///c:/Users/elhas/Desktop/android-agent-skills/.github/skills/android-domain-layer/SKILL.md)

**Change 1 — Remove `@Inject` from use case examples (Issue #1)**
Replace `@Inject constructor` with plain `constructor` (or primary constructor syntax). Add a note explaining that DI annotations belong in the DI module, not in domain classes, and that the DI layer can still provide these classes without `@Inject` on the domain side (using `@Provides` in a Hilt module).

**Change 2 — Strengthen repository interface rules (Issue #2)**
Expand Section 2 rules to explicitly prohibit **all** Android framework types: `LiveData`, `MutableLiveData`, `Context`, `Uri`, `Intent`, `Bundle`, etc. Only pure Kotlin/`kotlinx.coroutines` types (`Flow`, `suspend`, `Result`) are allowed.

**Change 3 — Add domain error handling guidance (Issue #3)**
Add a new section covering domain-specific error modeling using sealed classes or sealed interfaces. Example:

```kotlin
sealed interface NewsError {
    data object NetworkUnavailable : NewsError
    data class NotFound(val id: String) : NewsError
}
```

**Change 4 — Add Gradle / module dependency guidance (Issue #4)**
Add a section stating that the domain module's `build.gradle` should use `kotlin("jvm")` (pure JVM plugin), **not** `com.android.library`, and must not depend on Room, Retrofit, Hilt, or any Android SDK artifact.

**Change 5 — Add recommended package structure (Issue #5)**
Add a short section showing the canonical package layout:
```
domain/
  model/        # Domain models
  repository/   # Repository interfaces
  usecase/      # Use cases
```

## User Review Required

> [!IMPORTANT]
> **Issue #1 (`@Inject` removal)** is the most impactful change. In many real-world Android projects, developers pragmatically use `@Inject` inside the domain layer for convenience. Removing it means use cases must be provided via `@Provides` functions in a Hilt module, which adds boilerplate. This plan follows **strict** clean architecture as requested, but please confirm you want to enforce this.

> [!NOTE]
> **Issue #3 (domain error modeling)** — The added sealed class approach is one common pattern. An alternative is to rely solely on `kotlin.Result` with custom exception types. Let me know if you prefer one over the other.

## Verification Plan

### Manual Verification
Since these are documentation-only changes (a SKILL.md guidance file), there is no automated test to run. Verification will consist of:

1. **Read the final SKILL.md** and confirm every code example is free of Android/framework imports and annotations.
2. **Cross-reference with the data-layer skill** to ensure the two skills are consistent and complementary (e.g., data layer references domain interfaces correctly).
3. **Checklist review** against the five issues listed above to confirm each is addressed.
