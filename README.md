# Flutter GetX Architecture Guide

A practical, opinionated reference for building Flutter apps with **GetX** — covering folder structure, state management, routing, dependency injection, networking, theming, localization, and feature scaffolding.

Designed to be copied into any new Flutter project as the baseline architecture.

## What's inside

- **Per-feature layered structure** — `presentation/` + `domain/` + `data/` + `binding/`.
- **GetX state management** — `GetxController`, `Rx`/`Obx`, `GetBuilder` with scoped IDs.
- **Named routing** — `GetMaterialApp` + `AppRoutes` constants + `AppNavigator.routesList` + `AppGo` wrapper for custom transitions.
- **Bindings & DI** — `CoreBinding` for global services, per-feature bindings for data source → repo → controller.
- **Networking** — single Dio-based `NetworkAdapter`, typed `NetworkRequest`, endpoint enum `NetworkRouter`, `Either<Failure, T>` returns via `dartz`.
- **Theme system** — `AppTheme`, `ColorManager`, `AppTypography`, light/dark, RTL-aware.
- **Localization** — GetX `Translations`, `LocalizationKeys`, language registry, RTL handling.
- **Optional flavors** — `flutter_flavor` + `envied`, cleanly separated so you can drop them.
- **Naming conventions, do/don't, and a step-by-step new feature checklist.**

## Read the guide

[ARCHITECTURE.md](./ARCHITECTURE.md)

## Tech stack the guide assumes

- Flutter (Dart `>=3.3.4 <4.0.0`)
- [`get`](https://pub.dev/packages/get) — state management + routing
- [`dio`](https://pub.dev/packages/dio) — networking
- [`dartz`](https://pub.dev/packages/dartz) — `Either<Failure, T>`
- [`shared_preferences`](https://pub.dev/packages/shared_preferences) + [`flutter_secure_storage`](https://pub.dev/packages/flutter_secure_storage) — storage
- Optional: [`flutter_flavor`](https://pub.dev/packages/flutter_flavor), [`envied`](https://pub.dev/packages/envied), [`flutter_hooks`](https://pub.dev/packages/flutter_hooks)

## Who this is for

- Teams that want a consistent architecture across multiple Flutter projects.
- Developers starting a new Flutter app and looking for a battle-tested layout.
- Developers joining an existing project that follows this pattern and need a single reference.

## How to use it

1. Read [ARCHITECTURE.md](./ARCHITECTURE.md) end-to-end once.
2. When starting a new project, follow **section 19 — New Project Bootstrap Guide**.
3. When adding a new feature, follow **section 17 — Adding a New Feature: Checklist**.
4. Replace `App` / `<Brand>` placeholders with your project's prefix (e.g. `Falcon*` widgets, `FalconRoutes`, etc.).

## License

MIT — feel free to use, modify, and adapt for any project.
