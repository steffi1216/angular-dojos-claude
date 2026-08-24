# Angular Dojo: runInInjectionContext & EnvironmentInjector
**Datum:** 2026-08-24
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `runInInjectionContext` und dem `EnvironmentInjector` Dependency Injection außerhalb von Klassen-Konstruktoren nutzt – ein essenzielles Muster für moderne funktionale Guards, Resolver und wiederverwendbare Utility-Funktionen.

## Hintergrund & Theorie

Seit Angular 14 ermöglicht `runInInjectionContext()` das Ausführen von Code in einem Injection-Kontext, ohne dass ein `@Injectable()`-Klassen-Konstruktor benötigt wird. Das ist die Grundlage für:

- **Functional Guards** und **Functional Resolvers** (seit Angular 15)
- **`inject()`-Aufrufe** außerhalb von Komponenten/Services
- Wiederverwendbare Helfer-Funktionen, die auf DI angewiesen sind

Der `EnvironmentInjector` ist der Injector auf App-Ebene (Root-Kontext). Mit ihm lässt sich `runInInjectionContext()` flexibel aus beliebigem Code aufrufen – z. B. in Utility-Funktionen, Factories oder Tests.

```typescript
// inject() ist nur in einem Injection-Kontext erlaubt
const value = inject(MyService); // ✅ im Konstruktor
// inject(MyService); // ❌ außerhalb → Fehler!

// Mit runInInjectionContext geht es überall:
injector.runInInjectionContext(() => {
  const svc = inject(MyService); // ✅
});
```

`runInInjectionContext` ist auch die Basis von `toSignal()`, `takeUntilDestroyed()` und anderen modernen APIs, die intern `inject()` nutzen.

## Aufgabe

Erstelle eine wiederverwendbare **Factory-Funktion** `createCachingLoader<T>()`, die:

1. Einen `HttpClient` und einen `CACHE_TTL`-Token per `inject()` bezieht
2. Eine Funktion zurückgibt, die eine URL lädt, das Ergebnis in einem `Map`-Cache hält und abgelaufene Einträge invalidiert
3. In einem funktionalen Guard genutzt wird, der prüft, ob eine Konfiguration geladen werden kann

### Schritte

1. Definiere einen `InjectionToken<number>` namens `CACHE_TTL` (Standard: 30 000 ms) und stelle ihn in `app.config.ts` bereit.
2. Schreibe die Funktion `createCachingLoader<T>(url: string)` – sie soll intern `inject(HttpClient)` und `inject(CACHE_TTL)` aufrufen (also im Injection-Kontext nutzbar sein).
3. Nutze `runInInjectionContext` mit dem `EnvironmentInjector`, um `createCachingLoader` aus einem Service heraus aufzurufen, der selbst keinen direkten Konstruktor-Zugriff auf `HttpClient` haben soll.
4. Schreibe einen funktionalen Route-Guard `configLoadedGuard`, der den Loader nutzt und `true` zurückgibt, sobald die Konfiguration erfolgreich geladen wurde.
5. Binde den Guard in `app.routes.ts` an eine Route.

## Hints

<details>
<summary>Hint 1 – InjectionToken und Bereitstellung</summary>

```typescript
// injection-tokens.ts
import { InjectionToken } from '@angular/core';

export const CACHE_TTL = new InjectionToken<number>('CACHE_TTL', {
  providedIn: 'root',
  factory: () => 30_000,
});

// app.config.ts – optionale Überschreibung:
{ provide: CACHE_TTL, useValue: 60_000 }
```
</details>

<details>
<summary>Hint 2 – runInInjectionContext im Service</summary>

```typescript
import { inject, Injectable, EnvironmentInjector, runInInjectionContext } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoaderFactoryService {
  private envInjector = inject(EnvironmentInjector);

  create<T>(url: string) {
    // createCachingLoader nutzt intern inject() – daher braucht es den Kontext:
    return runInInjectionContext(this.envInjector, () => createCachingLoader<T>(url));
  }
}
```
</details>

<details>
<summary>Hint 3 – Funktionaler Guard</summary>

```typescript
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { map, catchError, of } from 'rxjs';

export const configLoadedGuard: CanActivateFn = () => {
  const factory = inject(LoaderFactoryService);
  const load = factory.create<AppConfig>('/assets/config.json');
  return load().pipe(
    map(() => true),
    catchError(() => of(false)),
  );
};
```
</details>

## Beispiellösung

```typescript
// injection-tokens.ts
import { InjectionToken } from '@angular/core';

export const CACHE_TTL = new InjectionToken<number>('CACHE_TTL', {
  providedIn: 'root',
  factory: () => 30_000,
});

// caching-loader.ts
import { inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, of, tap } from 'rxjs';
import { CACHE_TTL } from './injection-tokens';

interface CacheEntry<T> {
  value: T;
  expiresAt: number;
}

export function createCachingLoader<T>(url: string): () => Observable<T> {
  // inject() ist hier erlaubt, weil die Funktion im Injection-Kontext aufgerufen wird
  const http = inject(HttpClient);
  const ttl = inject(CACHE_TTL);
  const cache = new Map<string, CacheEntry<T>>();

  return () => {
    const entry = cache.get(url);
    if (entry && entry.expiresAt > Date.now()) {
      return of(entry.value);
    }
    return http.get<T>(url).pipe(
      tap(value => cache.set(url, { value, expiresAt: Date.now() + ttl })),
    );
  };
}

// loader-factory.service.ts
import { inject, Injectable, EnvironmentInjector, runInInjectionContext } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class LoaderFactoryService {
  private envInjector = inject(EnvironmentInjector);

  create<T>(url: string): () => Observable<T> {
    return runInInjectionContext(this.envInjector, () => createCachingLoader<T>(url));
  }
}

// config.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn } from '@angular/router';
import { map, catchError, of } from 'rxjs';
import { LoaderFactoryService } from './loader-factory.service';

export interface AppConfig {
  apiUrl: string;
  featureFlags: Record<string, boolean>;
}

export const configLoadedGuard: CanActivateFn = () => {
  const factory = inject(LoaderFactoryService);
  const loadConfig = factory.create<AppConfig>('/assets/config.json');
  return loadConfig().pipe(
    map(() => true),
    catchError(() => {
      console.error('Config konnte nicht geladen werden');
      return of(false);
    }),
  );
};

// app.routes.ts
import { Routes } from '@angular/router';
import { configLoadedGuard } from './config.guard';

export const routes: Routes = [
  {
    path: 'dashboard',
    canActivate: [configLoadedGuard],
    loadComponent: () => import('./dashboard/dashboard.component'),
  },
];
```

## Weiterführendes
- Die offizielle Angular-Doku zu [Injection context](https://angular.dev/guide/di/dependency-injection-context) erklärt alle Stellen, an denen `inject()` nativ erlaubt ist.
- `runInInjectionContext` ist auch nützlich, um `effect()` oder `toSignal()` in nicht-reaktiven Kontexten zu nutzen – kombiniere es mit `DestroyRef` für automatisches Cleanup.
- Schau dir an, wie Angular selbst `runInInjectionContext` intern in `Router`-Guards, `APP_INITIALIZER` und `HttpInterceptorFn` einsetzt (Quellcode auf GitHub).
