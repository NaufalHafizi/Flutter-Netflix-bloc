## Flutter SOLID + BLoC Architecture Rule (Repository-Aware)

This rule guides all AI-generated code for Flutter projects using Clean Architecture (data/domain/presentation), SOLID principles, BLoC/Cubit for state management, Dio for networking, dartz Either for error handling, and get_it for dependency injection.

It is repository-aware and must align with the existing structure in this project. Adapt paths and naming to the current codebase and keep them consistent as the repository grows.

---

### 1) Project structure and placement (MUST follow)

Place new code according to these folders if they exist; otherwise create them:
- `lib/core/`: cross-cutting concerns (configs, constants, network, usecase base, shared models/entities)
- `lib/common/`: reusable widgets, helpers, mappers
- `lib/domain/<feature>/`:
  - `entities/`: pure domain models (suffix: `Entity`)
  - `repositories/`: abstract repository contracts (suffix: `Repository`)
  - `usecases/`: single-purpose interactors (suffix: `UseCase`)
- `lib/data/<feature>/`:
  - `models/`: data-layer models (suffix: `Model`)
  - `sources/`: remote/local data sources (suffix: `ApiServiceImpl` or `LocalServiceImpl`)
  - `repositories/`: concrete `RepositoryImpl` implementing domain contracts
- `lib/presentation/<feature>/`:
  - `bloc/` or `cubit/`: state management (suffix: `Bloc`/`Cubit`, `State`, optional `Event`)
  - `pages/`: screens
  - `widgets/`: feature-specific UI components
- `lib/service_locator.dart`: register DI for services, repositories, and use cases

Always mirror existing features (e.g., `auth/`, `movie/`, `tv/`) when adding new features.

---

### 2) SOLID principles (enforced heuristics)
- Single Responsibility: each class has one reason to change. Use dedicated files per class.
- Open/Closed: extend behavior via new classes (e.g., new `UseCase`, new `ApiServiceImpl`) rather than modifying existing ones.
- Liskov Substitution: implementations must honor their abstractions (`RepositoryImpl` must be substitutable for `Repository`).
- Interface Segregation: keep repository contracts minimal and cohesive; avoid “fat” interfaces.
- Dependency Inversion: UI depends on domain abstractions; domain does not depend on data/UI. Use `get_it` for inversion at composition root.

---

### 3) Dependency Injection (get_it) conventions
- Use the existing `GetIt sl` instance in `lib/service_locator.dart`.
- Register new dependencies the same way current code does (prefer `registerSingleton<T>(T())` to stay consistent unless lifecycle demands `registerLazySingleton`).
- Group registrations by layer: `DioClient` and infrastructure first, then services (data sources), repositories, and finally use cases.
- When adding a new feature, update `service_locator.dart` by registering:
  - Data source service (e.g., `XApiServiceImpl`)
  - Repository implementation (e.g., `XRepositoryImpl`)
  - Each `UseCase`

Example registration snippet (follow existing style and imports):
```dart
// Services
sl.registerSingleton<FeatureService>(FeatureApiServiceImpl());

// Repositories
sl.registerSingleton<FeatureRepository>(FeatureRepositoryImpl());

// Usecases
sl.registerSingleton<DoSomethingFeatureUseCase>(DoSomethingFeatureUseCase());
```

---

### 4) Use cases (lib/core/usecase/usecase.dart)
- Base contract exists:
```dart
abstract class UseCase<Type, Params> {
  Future<Type> call({Params params});
}
```
- Implement use cases in `lib/domain/<feature>/usecases`.
- Naming: VerbNoun + `UseCase` (e.g., `GetTrendingMoviesUseCase`).
- Return type: use `Either<FailureOrMessage, T>` consistently with current code. If the codebase uses raw `Either`, follow the local pattern.
- Do not inject services directly in use cases; call the corresponding repository via `sl<Repository>()` (consistent with the existing pattern), or receive the repository via constructor if the project evolves to constructor injection.

Example:
```dart
class SearchMovieUseCase extends UseCase<Either, String> {
  @override
  Future<Either> call({String? params}) async {
    return await sl<MovieRepository>().searchMovie(params!);
  }
}
```

---

### 5) Repositories and data sources
- Domain repositories (`lib/domain/<feature>/repositories`) define abstract contracts only.
- Data implementations live in `lib/data/<feature>/repositories` with suffix `RepositoryImpl` and must depend on data sources in `lib/data/<feature>/sources`.
- Data sources:
  - Remote: use `DioClient` from `lib/core/network/dio_client.dart` and `ApiUrl` constants from `lib/core/constants/api_url.dart`.
  - Return `Either` from all data methods. Do not throw inside repositories; convert to `Left`.
- Mapping:
  - Convert `Model` to `Entity` using a `Mapper` in `lib/common/helper/mapper` (create a new mapper if needed).
  - Repositories handle mapping from raw JSON/model to domain entities.

Repository implementation pattern (follow existing folding/mapping style):
```dart
class FeatureRepositoryImpl extends FeatureRepository {
  @override
  Future<Either> getSomething() async {
    final returnedData = await sl<FeatureService>().getSomething();
    return returnedData.fold(
      (error) => Left(error),
      (data) {
        final entities = List.from(data['content'])
            .map((item) => FeatureMapper.toEntity(FeatureModel.fromJson(item)))
            .toList();
        return Right(entities);
      },
    );
  }
}
```

---

### 6) BLoC/Cubit state management
- Prefer `Cubit` for simple request/response flows; use `Bloc` with events only when multiple orthogonal events exist.
- Place state management under `lib/presentation/<feature>/bloc` (or `cubit`).
- Naming:
  - `FeatureCubit`/`FeatureBloc`
  - `FeatureState` with concrete states like `FeatureLoading`, `FeatureLoaded`, `FeatureFailure`
- Interact with use cases through DI:
```dart
class TrendingsCubit extends Cubit<TrendingsState> {
  TrendingsCubit() : super(TrendingMoviesLoading());

  Future<void> getTrendingMovies() async {
    final result = await sl<GetTrendingMoviesUseCase>().call();
    result.fold(
      (error) => emit(FailureLoadTrendingMovies(errorMessage: error)),
      (data) => emit(TrendingMoviesLoaded(movies: data)),
    );
  }
}
```
- UI uses `BlocProvider`/`BlocBuilder` and never calls repositories/services directly.
- One Cubit/Bloc per logical UI unit. Avoid putting multiple unrelated responsibilities in one state manager.

---

### 7) Networking (Dio)
- Use `DioClient` from `lib/core/network/dio_client.dart` for HTTP.
- Build URLs with `ApiUrl` constants. Do not hardcode strings.
- Respect configured timeouts and interceptors.
- Data sources may expose methods like `getSomething`, `searchSomething`, returning `Either`.

Example remote call in a service:
```dart
class FeatureApiServiceImpl implements FeatureService {
  @override
  Future<Either> getSomething() async {
    try {
      final response = await sl<DioClient>().get(ApiUrl.someEndpoint);
      return Right(response.data);
    } catch (e) {
      return Left(e.toString());
    }
  }
}
```

---

### 8) Entities, models, and mappers
- Domain entities are immutable data holders with names ending in `Entity` and live in `lib/domain/<feature>/entities`.
- Data models end in `Model` and live in `lib/data/<feature>/models` with `fromJson`/`toJson`.
- Mappers live under `lib/common/helper/mapper`. Create one per aggregate/root type, with static methods like `toEntity`.
- UI must use `Entity` types; never `Model`.

---

### 9) Naming, imports, and formatting
- Use explicit imports from feature paths; avoid barrel files unless already present.
- File naming: snake_case, class names: PascalCase.
- Keep files focused; one top-level type per file where feasible.
- Follow `flutter_lints` and existing `analysis_options.yaml`.

---

### 10) Error handling and loading patterns
- Always model loading and failure states in Cubit/Bloc states.
- Use `Either` to represent success/failure; never throw from repository outward.
- Map low-level errors to a user-presentable message at the Cubit/Bloc layer.

---

### 11) Feature scaffolding checklist (apply these steps atomically)
1. Domain
   - Create `entities/YourEntity.dart` if needed
   - Define `repositories/YourRepository.dart` (abstract)
   - Add `usecases/VerbYourNounUseCase.dart` extending `UseCase<Either, Params?>`
2. Data
   - Create `models/YourModel.dart` with `fromJson`
   - Create `sources/YourApiServiceImpl.dart` using `DioClient`
   - Create `repositories/YourRepositoryImpl.dart` that calls the service and maps to entities using a mapper
3. Common
   - Add `common/helper/mapper/your_mapper.dart` with `toEntity`
4. Presentation
   - Add `presentation/<feature>/bloc`: `YourCubit` + `YourState`
   - Add `presentation/<feature>/pages` and `widgets` as needed
5. DI
   - Register service, repository, and all new use cases in `service_locator.dart`
6. UI
   - Wire with `BlocProvider` and `BlocBuilder`/`BlocListener`

---

### 12) Repository-aware behavior (auto-adapt as the project grows)
- When generating code:
  - Detect and mirror existing feature folders (e.g., `auth/`, `movie/`, `tv/`) and their internal structure.
  - Reuse existing base classes and helpers: `UseCase`, `DioClient`, `ApiUrl`, mappers under `common/helper/mapper`.
  - Match current DI registration style in `service_locator.dart` (constructor usage, singleton vs lazy-singleton).
  - Follow existing naming patterns for states, cubits, repositories, services, and mappers.
- If a folder is missing for a new feature, create it following the established structure.
- Never break or refactor existing code automatically; add new code in a backward-compatible way consistent with the current architecture.

---

### 13) Testing guidance (recommended)
- Add unit tests for use cases and repositories in `test/`.
- Mock repositories/services for use case tests; verify `Either` behavior for success/failure.
- For Cubit/Bloc, test state transitions given mocked use cases.

---

### 14) Performance and UX
- Debounce user-driven searches in Cubit/Bloc.
- Cache where appropriate at the repository layer if the project introduces caching.
- Keep rebuild surfaces small using focused `BlocBuilder` widgets.

---

### 15) Migration notes
- If future refactors prefer constructor injection for use cases/repositories, keep this rule consistent with the prevailing pattern in the codebase. Always prioritize the repository’s current conventions.