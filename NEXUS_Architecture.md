# NEXUS — Software Architecture Document
### Inter-School App Development Competition | Android (Kotlin) | v1.0

---

## TABLE OF CONTENTS

1. [Executive Summary](#1-executive-summary)
2. [Technology Stack](#2-technology-stack)
3. [Architecture Overview](#3-architecture-overview)
4. [Folder Structure](#4-folder-structure)
5. [Database Schema](#5-database-schema)
6. [Navigation Architecture](#6-navigation-architecture)
7. [Screen Inventory](#7-screen-inventory)
8. [User Roles & Permissions](#8-user-roles--permissions)
9. [User Stories](#9-user-stories)
10. [Development Roadmap](#10-development-roadmap)
11. [Architectural Decision Records (ADRs)](#11-architectural-decision-records)

---

## 1. Executive Summary

**NEXUS** is a school management super-app designed to bridge communication gaps between students, teachers, and administrators. It consolidates incident reporting, educational resources, an AI assistant, push notifications, and analytics into a single cohesive platform.

| Attribute        | Detail                                              |
|------------------|-----------------------------------------------------|
| Platform         | Android (API 26+ / Android 8.0 Oreo and above)     |
| Language         | Kotlin                                              |
| Architecture     | Clean Architecture + MVVM                           |
| Backend          | Firebase (Auth, Firestore, Storage, FCM)            |
| Offline Support  | Room Database + WorkManager sync                    |
| Target Users     | Students, Teachers, Administrators                  |
| Design System    | Material 3 (Material You)                           |

---

## 2. Technology Stack

### 2.1 Core Android

| Category         | Library / Tool                              | Purpose                              |
|------------------|---------------------------------------------|--------------------------------------|
| Language         | Kotlin 1.9+                                 | Primary language                     |
| Build Tool       | Gradle (Kotlin DSL)                         | Build system                         |
| Min SDK          | API 26 (Android 8.0)                        | Wide device coverage (~95%)          |
| Target SDK       | API 35                                      | Latest features                      |
| DI               | Hilt (Dagger)                               | Dependency Injection                 |
| Async            | Kotlin Coroutines + Flow                    | Reactive async programming           |
| Navigation       | Navigation Component 2.x                   | Single-Activity navigation           |
| Lifecycle        | ViewModel + LiveData / StateFlow            | UI state management                  |
| UI Binding       | ViewBinding                                 | Type-safe view references            |

### 2.2 Firebase Services

| Service              | Usage                                             |
|----------------------|---------------------------------------------------|
| Firebase Auth        | Email/password + Google Sign-In                  |
| Cloud Firestore      | Primary NoSQL cloud database                     |
| Firebase Storage     | File/image uploads (resources, avatars)          |
| Firebase Cloud Msg.  | Push notifications (FCM)                         |
| Firebase Analytics   | Event tracking, user behavior                    |
| Firebase Crashlytics | Crash reporting and diagnostics                  |
| Firebase Remote Cfg. | Feature flags and remote config                  |

### 2.3 Local Persistence

| Library       | Purpose                                       |
|---------------|-----------------------------------------------|
| Room 2.x      | Local SQLite ORM for offline data             |
| DataStore     | Preferences storage (user settings, tokens)   |
| WorkManager   | Background sync jobs (offline → online)       |

### 2.4 UI & Design

| Library              | Purpose                                       |
|----------------------|-----------------------------------------------|
| Material 3           | Design components (M3 theme)                 |
| Coil                 | Async image loading + caching                |
| Lottie               | JSON-based animations                        |
| MPAndroidChart       | Analytics charts and graphs                  |
| Shimmer              | Loading skeleton placeholders                |

### 2.5 Networking & Serialization

| Library            | Purpose                                         |
|--------------------|-------------------------------------------------|
| Kotlin Serialization | JSON parsing (Firestore DTOs)               |
| OkHttp             | HTTP client (if external APIs needed)           |

### 2.6 Testing

| Library              | Purpose                                       |
|----------------------|-----------------------------------------------|
| JUnit 4/5            | Unit testing                                 |
| MockK                | Kotlin-friendly mocking                      |
| Turbine              | Flow testing                                 |
| Espresso             | UI instrumented testing                      |
| Hilt Testing         | DI for test environment                      |

---

## 3. Architecture Overview

### 3.1 Layered Architecture (Clean Architecture + MVVM)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                            │
│                                                                      │
│   Activity (MainActivity)  ←──  NavHostFragment                      │
│        │                                                             │
│   Fragments (Screens)  ←──────  ViewModels  ←──  UI State / Events  │
│        │                             │                               │
│   ViewBinding / Adapters        StateFlow / SharedFlow              │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │  calls Use Cases
┌──────────────────────────────────▼───────────────────────────────────┐
│                          DOMAIN LAYER                                │
│                                                                      │
│   Use Cases (Interactors)   ─── Domain Models ─── Repository Interfaces│
│                                                                      │
│   (Pure Kotlin — NO Android / Firebase dependencies)                │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │  implemented by
┌──────────────────────────────────▼───────────────────────────────────┐
│                           DATA LAYER                                 │
│                                                                      │
│   RepositoryImpl                                                     │
│        ├── RemoteDataSource   ── Firebase (Firestore, Auth, Storage) │
│        └── LocalDataSource    ── Room DAO + Entities                 │
│                                                                      │
│   Mappers: Entity ↔ Domain Model ↔ DTO                              │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow Pattern

```
User Action
    │
    ▼
Fragment (observes UI state)
    │
    ▼
ViewModel (sends Intent / calls use case)
    │
    ▼
Use Case (business logic, pure Kotlin)
    │
    ▼
Repository Interface
    │
    ├──── [Online]  RemoteDataSource → Firestore / Auth / Storage
    │                                         │
    │                              write-through to Room
    │
    └──── [Offline] LocalDataSource → Room Database
                                          │
                              WorkManager queues sync job
                                          │
                              When online: sync to Firestore
```

### 3.3 UI State Management

Every screen uses a sealed `UiState` class:

```
UiState<T>
  ├── Loading
  ├── Success(data: T)
  ├── Empty
  └── Error(message: String)
```

One-time events (navigation, snackbars, toasts) use `SharedFlow<UiEvent>`.

### 3.4 Offline Sync Strategy

| Scenario             | Behavior                                                  |
|----------------------|-----------------------------------------------------------|
| Online read          | Fetch from Firestore, cache to Room                       |
| Online write         | Write to Firestore + update Room                          |
| Offline read         | Serve from Room cache                                     |
| Offline write        | Write to Room with `syncStatus = PENDING`                 |
| Network restored     | WorkManager triggers sync of all PENDING records          |
| Conflict resolution  | Server timestamp wins (last-write-wins)                   |

---

## 4. Folder Structure

```
nexus/
├── app/
│   ├── src/
│   │   ├── main/
│   │   │   ├── java/com/nexus/app/
│   │   │   │   │
│   │   │   │   ├── NexusApplication.kt            ← Hilt application entry point
│   │   │   │   │
│   │   │   │   ├── core/                           ← Shared, reusable infrastructure
│   │   │   │   │   ├── base/
│   │   │   │   │   │   ├── BaseFragment.kt
│   │   │   │   │   │   ├── BaseViewModel.kt
│   │   │   │   │   │   └── BaseAdapter.kt
│   │   │   │   │   ├── di/
│   │   │   │   │   │   ├── AppModule.kt
│   │   │   │   │   │   ├── FirebaseModule.kt
│   │   │   │   │   │   ├── DatabaseModule.kt
│   │   │   │   │   │   └── RepositoryModule.kt
│   │   │   │   │   ├── utils/
│   │   │   │   │   │   ├── NetworkMonitor.kt
│   │   │   │   │   │   ├── DateUtils.kt
│   │   │   │   │   │   ├── FileUtils.kt
│   │   │   │   │   │   ├── ValidationUtils.kt
│   │   │   │   │   │   └── Constants.kt
│   │   │   │   │   ├── extensions/
│   │   │   │   │   │   ├── FragmentExtensions.kt
│   │   │   │   │   │   ├── ViewExtensions.kt
│   │   │   │   │   │   ├── FlowExtensions.kt
│   │   │   │   │   │   └── StringExtensions.kt
│   │   │   │   │   ├── ui/
│   │   │   │   │   │   ├── UiState.kt
│   │   │   │   │   │   ├── UiEvent.kt
│   │   │   │   │   │   └── components/            ← Shared custom views
│   │   │   │   │   │       ├── LoadingDialog.kt
│   │   │   │   │   │       ├── EmptyStateView.kt
│   │   │   │   │   │       └── ConfirmationDialog.kt
│   │   │   │   │   └── workers/
│   │   │   │   │       ├── SyncWorker.kt
│   │   │   │   │       └── NotificationWorker.kt
│   │   │   │   │
│   │   │   │   ├── data/                           ← Data layer
│   │   │   │   │   ├── local/
│   │   │   │   │   │   ├── NexusDatabase.kt        ← Room database
│   │   │   │   │   │   ├── dao/
│   │   │   │   │   │   │   ├── UserDao.kt
│   │   │   │   │   │   │   ├── ReportDao.kt
│   │   │   │   │   │   │   ├── ResourceDao.kt
│   │   │   │   │   │   │   └── NotificationDao.kt
│   │   │   │   │   │   └── entities/
│   │   │   │   │   │       ├── UserEntity.kt
│   │   │   │   │   │       ├── ReportEntity.kt
│   │   │   │   │   │       ├── ResourceEntity.kt
│   │   │   │   │   │       └── NotificationEntity.kt
│   │   │   │   │   ├── remote/
│   │   │   │   │   │   ├── datasource/
│   │   │   │   │   │   │   ├── AuthRemoteDataSource.kt
│   │   │   │   │   │   │   ├── UserRemoteDataSource.kt
│   │   │   │   │   │   │   ├── ReportRemoteDataSource.kt
│   │   │   │   │   │   │   ├── ResourceRemoteDataSource.kt
│   │   │   │   │   │   │   └── NotificationRemoteDataSource.kt
│   │   │   │   │   │   └── dto/                   ← Firestore-mapped DTOs
│   │   │   │   │   │       ├── UserDto.kt
│   │   │   │   │   │       ├── ReportDto.kt
│   │   │   │   │   │       ├── ResourceDto.kt
│   │   │   │   │   │       └── NotificationDto.kt
│   │   │   │   │   ├── repository/                ← Repository implementations
│   │   │   │   │   │   ├── AuthRepositoryImpl.kt
│   │   │   │   │   │   ├── UserRepositoryImpl.kt
│   │   │   │   │   │   ├── ReportRepositoryImpl.kt
│   │   │   │   │   │   ├── ResourceRepositoryImpl.kt
│   │   │   │   │   │   └── NotificationRepositoryImpl.kt
│   │   │   │   │   └── mappers/                   ← DTO ↔ Entity ↔ Domain
│   │   │   │   │       ├── UserMapper.kt
│   │   │   │   │       ├── ReportMapper.kt
│   │   │   │   │       ├── ResourceMapper.kt
│   │   │   │   │       └── NotificationMapper.kt
│   │   │   │   │
│   │   │   │   ├── domain/                         ← Domain layer (pure Kotlin)
│   │   │   │   │   ├── model/
│   │   │   │   │   │   ├── User.kt
│   │   │   │   │   │   ├── Report.kt
│   │   │   │   │   │   ├── Resource.kt
│   │   │   │   │   │   ├── Notification.kt
│   │   │   │   │   │   ├── AnalyticsSummary.kt
│   │   │   │   │   │   └── AiClassification.kt
│   │   │   │   │   ├── repository/                ← Interfaces
│   │   │   │   │   │   ├── AuthRepository.kt
│   │   │   │   │   │   ├── UserRepository.kt
│   │   │   │   │   │   ├── ReportRepository.kt
│   │   │   │   │   │   ├── ResourceRepository.kt
│   │   │   │   │   │   └── NotificationRepository.kt
│   │   │   │   │   └── usecase/
│   │   │   │   │       ├── auth/
│   │   │   │   │       │   ├── LoginUseCase.kt
│   │   │   │   │       │   ├── RegisterUseCase.kt
│   │   │   │   │       │   ├── LogoutUseCase.kt
│   │   │   │   │       │   └── ResetPasswordUseCase.kt
│   │   │   │   │       ├── report/
│   │   │   │   │       │   ├── CreateReportUseCase.kt
│   │   │   │   │       │   ├── GetReportsUseCase.kt
│   │   │   │   │       │   ├── UpdateReportStatusUseCase.kt
│   │   │   │   │       │   └── DeleteReportUseCase.kt
│   │   │   │   │       ├── resource/
│   │   │   │   │       │   ├── UploadResourceUseCase.kt
│   │   │   │   │       │   ├── GetResourcesUseCase.kt
│   │   │   │   │       │   └── DeleteResourceUseCase.kt
│   │   │   │   │       ├── ai/
│   │   │   │   │       │   └── ClassifyQueryUseCase.kt
│   │   │   │   │       ├── notification/
│   │   │   │   │       │   ├── GetNotificationsUseCase.kt
│   │   │   │   │       │   └── MarkReadUseCase.kt
│   │   │   │   │       └── admin/
│   │   │   │   │           ├── GetAllUsersUseCase.kt
│   │   │   │   │           ├── UpdateUserRoleUseCase.kt
│   │   │   │   │           └── GetAnalyticsUseCase.kt
│   │   │   │   │
│   │   │   │   └── presentation/                   ← UI layer
│   │   │   │       ├── MainActivity.kt             ← Single Activity host
│   │   │   │       ├── auth/
│   │   │   │       │   ├── splash/
│   │   │   │       │   │   └── SplashFragment.kt
│   │   │   │       │   ├── login/
│   │   │   │       │   │   ├── LoginFragment.kt
│   │   │   │       │   │   └── LoginViewModel.kt
│   │   │   │       │   ├── register/
│   │   │   │       │   │   ├── RegisterFragment.kt
│   │   │   │       │   │   └── RegisterViewModel.kt
│   │   │   │       │   └── forgot_password/
│   │   │   │       │       ├── ForgotPasswordFragment.kt
│   │   │   │       │       └── ForgotPasswordViewModel.kt
│   │   │   │       ├── dashboard/
│   │   │   │       │   ├── DashboardFragment.kt
│   │   │   │       │   └── DashboardViewModel.kt
│   │   │   │       ├── reports/
│   │   │   │       │   ├── list/
│   │   │   │       │   │   ├── ReportListFragment.kt
│   │   │   │       │   │   ├── ReportListViewModel.kt
│   │   │   │       │   │   └── ReportAdapter.kt
│   │   │   │       │   ├── detail/
│   │   │   │       │   │   ├── ReportDetailFragment.kt
│   │   │   │       │   │   └── ReportDetailViewModel.kt
│   │   │   │       │   └── create/
│   │   │   │       │       ├── CreateReportFragment.kt
│   │   │   │       │       └── CreateReportViewModel.kt
│   │   │   │       ├── resources/
│   │   │   │       │   ├── hub/
│   │   │   │       │   │   ├── ResourceHubFragment.kt
│   │   │   │       │   │   ├── ResourceHubViewModel.kt
│   │   │   │       │   │   └── ResourceAdapter.kt
│   │   │   │       │   ├── detail/
│   │   │   │       │   │   ├── ResourceDetailFragment.kt
│   │   │   │       │   │   └── ResourceDetailViewModel.kt
│   │   │   │       │   └── upload/
│   │   │   │       │       ├── UploadResourceFragment.kt
│   │   │   │       │       └── UploadResourceViewModel.kt
│   │   │   │       ├── ai/
│   │   │   │       │   ├── AiAssistantFragment.kt
│   │   │   │       │   └── AiAssistantViewModel.kt
│   │   │   │       ├── notifications/
│   │   │   │       │   ├── NotificationsFragment.kt
│   │   │   │       │   ├── NotificationsViewModel.kt
│   │   │   │       │   └── NotificationAdapter.kt
│   │   │   │       ├── profile/
│   │   │   │       │   ├── ProfileFragment.kt
│   │   │   │       │   ├── ProfileViewModel.kt
│   │   │   │       │   ├── edit/
│   │   │   │       │   │   ├── EditProfileFragment.kt
│   │   │   │       │   │   └── EditProfileViewModel.kt
│   │   │   │       │   └── settings/
│   │   │   │       │       ├── SettingsFragment.kt
│   │   │   │       │       └── SettingsViewModel.kt
│   │   │   │       └── admin/
│   │   │   │           ├── dashboard/
│   │   │   │           │   ├── AdminDashboardFragment.kt
│   │   │   │           │   └── AdminDashboardViewModel.kt
│   │   │   │           ├── users/
│   │   │   │           │   ├── UserManagementFragment.kt
│   │   │   │           │   ├── UserManagementViewModel.kt
│   │   │   │           │   ├── UserAdapter.kt
│   │   │   │           │   └── detail/
│   │   │   │           │       ├── UserDetailFragment.kt
│   │   │   │           │       └── UserDetailViewModel.kt
│   │   │   │           ├── moderation/
│   │   │   │           │   ├── ModerationFragment.kt
│   │   │   │           │   └── ModerationViewModel.kt
│   │   │   │           └── analytics/
│   │   │   │               ├── AnalyticsFragment.kt
│   │   │   │               └── AnalyticsViewModel.kt
│   │   │   │
│   │   │   ├── res/
│   │   │   │   ├── layout/                         ← XML layouts
│   │   │   │   ├── navigation/
│   │   │   │   │   ├── nav_auth.xml                ← Auth nav graph
│   │   │   │   │   ├── nav_main.xml                ← Root nav graph
│   │   │   │   │   ├── nav_reports.xml
│   │   │   │   │   ├── nav_resources.xml
│   │   │   │   │   └── nav_admin.xml
│   │   │   │   ├── values/
│   │   │   │   │   ├── themes.xml                  ← Material 3 theme
│   │   │   │   │   ├── colors.xml
│   │   │   │   │   ├── strings.xml
│   │   │   │   │   └── dimens.xml
│   │   │   │   ├── drawable/
│   │   │   │   ├── anim/
│   │   │   │   └── raw/                            ← Lottie JSON files
│   │   │   └── AndroidManifest.xml
│   │   └── test/ + androidTest/
│   └── build.gradle.kts
├── build.gradle.kts
├── settings.gradle.kts
└── google-services.json
```

---

## 5. Database Schema

### 5.1 Firestore Collections

#### `users` collection
```
users/{userId}
  ├── uid              : String      [Firebase Auth UID]
  ├── displayName      : String
  ├── email            : String
  ├── photoUrl         : String?
  ├── role             : String      ["STUDENT" | "TEACHER" | "ADMIN"]
  ├── classGrade       : String?     [e.g., "10-A"]
  ├── schoolId         : String
  ├── isActive         : Boolean
  ├── fcmToken         : String?     [for push notifications]
  ├── createdAt        : Timestamp
  └── updatedAt        : Timestamp
```

#### `reports` collection
```
reports/{reportId}
  ├── reportId         : String
  ├── title            : String
  ├── description      : String
  ├── category         : String      ["INCIDENT" | "MAINTENANCE" | "BULLYING" | "ACADEMIC" | "OTHER"]
  ├── priority         : String      ["LOW" | "MEDIUM" | "HIGH" | "URGENT"]
  ├── status           : String      ["PENDING" | "IN_REVIEW" | "RESOLVED" | "REJECTED"]
  ├── submittedBy      : String      [userId ref]
  ├── assignedTo       : String?     [userId ref — teacher/admin]
  ├── attachmentUrls   : List<String>
  ├── aiClassification : String?     [assigned by rule-based AI]
  ├── isAnonymous      : Boolean
  ├── resolution       : String?
  ├── resolvedAt       : Timestamp?
  ├── createdAt        : Timestamp
  └── updatedAt        : Timestamp

  └── comments/ (sub-collection)
        comments/{commentId}
          ├── commentId    : String
          ├── authorId     : String
          ├── text         : String
          ├── isInternal   : Boolean  [admin-only notes]
          └── createdAt    : Timestamp
```

#### `resources` collection
```
resources/{resourceId}
  ├── resourceId       : String
  ├── title            : String
  ├── description      : String
  ├── category         : String      ["NOTES" | "PAST_PAPERS" | "VIDEOS" | "LINKS" | "FORMS"]
  ├── subject          : String
  ├── grade            : String?
  ├── fileUrl          : String      [Firebase Storage URL]
  ├── thumbnailUrl     : String?
  ├── fileType         : String      ["PDF" | "IMAGE" | "VIDEO" | "LINK"]
  ├── fileSizeBytes    : Long?
  ├── uploadedBy       : String      [userId ref]
  ├── approvalStatus   : String      ["PENDING" | "APPROVED" | "REJECTED"]
  ├── downloadCount    : Int
  ├── tags             : List<String>
  ├── createdAt        : Timestamp
  └── updatedAt        : Timestamp
```

#### `notifications` collection
```
notifications/{notificationId}
  ├── notificationId   : String
  ├── title            : String
  ├── body             : String
  ├── type             : String      ["ANNOUNCEMENT" | "REPORT_UPDATE" | "RESOURCE" | "SYSTEM"]
  ├── targetAudience   : String      ["ALL" | "STUDENTS" | "TEACHERS" | "SPECIFIC"]
  ├── targetUserIds    : List<String>?
  ├── linkedDocId      : String?     [report or resource ID]
  ├── sentBy           : String      [userId of admin]
  ├── isRead           : Boolean     [for user-level reads, use sub-collection]
  └── createdAt        : Timestamp

  └── readReceipts/ (sub-collection)
        readReceipts/{userId}
          ├── userId       : String
          └── readAt       : Timestamp
```

#### `analytics_events` collection
```
analytics_events/{eventId}
  ├── eventId          : String
  ├── eventType        : String      ["REPORT_CREATED" | "RESOURCE_DOWNLOADED" | "LOGIN" | "AI_QUERY"]
  ├── userId           : String
  ├── metadata         : Map<String, Any>
  └── timestamp        : Timestamp
```

#### `app_config` collection (admin-controlled)
```
app_config/settings
  ├── maintenanceMode  : Boolean
  ├── allowRegistration: Boolean
  ├── appVersion       : String
  ├── reportCategories : List<String>
  └── resourceSubjects : List<String>
```

---

### 5.2 Room Database (Local Cache)

**Database name:** `nexus_database`  
**Version:** 1

#### Table: `users`
| Column          | Type    | Constraint          |
|-----------------|---------|---------------------|
| uid             | TEXT    | PRIMARY KEY         |
| display_name    | TEXT    | NOT NULL            |
| email           | TEXT    | NOT NULL            |
| photo_url       | TEXT    | NULLABLE            |
| role            | TEXT    | NOT NULL            |
| class_grade     | TEXT    | NULLABLE            |
| school_id       | TEXT    | NOT NULL            |
| is_active       | INTEGER | NOT NULL (0/1)      |
| last_synced_at  | INTEGER | NOT NULL (epoch ms) |

#### Table: `reports`
| Column            | Type    | Constraint              |
|-------------------|---------|-------------------------|
| report_id         | TEXT    | PRIMARY KEY             |
| title             | TEXT    | NOT NULL                |
| description       | TEXT    | NOT NULL                |
| category          | TEXT    | NOT NULL                |
| priority          | TEXT    | NOT NULL                |
| status            | TEXT    | NOT NULL                |
| submitted_by      | TEXT    | NOT NULL                |
| assigned_to       | TEXT    | NULLABLE                |
| is_anonymous      | INTEGER | NOT NULL                |
| ai_classification | TEXT    | NULLABLE                |
| sync_status       | TEXT    | NOT NULL [SYNCED/PENDING]|
| created_at        | INTEGER | NOT NULL                |
| updated_at        | INTEGER | NOT NULL                |
| last_synced_at    | INTEGER | NOT NULL                |

#### Table: `resources`
| Column          | Type    | Constraint              |
|-----------------|---------|-------------------------|
| resource_id     | TEXT    | PRIMARY KEY             |
| title           | TEXT    | NOT NULL                |
| description     | TEXT    | NOT NULL                |
| category        | TEXT    | NOT NULL                |
| subject         | TEXT    | NOT NULL                |
| grade           | TEXT    | NULLABLE                |
| file_url        | TEXT    | NOT NULL                |
| file_type       | TEXT    | NOT NULL                |
| uploaded_by     | TEXT    | NOT NULL                |
| approval_status | TEXT    | NOT NULL                |
| download_count  | INTEGER | NOT NULL                |
| sync_status     | TEXT    | NOT NULL                |
| created_at      | INTEGER | NOT NULL                |
| last_synced_at  | INTEGER | NOT NULL                |

#### Table: `notifications`
| Column          | Type    | Constraint          |
|-----------------|---------|---------------------|
| notification_id | TEXT    | PRIMARY KEY         |
| title           | TEXT    | NOT NULL            |
| body            | TEXT    | NOT NULL            |
| type            | TEXT    | NOT NULL            |
| linked_doc_id   | TEXT    | NULLABLE            |
| is_read         | INTEGER | NOT NULL (0/1)      |
| created_at      | INTEGER | NOT NULL            |
| last_synced_at  | INTEGER | NOT NULL            |

---

## 6. Navigation Architecture

### 6.1 Single Activity Pattern

`MainActivity` hosts a single `NavHostFragment` and a `BottomNavigationView`. Two root nav graphs handle auth vs main app routing.

### 6.2 Top-Level Nav Graph Structure

```
nav_root.xml
  ├── SplashFragment                (startDestination)
  │     ├── → nav_auth (not logged in)
  │     └── → nav_main (logged in)
  │
  ├── <include app:graph="@navigation/nav_auth" />
  └── <include app:graph="@navigation/nav_main" />
```

### 6.3 Auth Nav Graph (`nav_auth.xml`)

```
LoginFragment (startDestination)
  ├── → RegisterFragment
  │       └── → LoginFragment (pop back on success)
  ├── → ForgotPasswordFragment
  │       └── → LoginFragment (after reset email sent)
  └── → nav_main (on successful login, pop auth stack)
```

### 6.4 Main Nav Graph (`nav_main.xml`) — Bottom Navigation

```
Bottom Navigation Tabs:
  [Dashboard]  [Reports]  [Resources]  [AI]  [Notifications]

Each tab is a nested nav graph:

DashboardFragment (tab 1 — startDestination)
  └── → AdminDashboardFragment (role: ADMIN only)
        ├── → UserManagementFragment
        │       └── → UserDetailFragment
        ├── → ModerationFragment
        └── → AnalyticsFragment

ReportListFragment (tab 2)
  ├── → ReportDetailFragment
  │       └── → CreateReportFragment (edit mode)
  └── → CreateReportFragment (create mode)

ResourceHubFragment (tab 3)
  ├── → ResourceDetailFragment
  └── → UploadResourceFragment (teacher/admin only)

AiAssistantFragment (tab 4)  [standalone, no sub-navigation]

NotificationsFragment (tab 5)  [standalone, no sub-navigation]

─── Global Destinations (accessible from anywhere) ───
ProfileFragment
  ├── → EditProfileFragment
  └── → SettingsFragment
```

### 6.5 Role-Based Navigation Guard

A `NavController.OnDestinationChangedListener` in `MainActivity` checks the current user's role against a `PermissionMap`. Unauthorized access redirects to a `403 AccessDeniedFragment`.

---

## 7. Screen Inventory

### Auth Screens (4)

| # | Screen              | Description                                                     |
|---|---------------------|-----------------------------------------------------------------|
| 1 | Splash              | Logo animation, auth state check, route to Login or Dashboard   |
| 2 | Login               | Email + password, Google Sign-In option, forgot password link   |
| 3 | Register            | Name, email, password, role selection, school code entry        |
| 4 | Forgot Password     | Email input, sends Firebase reset link, confirmation UI         |

### Main App Screens (19)

| # | Screen              | Tab / Location   | Description                                                   |
|---|---------------------|------------------|---------------------------------------------------------------|
| 5 | Dashboard           | Tab 1            | Summary cards: pending reports, new resources, quick actions  |
| 6 | Report List         | Tab 2            | All user reports with filter (status, category, date)         |
| 7 | Report Detail       | Reports          | Full report view, status badge, comments thread               |
| 8 | Create Report       | Reports FAB      | Multi-step form: category → details → attachments → review    |
| 9 | Edit Report         | Report Detail    | Edit pre-submitted report (PENDING status only)               |
| 10| Resource Hub        | Tab 3            | Grid/list of resources, search + category filter              |
| 11| Resource Detail     | Resources        | Preview, download, metadata, share link                       |
| 12| Upload Resource     | Resources FAB    | Title, description, subject, grade, file picker, submit       |
| 13| AI Assistant        | Tab 4            | Chat-like UI, query input, rule-based response cards          |
| 14| Notifications       | Tab 5            | Chronological notification list with read/unread state        |
| 15| Profile             | Top Bar / Drawer | Avatar, name, role badge, stats (reports, uploads)            |
| 16| Edit Profile        | Profile          | Update name, photo, class, password change                    |
| 17| Settings            | Profile          | Dark mode toggle, notification prefs, language, logout        |

### Admin Panel Screens (4)

| # | Screen              | Access       | Description                                                       |
|---|---------------------|--------------|-------------------------------------------------------------------|
| 18| Admin Dashboard     | Admin only   | KPI tiles: total users, open reports, resource pending approvals  |
| 19| User Management     | Admin only   | Full user list, search, role change, deactivate account           |
| 20| Moderation          | Admin only   | Pending resources awaiting approval; flag/approve/reject reports  |
| 21| Analytics           | Admin only   | Line/bar charts: report trends, active users, resource stats      |

### Supporting Screens (2)

| # | Screen              | Description                                                      |
|---|---------------------|------------------------------------------------------------------|
| 22| User Detail (Admin) | Individual user profile viewed by admin; role override controls  |
| 23| Access Denied       | Shown when a non-admin tries to reach an admin-only destination  |

---

## 8. User Roles & Permissions

### 8.1 Role Definitions

| Role    | Description                                               |
|---------|-----------------------------------------------------------|
| STUDENT | Default role; can submit reports, browse resources        |
| TEACHER | Can upload resources, view reports from their class       |
| ADMIN   | Full access: user management, moderation, analytics, push |

### 8.2 Feature Permission Matrix

| Feature                      | STUDENT | TEACHER | ADMIN |
|------------------------------|:-------:|:-------:|:-----:|
| Login / Register             | ✅      | ✅      | ✅    |
| View Dashboard               | ✅      | ✅      | ✅    |
| Submit Report                | ✅      | ✅      | ✅    |
| View Own Reports             | ✅      | ✅      | ✅    |
| View All Reports             | ❌      | ✅      | ✅    |
| Resolve / Assign Reports     | ❌      | ✅      | ✅    |
| Browse Resources             | ✅      | ✅      | ✅    |
| Upload Resources             | ❌      | ✅      | ✅    |
| Approve Resources            | ❌      | ❌      | ✅    |
| Use AI Assistant             | ✅      | ✅      | ✅    |
| View Notifications           | ✅      | ✅      | ✅    |
| Send Announcements (FCM)     | ❌      | ❌      | ✅    |
| Manage Users                 | ❌      | ❌      | ✅    |
| View Analytics               | ❌      | ❌      | ✅    |
| Access Admin Panel           | ❌      | ❌      | ✅    |

---

## 9. User Stories

### 9.1 Authentication

```
US-AUTH-01  As a student, I want to register using my email and a school code
            so that only verified school members can access the app.
            Acceptance: School code validated against Firestore app_config.

US-AUTH-02  As any user, I want to log in with email/password
            so that I can access my personalized dashboard.
            Acceptance: Firebase Auth token stored, user doc fetched to Room.

US-AUTH-03  As any user, I want to reset my forgotten password via email
            so that I am never permanently locked out.
            Acceptance: Firebase send-reset-email called; confirmation shown.

US-AUTH-04  As any user, I want my session to persist across app restarts
            so that I don't have to log in every time.
            Acceptance: Auth state listener keeps user logged in until explicit logout.
```

### 9.2 Dashboard

```
US-DASH-01  As a student, I want to see my open reports count and recent
            notifications on my dashboard so I can quickly check my status.

US-DASH-02  As an admin, I want to see KPIs (total users, open reports,
            pending approvals) on the dashboard so I know the system's health.

US-DASH-03  As a teacher, I want to see reports assigned to me on the
            dashboard for quick access without navigating to the Reports tab.
```

### 9.3 Report Management

```
US-REP-01   As a student, I want to submit a report in multiple categories
            (incident, maintenance, bullying) so I can report any type of issue.
            Acceptance: Report saved to Firestore + Room; category auto-classified by AI.

US-REP-02   As a student, I want to submit a report anonymously
            so that I feel safe reporting sensitive incidents.
            Acceptance: `isAnonymous=true` hides submitter name in all teacher/admin views.

US-REP-03   As a student, I want to attach photos to my report
            so that evidence is clearly documented.
            Acceptance: Up to 5 images; uploaded to Firebase Storage; URLs stored in report.

US-REP-04   As a student, I want to track the status of my submitted report
            (Pending → In Review → Resolved) so I know it is being addressed.

US-REP-05   As a teacher, I want to add internal comments on a report
            so I can document resolution steps without students seeing them.
            Acceptance: `isInternal=true` comments visible only to TEACHER/ADMIN roles.

US-REP-06   As an admin, I want to assign a report to a specific teacher
            so that the right person is responsible for resolving it.

US-REP-07   As a user offline, I want to submit a report that syncs when
            I'm back online so that poor connectivity doesn't block me.
            Acceptance: Saved to Room with syncStatus=PENDING; WorkManager syncs on reconnect.
```

### 9.4 Resource Hub

```
US-RES-01   As a student, I want to browse educational resources filtered by
            subject and grade so I find relevant material quickly.

US-RES-02   As a student, I want to download a PDF resource for offline reading
            so I can study without internet access.

US-RES-03   As a teacher, I want to upload study notes and past papers for
            my students so they have centralized access to study materials.
            Acceptance: File uploaded to Firebase Storage; approval status set to PENDING.

US-RES-04   As an admin, I want to approve or reject uploaded resources before
            they go live so that only quality content is published.

US-RES-05   As any user, I want to search resources by keyword or tag
            so that I can find content without browsing every category.
```

### 9.5 AI Assistant

```
US-AI-01    As a student, I want to ask a question in plain text and receive
            a categorized, helpful response so I know where to go for help.
            Acceptance: Rule-based classifier matches keywords → predefined response set.

US-AI-02    As a user, I want the AI to suggest which report category to use
            when describing my issue so I don't choose the wrong category.
            Acceptance: AI response includes a "Submit Report" shortcut pre-filled with category.

US-AI-03    As a user, I want to ask FAQ-type questions (school timings, events,
            contacts) and receive instant answers without leaving the app.
```

### 9.6 Notifications

```
US-NOT-01   As a student, I want to receive a push notification when my report
            status changes so I am informed without opening the app.
            Acceptance: FCM triggered server-side on Firestore document update (Cloud Function).

US-NOT-02   As an admin, I want to send an announcement to all students, all
            teachers, or the entire school from the admin panel.
            Acceptance: FCM topic-based messaging used for broadcast notifications.

US-NOT-03   As any user, I want to see a badge count on the Notifications tab
            showing unread notifications so I don't miss updates.
```

### 9.7 Admin Panel

```
US-ADM-01   As an admin, I want to view all registered users, filter by role,
            and deactivate accounts if needed.

US-ADM-02   As an admin, I want to promote a student or teacher to admin role
            so that I can distribute administration responsibilities.

US-ADM-03   As an admin, I want to view analytics showing weekly report
            submissions by category so I can identify recurring problem areas.

US-ADM-04   As an admin, I want to see resource usage statistics (downloads per
            subject) so I understand what content students value most.
```

---

## 10. Development Roadmap

### Phase 0 — Project Setup (Days 1–3)

| Task                                              | Output                              |
|---------------------------------------------------|-------------------------------------|
| Initialize Android Studio project with Kotlin DSL | `build.gradle.kts` configured       |
| Set up Hilt, Navigation Component, ViewBinding    | DI graph functional                 |
| Configure Firebase project + `google-services.json`| Firebase SDK integrated            |
| Set up Material 3 theme (colors, typography, shapes)| `themes.xml` finalized             |
| Create base classes (BaseFragment, BaseViewModel) | Core scaffold ready                 |
| Set up Room database + DAO skeletons              | `NexusDatabase.kt` created          |
| Set up Firestore security rules draft             | Least-privilege rules in place      |

---

### Phase 1 — Authentication (Days 4–7)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| Firebase Auth integration (email/password)        | —                                   |
| Splash screen with auth state routing             | Splash (S1)                         |
| Login screen + ViewModel + use case               | Login (S2)                          |
| Register screen with school code validation       | Register (S3)                       |
| Forgot Password screen                            | Forgot Password (S4)                |
| User document creation on Firestore on register   | —                                   |
| DataStore persistence for session                 | —                                   |

**Milestone:** Any user can create an account, log in, and be routed to the app correctly.

---

### Phase 2 — Dashboard & Navigation Shell (Days 8–10)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| MainActivity with BottomNavigationView            | —                                   |
| NavController wired to bottom nav tabs            | —                                   |
| Dashboard screen with static summary cards        | Dashboard (S5)                      |
| Role-based navigation guard implemented           | Access Denied (S23)                 |
| Profile screen (read-only)                        | Profile (S15)                       |

**Milestone:** Bottom navigation works; all tabs load empty/stub screens.

---

### Phase 3 — Report Management (Days 11–17)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| Firestore `reports` collection + Room entity      | —                                   |
| ReportRepository (remote + local)                 | —                                   |
| Report List screen with filters                   | Report List (S6)                    |
| Create Report screen (multi-step form)            | Create Report (S8)                  |
| Report Detail screen with status badge            | Report Detail (S7)                  |
| Edit Report screen                                | Edit Report (S9)                    |
| Comment sub-collection (internal/public)          | —                                   |
| Firebase Storage image upload for attachments     | —                                   |
| Offline: save to Room with PENDING status         | —                                   |
| WorkManager sync job for pending reports          | —                                   |

**Milestone:** Full CRUD for reports; works offline; syncs on reconnect.

---

### Phase 4 — Resource Hub (Days 18–23)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| Firestore `resources` collection + Room entity    | —                                   |
| Firebase Storage integration for file uploads     | —                                   |
| Resource Hub screen with search + category filter | Resource Hub (S10)                  |
| Resource Detail screen with download button       | Resource Detail (S11)               |
| Upload Resource screen (teacher/admin)            | Upload Resource (S12)               |
| Resource approval flow (admin)                    | —                                   |

**Milestone:** Teachers upload resources; students browse and download; admins approve.

---

### Phase 5 — AI Assistant (Days 24–27)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| Rule-based classifier (keyword → intent map)      | —                                   |
| FAQ response dataset (JSON/Kotlin sealed class)   | —                                   |
| AI Assistant chat-like UI                         | AI Assistant (S13)                  |
| Category suggestion integration with Create Report| —                                   |
| Context-aware quick actions in AI responses       | —                                   |

**AI Classification Rule Map:**

```
"broken", "repair", "toilet", "light" → MAINTENANCE
"fight", "hit", "hurt", "abuse"       → INCIDENT
"bully", "tease", "harass"            → BULLYING
"grade", "marks", "exam", "teacher"   → ACADEMIC
"fee", "canteen", "timing"            → FAQ_GENERAL
```

**Milestone:** AI Assistant provides helpful rule-based responses and guides users to correct report categories.

---

### Phase 6 — Notifications (Days 28–31)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| FCM integration + token stored to Firestore       | —                                   |
| Notifications screen                              | Notifications (S14)                 |
| Notification listener service                     | —                                   |
| Unread badge count on bottom nav tab              | —                                   |
| Admin: send announcement to FCM topic             | —                                   |
| Report status change triggers FCM (Cloud Function)| —                                   |

**Milestone:** Users receive push notifications for report updates and admin announcements.

---

### Phase 7 — Admin Panel & Analytics (Days 32–38)

| Task                                              | Screens Completed                   |
|---------------------------------------------------|-------------------------------------|
| Admin Dashboard with KPI cards                   | Admin Dashboard (S18)               |
| User Management list + role change                | User Management (S19), User Detail (S22)|
| Moderation screen (approve/reject resources)      | Moderation (S20)                    |
| Analytics screen with MPAndroidChart              | Analytics (S21)                     |
| Event logging to `analytics_events` Firestore     | —                                   |
| Firestore security rules — admin scope            | —                                   |

**Milestone:** Admin panel fully functional; analytics charts showing real data.

---

### Phase 8 — Polish, Testing & Competition Prep (Days 39–45)

| Task                                              | Status Target               |
|---------------------------------------------------|-----------------------------|
| UI/UX review — Material 3 consistency audit       | All screens M3 compliant    |
| Lottie animations for empty states / success      | 5+ animations added         |
| Shimmer loading placeholders on all list screens  | Implemented                 |
| Edit Profile + Settings screens                   | S16, S17 completed          |
| Firebase Crashlytics integration                  | Crash reporting active      |
| Unit tests for all use cases                      | ≥ 80% domain coverage       |
| UI tests for auth + report creation flows         | Critical paths covered      |
| Performance: Firestore pagination (Paging 3)      | List screens paginated      |
| App icon + splash branding finalized              | Competition-ready branding  |
| APK release build + ProGuard rules                | Signed APK generated        |

---

## 11. Architectural Decision Records

### ADR-01: Single Activity Architecture
**Decision:** One `MainActivity` with `NavHostFragment`. No multi-activity.  
**Rationale:** Simplifies shared ViewModel scope, bottom nav state retention, and deep-link handling with Navigation Component.

### ADR-02: Clean Architecture Over Vanilla MVVM
**Decision:** Three distinct layers — Presentation, Domain, Data — with strict dependency rules (outer layers depend on inner, never the reverse).  
**Rationale:** Domain layer remains testable without Android or Firebase dependencies. Enables swapping Firebase for a different backend without touching use cases.

### ADR-03: Kotlin Flow Over LiveData for State
**Decision:** `StateFlow` for UI state, `SharedFlow` for one-time events (navigation, snackbars).  
**Rationale:** Flow is coroutine-native, composable, and handles lifecycle with `repeatOnLifecycle`. LiveData is kept only for legacy Room observation compatibility.

### ADR-04: Room as Offline Cache, Not Primary DB
**Decision:** Firestore is the source of truth; Room is a write-through cache.  
**Rationale:** Avoids data divergence. WorkManager reconciles Room PENDING records to Firestore on connectivity restore.

### ADR-05: Rule-Based AI, Not ML
**Decision:** AI Assistant uses a deterministic keyword-to-intent classifier (no ML model).  
**Rationale:** Avoids on-device model overhead; eliminates internet dependency for AI; fully auditable and competition-safe. Extensible to cloud ML post-competition.

### ADR-06: Hilt for Dependency Injection
**Decision:** Hilt over manual DI or Koin.  
**Rationale:** Official Google recommendation; compile-time verification; tight integration with ViewModel, WorkManager, and Navigation Component.

### ADR-07: Firebase Security Rules as First-Class Code
**Decision:** Firestore security rules version-controlled in `/firebase/firestore.rules`.  
**Rationale:** Role checks in rules (e.g., `request.auth.token.role == 'ADMIN'`) prevent unauthorized Firestore access even if client-side permission guards are bypassed.

---

*Document Version: 1.0 | Architecture: Clean Architecture + MVVM | Prepared for Yuvakarshan 2K26 — NEXUS*
