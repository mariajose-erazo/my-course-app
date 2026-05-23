# Platziflix — Arquitectura del Sistema (Big Picture)

Sistema multi-plataforma para una plataforma de cursos online tipo Netflix. Compuesto por **cuatro proyectos independientes** que se comunican a través de una única API REST.

---

## 1. Vista de pájaro

```
                          ┌─────────────────────────────┐
                          │       PostgreSQL 15         │
                          │   (platziflix_db, :5432)    │
                          └──────────────┬──────────────┘
                                         │  SQLAlchemy 2.0 + Alembic
                                         │
                          ┌──────────────▼──────────────┐
                          │   Backend  —  FastAPI       │
                          │   uv + Python 3.11          │
                          │   http://localhost:8000     │
                          │   (Docker Compose)          │
                          └──────┬──────────────┬───────┘
                                 │  REST/JSON   │
              ┌──────────────────┘              └──────────────────┐
              │                                                    │
   ┌──────────▼───────────┐                          ┌─────────────▼──────────────┐
   │ Frontend — Next.js15 │                          │     Mobile  (2 apps)       │
   │ React 19 + TS + SCSS │                          ├────────────────────────────┤
   │ http://localhost:3000│                          │ PlatziFlixAndroid          │
   │ Server Components +  │                          │   Kotlin + Jetpack Compose │
   │ fetch directo        │                          │   Retrofit + MVVM          │
   │ Vitest + RTL         │                          ├────────────────────────────┤
   └──────────────────────┘                          │ PlatziFlixiOS              │
                                                     │   Swift + SwiftUI          │
                                                     │   URLSession + Repository  │
                                                     └────────────────────────────┘
```

Los cuatro clientes (Frontend web, Android, iOS) son **iguales en jerarquía**: ninguno depende del otro y todos consumen el mismo contrato REST del Backend.

---

## 2. Componentes

### 2.1 Backend — `Backend/`
- **Stack**: FastAPI + SQLAlchemy 2.0 + Pydantic + Alembic, gestionado con `uv`, empacado en Docker.
- **Patrón**: Monolito con **Service Layer** (`app/services/course_service.py`). Sin Repository Pattern explícito — los servicios consultan SQLAlchemy directamente. DI vía `Depends()` de FastAPI.
- **Estructura clave**:
  - `app/main.py` — entry point y rutas
  - `app/models/` — ORM (Course, Teacher, Lesson, Class, CourseRating, course_teachers, BaseModel con soft-delete)
  - `app/schemas/` — Pydantic (solo ratings tipados, el resto de respuestas son dicts)
  - `app/services/course_service.py` — lógica de negocio
  - `app/db/base.py` — engine + `get_db()`
  - `app/alembic/versions/` — 2 migraciones (schema inicial + course_ratings)
  - `specs/` — contratos y guía de setup (no ejecutable)
- **Operación**: `make start | stop | migrate | seed | seed-fresh | logs | create-migration`

### 2.2 Frontend — `Frontend/`
- **Stack**: Next.js 15 (App Router) + React 19 + TypeScript + SCSS (CSS Modules) + Vitest + Testing Library.
- **Patrón**: **Hybrid data fetching**:
  - Catálogo y detalle → `fetch` directo desde Server Components (`cache: 'no-store'`)
  - Ratings → capa de servicios tipada en `src/services/ratingsApi.ts` con timeout, `ApiError` y type guards
- **Rutas (App Router)**:
  - `/` — grid de cursos (`src/app/page.tsx`)
  - `/course/[slug]` — detalle con `error.tsx`, `loading.tsx`, `not-found.tsx`
  - `/classes/[class_id]` — reproductor de video
- **Estilos**: SCSS global (`src/styles/vars.scss`, `reset.scss`) inyectado a todos los módulos vía `sassOptions.prependData` en `next.config.ts`. Token system con función `color()`.
- **Componentes reutilizables**: `Course`, `CourseDetail`, `StarRating`, `VideoPlayer`.

### 2.3 Mobile/PlatziFlixAndroid — `Mobile/PlatziFlixAndroid/`
- **Stack**: Kotlin + Jetpack Compose + Retrofit 2.9 + Coroutines + Navigation Compose.
- **Arquitectura**: **Clean Architecture** estricta con capas:
  - `domain/` — `Course` model + `CourseRepository` interface
  - `data/` — `CourseDTO`, `CourseMapper`, `RemoteCourseRepository`, `ApiService` (Retrofit)
  - `presentation/` — Composables (`CourseListScreen`, `CourseCard`, `LoadingIndicator`, `ErrorMessage`) + `CourseListViewModel`
- **Estado**: **MVVM + MVI híbrido** — `StateFlow<CourseListUiState>` + eventos `CourseListUiEvent`.
- **Red**: `BASE_URL = http://10.0.2.2:8000/` (loopback del emulador Android al host).
- **Navegación**: Navigation Compose declarado pero rutas internas aún TODO.

### 2.4 Mobile/PlatziFlixiOS — `Mobile/PlatziFlixiOS/`
- **Stack**: Swift + SwiftUI + URLSession (async/await).
- **Arquitectura**: **Clean Architecture + MVVM + Repository**:
  - `Domain/` — `Course` (con `isActive`, `displayDescription`) + `CourseRepository` protocol
  - `Data/` — `CourseDTO`, `CourseDetailDTO`, `CourseMapper`, `RemoteCourseRepository`, `CourseAPIEndpoints`
  - `Services/` — `NetworkManager` singleton + `APIEndpoint` protocol + `NetworkError` enum
  - `Presentation/` — `CourseListView` + `CourseCardView` + `CourseListViewModel` (`@MainActor ObservableObject`)
- **Estado**: `@Published` properties (courses, isLoading, errorMessage, searchText con debounce 300ms).
- **Red**: `BASE_URL = http://localhost:8000` (orientado a simulador).
- **Navegación**: `NavigationView`; navegación a detalle es TODO.

---

## 3. Modelo de datos compartido

Backend es la fuente de verdad. Las cuatro entidades viven en `Backend/app/models/`:

| Entidad           | Campos clave                                              | Relaciones                                         |
| ----------------- | --------------------------------------------------------- | -------------------------------------------------- |
| **Course**        | id, name, description, thumbnail, **slug** (unique)       | N:N Teacher, 1:N Lesson, 1:N CourseRating          |
| **Teacher**       | id, name, email (unique)                                  | N:N Course                                         |
| **Lesson**        | id, course_id, name, description, slug, video_url         | N:1 Course                                         |
| **Class**         | (modelo definido pero **no usado** en la API)             | —                                                  |
| **CourseRating**  | id, course_id, user_id, rating (CHECK 1–5)                | N:1 Course                                         |
| `course_teachers` | tabla de unión (course_id, teacher_id)                    | —                                                  |
| `BaseModel`       | id, created_at, updated_at, deleted_at (soft-delete)      | Abstract base                                      |

**Observación crítica**: en la API, lo que el contrato llama `"classes"` se sirve realmente desde la tabla `Lesson` (ver mapeo en `course_service.py`). El modelo `Class` está huérfano. Los clientes (Frontend tipo `Class`, Android `Course`, iOS `Course`) usan el lenguaje **"Class"** del contrato, no el del modelo.

---

## 4. Contrato API (estado real, no el de CLAUDE.md)

| Método | Ruta                                              | Propósito                                                  |
| ------ | ------------------------------------------------- | ---------------------------------------------------------- |
| GET    | `/`                                               | Bienvenida                                                 |
| GET    | `/health`                                         | Status + DB + courses_count                                |
| GET    | `/courses`                                        | Lista de cursos **con `average_rating` y `total_ratings`** |
| GET    | `/courses/{slug}`                                 | Detalle con teachers, lessons (como "classes") y rating_distribution |
| GET    | `/classes/{class_id}`                             | Una clase (lección) por id                                 |
| POST   | `/courses/{course_id}/ratings`                    | Crear/actualizar rating de un usuario                      |
| GET    | `/courses/{course_id}/ratings`                    | Listar ratings activos                                     |
| GET    | `/courses/{course_id}/ratings/stats`              | avg + total + distribución                                 |
| GET    | `/courses/{course_id}/ratings/user/{user_id}`     | Rating de un usuario                                       |
| PUT    | `/courses/{course_id}/ratings/{user_id}`          | Actualizar (404 si no existe)                              |
| DELETE | `/courses/{course_id}/ratings/{user_id}`          | Soft-delete                                                |

Hoy **solo el Frontend** consume los endpoints de ratings (`ratingsApi.ts`). Android e iOS aún se limitan al catálogo (`/courses`).

---

## 5. Flujos end-to-end

**Ver catálogo (Web)**
`Browser → Next.js Server Component (page.tsx) → fetch http://localhost:8000/courses → CourseService.get_all_courses() → SQLAlchemy + agregación de ratings → JSON → render del grid con <Course>`

**Ver detalle + reproductor (Web)**
`/course/[slug] → fetch /courses/{slug} → click clase → /classes/[class_id] → fetch /classes/{class_id} → <VideoPlayer>`

**Calificar curso (Web)**
`<StarRating> → ratingsApi.createRating() → POST /courses/{id}/ratings → CourseService.add_course_rating() (upsert) → 201 + RatingResponse`

**Catálogo (Android)**
`MainActivity → CourseListScreen → CourseListViewModel.handleEvent(Load) → RemoteCourseRepository.getCourses() → ApiService (Retrofit) → CourseMapper.fromDTOList → StateFlow → recomposición`

**Catálogo (iOS)**
`CourseListView.task → CourseListViewModel.loadCourses() → RemoteCourseRepository → NetworkManager.request() (URLSession async) → CourseMapper.toDomain → @Published courses → SwiftUI re-render`

---

## 6. Base URLs por cliente

| Cliente   | URL base del Backend         | Razón                                                    |
| --------- | ---------------------------- | -------------------------------------------------------- |
| Frontend  | `http://localhost:8000`      | Mismo host de desarrollo                                 |
| Android   | `http://10.0.2.2:8000`       | Loopback del emulador Android hacia el host              |
| iOS       | `http://localhost:8000`      | Simulador iOS comparte loopback con el host              |
| Dispositivos reales | (no configurado)   | Requeriría IP de LAN + permisos de cleartext / ATS       |

---

## 7. Testing por capa

| Proyecto  | Framework                | Cobertura observada                                            |
| --------- | ------------------------ | -------------------------------------------------------------- |
| Backend   | pytest + httpx           | `test_course_rating_service.py`, `test_rating_db_constraints.py`, `test_rating_endpoints.py`, `test_main.py` |
| Frontend  | Vitest + Testing Library | `Course.test.tsx`, `StarRating.test.tsx`, `VideoPlayer.test.tsx`, `classes/[class_id]/page.test.tsx` |
| Android   | (configuración estándar) | Sin tests visibles aún                                         |
| iOS       | (configuración estándar) | Sin tests visibles aún                                         |

---

## 8. Inconsistencias y deuda detectadas

1. **`Lesson` vs `Class`** — el modelo `Class` existe en el ORM pero no se usa; el contrato público habla de "classes" mientras el almacenamiento es `lessons`. Cualquier extensión debería decidir si renombrar o eliminar `Class`.
2. **Endpoints de ratings no documentados en CLAUDE.md / specs/** — el contrato real ya tiene 11 endpoints (CLAUDE.md menciona 4).
3. **Ratings solo consumidos por Web** — Android e iOS aún no soportan calificar.
4. **Navegación interna móvil** — Navigation Compose (Android) y navegación a detalle (iOS) están marcadas como TODO.
5. **Pydantic parcial** — solo el módulo de ratings usa schemas tipados; el resto de respuestas son `dict` ad-hoc.
6. **Sin Repository en Backend** — pese a lo que indica CLAUDE.md, el service consulta SQLAlchemy directamente.
7. **URL base hardcodeada por cliente** — no hay configuración por entorno (dev/staging/prod).

---

## 9. Cómo levantar todo el sistema

```bash
# 1. Backend (Docker + DB + migraciones + seed)
cd Backend
make start
make migrate
make seed

# 2. Frontend
cd Frontend
yarn install
yarn dev          # http://localhost:3000

# 3. Android — abrir Mobile/PlatziFlixAndroid en Android Studio, Run sobre emulador
# 4. iOS     — abrir Mobile/PlatziFlixiOS en Xcode, Run sobre simulador
```

Verificación rápida: `http://localhost:8000/health` debe devolver `database: "ok"` y `courses_count > 0`.
