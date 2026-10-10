# Angular Dojo: HTTP Caching Strategies mit Interceptors
**Datum:** 2026-10-10
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du implementierst einen HTTP-Caching-Interceptor, der verschiedene Caching-Strategien (Cache-First, Stale-While-Revalidate) unterstützt und lernst, wie du den Cache gezielt per `HttpContext` steuerst.

## Hintergrund & Theorie
Angular's `HttpClient` bietet keinen eingebauten Cache-Mechanismus. Das bedeutet: Jede Anfrage trifft den Server – selbst bei unveränderten Daten. Ein **HTTP-Caching-Interceptor** setzt sich transparent zwischen Anfragen und das Netzwerk.

Zwei wichtige Strategien:

- **Cache-First**: Gibt sofort den gecachten Wert zurück (wenn vorhanden), ohne den Server zu kontaktieren. Ideal für statische Stammdaten (Länder, Kategorien).
- **Stale-While-Revalidate (SWR)**: Gibt sofort den gecachten (möglicherweise veralteten) Wert zurück, feuert gleichzeitig eine Netzwerkanfrage und aktualisiert den Cache im Hintergrund. Nutzer sehen keine Ladezeit, erhalten aber zeitnah frische Daten.

Die Strategie pro Request wird über `HttpContext` – mit einem `HttpContextToken` – konfiguriert, sodass der Interceptor sich sauber in den bestehenden Code integriert, ohne `HttpClient`-Aufrufe zu verändern.

Ab Angular 15+ schreibt man Interceptors als einfache Funktionen (`HttpInterceptorFn`), die sich mit `withInterceptors()` in `provideHttpClient()` registrieren lassen.

## Aufgabe
Implementiere einen funktionalen `cachingInterceptor` mit folgenden Anforderungen:

1. Definiere zwei `HttpContextToken`s:
   - `CACHE_STRATEGY`: `'cache-first' | 'swr' | 'none'` (Default: `'none'`)
   - `CACHE_TTL`: `number` (Millisekunden, Default: `60_000`)

2. Nutze ein `Map<string, { data: unknown; timestamp: number }>` als In-Memory-Cache (als `Map` in einem Service oder als Modul-Level-Variable).

3. Implementiere die Logik im Interceptor:
   - Bei `'none'`: Anfrage unverändert durchleiten.
   - Bei `'cache-first'`: Wenn Cache-Eintrag existiert und nicht abgelaufen ist (TTL), gibt den Wert als `of(cachedResponse)` zurück. Sonst: Anfrage ausführen, Antwort cachen.
   - Bei `'swr'`: Wenn Cache-Eintrag existiert (egal ob abgelaufen), gibt ihn sofort zurück. Gleichzeitig Netzwerkanfrage starten und Cache aktualisieren (im Hintergrund via `tap`). Wenn kein Cache-Eintrag: Anfrage ausführen und Eintrag erstellen.

4. Baue eine Demo-Komponente, die denselben Endpoint zweimal aufruft – einmal mit `cache-first` und einmal mit `swr` – und zeige an, ob die Daten aus dem Cache oder dem Netzwerk kamen.

### Schritte
1. `HttpContextToken`s definieren und einen `CacheService` (oder Modul-Variable) für den In-Memory-Cache anlegen.
2. `cachingInterceptor`-Funktion implementieren mit der Strategie-Logik.
3. Interceptor in `app.config.ts` via `provideHttpClient(withInterceptors([cachingInterceptor]))` registrieren.
4. Demo-Komponente erstellen: Zwei Buttons triggern denselben GET-Request, Logs zeigen `[CACHE HIT]` vs. `[NETWORK]`.
5. TTL testen: Nach Ablauf der TTL soll eine `cache-first`-Anfrage wieder ans Netzwerk gehen.

## Hints

<details>
<summary>Hint 1 – HttpContextToken und Cache-Key</summary>

```typescript
import { HttpContextToken } from '@angular/common/http';

export const CACHE_STRATEGY = new HttpContextToken<'cache-first' | 'swr' | 'none'>(
  () => 'none'
);
export const CACHE_TTL = new HttpContextToken<number>(() => 60_000);

// Cache-Key: URL + serialisierte Params
function getCacheKey(req: HttpRequest<unknown>): string {
  return `${req.method}:${req.urlWithParams}`;
}
```

</details>

<details>
<summary>Hint 2 – Cache-First-Logik im Interceptor</summary>

```typescript
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { of } from 'rxjs';
import { tap } from 'rxjs/operators';

const cache = new Map<string, { response: HttpResponse<unknown>; timestamp: number }>();

export const cachingInterceptor: HttpInterceptorFn = (req, next) => {
  const strategy = req.context.get(CACHE_STRATEGY);
  const ttl = req.context.get(CACHE_TTL);

  if (strategy === 'none' || req.method !== 'GET') {
    return next(req);
  }

  const key = getCacheKey(req);
  const cached = cache.get(key);
  const now = Date.now();

  if (strategy === 'cache-first') {
    if (cached && now - cached.timestamp < ttl) {
      console.log('[CACHE HIT]', key);
      return of(cached.response);
    }
    return next(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          cache.set(key, { response: event, timestamp: Date.now() });
          console.log('[NETWORK]', key);
        }
      })
    );
  }

  // 'swr'
  if (cached) {
    console.log('[CACHE HIT – revalidating]', key);
    // Hintergrundaktualisierung
    next(req).pipe(
      tap(event => {
        if (event instanceof HttpResponse) {
          cache.set(key, { response: event, timestamp: Date.now() });
        }
      })
    ).subscribe();
    return of(cached.response);
  }

  return next(req).pipe(
    tap(event => {
      if (event instanceof HttpResponse) {
        cache.set(key, { response: event, timestamp: Date.now() });
        console.log('[NETWORK – primed cache]', key);
      }
    })
  );
};
```

</details>

<details>
<summary>Hint 3 – Verwendung im Service</summary>

```typescript
import { HttpContext } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class PostsService {
  private http = inject(HttpClient);

  getPostsCacheFirst() {
    return this.http.get('https://jsonplaceholder.typicode.com/posts', {
      context: new HttpContext()
        .set(CACHE_STRATEGY, 'cache-first')
        .set(CACHE_TTL, 30_000)
    });
  }

  getPostsSWR() {
    return this.http.get('https://jsonplaceholder.typicode.com/posts', {
      context: new HttpContext().set(CACHE_STRATEGY, 'swr')
    });
  }
}
```

</details>

## Beispiellösung

```typescript
// cache-tokens.ts
import { HttpContextToken } from '@angular/common/http';

export const CACHE_STRATEGY = new HttpContextToken<'cache-first' | 'swr' | 'none'>(
  () => 'none'
);
export const CACHE_TTL = new HttpContextToken<number>(() => 60_000);

// caching.interceptor.ts
import { HttpInterceptorFn, HttpRequest, HttpResponse } from '@angular/common/http';
import { of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { CACHE_STRATEGY, CACHE_TTL } from './cache-tokens';

interface CacheEntry {
  response: HttpResponse<unknown>;
  timestamp: number;
}

const cache = new Map<string, CacheEntry>();

function key(req: HttpRequest<unknown>): string {
  return `${req.method}:${req.urlWithParams}`;
}

export const cachingInterceptor: HttpInterceptorFn = (req, next) => {
  const strategy = req.context.get(CACHE_STRATEGY);
  const ttl = req.context.get(CACHE_TTL);

  if (req.method !== 'GET' || strategy === 'none') return next(req);

  const cacheKey = key(req);
  const entry = cache.get(cacheKey);
  const now = Date.now();

  if (strategy === 'cache-first') {
    if (entry && now - entry.timestamp < ttl) {
      console.log(`[CACHE HIT] ${cacheKey}`);
      return of(entry.response);
    }
    return next(req).pipe(
      tap(e => {
        if (e instanceof HttpResponse) {
          cache.set(cacheKey, { response: e, timestamp: Date.now() });
          console.log(`[NETWORK] ${cacheKey}`);
        }
      })
    );
  }

  // Stale-While-Revalidate
  if (entry) {
    console.log(`[SWR – stale hit, revalidating] ${cacheKey}`);
    next(req).pipe(
      tap(e => {
        if (e instanceof HttpResponse)
          cache.set(cacheKey, { response: e, timestamp: Date.now() });
      })
    ).subscribe();
    return of(entry.response);
  }

  return next(req).pipe(
    tap(e => {
      if (e instanceof HttpResponse) {
        cache.set(cacheKey, { response: e, timestamp: Date.now() });
        console.log(`[NETWORK – cache primed] ${cacheKey}`);
      }
    })
  );
};

// app.config.ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { cachingInterceptor } from './caching.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([cachingInterceptor]))
  ]
};

// posts.service.ts
import { HttpClient, HttpContext } from '@angular/common/http';
import { inject, Injectable } from '@angular/core';
import { CACHE_STRATEGY, CACHE_TTL } from './cache-tokens';

@Injectable({ providedIn: 'root' })
export class PostsService {
  private http = inject(HttpClient);
  private url = 'https://jsonplaceholder.typicode.com/posts?_limit=5';

  getCacheFirst() {
    return this.http.get(this.url, {
      context: new HttpContext()
        .set(CACHE_STRATEGY, 'cache-first')
        .set(CACHE_TTL, 30_000)
    });
  }

  getSWR() {
    return this.http.get(this.url, {
      context: new HttpContext().set(CACHE_STRATEGY, 'swr')
    });
  }
}

// demo.component.ts
import { Component, inject } from '@angular/core';
import { PostsService } from './posts.service';

@Component({
  selector: 'app-demo',
  standalone: true,
  template: `
    <button (click)="loadCacheFirst()">Cache-First laden</button>
    <button (click)="loadSWR()">SWR laden</button>
    <pre>{{ result | json }}</pre>
  `
})
export class DemoComponent {
  private posts = inject(PostsService);
  result: unknown;

  loadCacheFirst() {
    this.posts.getCacheFirst().subscribe(data => (this.result = data));
  }

  loadSWR() {
    this.posts.getSWR().subscribe(data => (this.result = data));
  }
}
```

## Weiterführendes
- Für Produktionseinsatz: Erweitere den Cache um **LRU-Eviction** (z. B. maximale Cache-Größe) und persistiere ihn optional via `localStorage` für Offline-Szenarien.
- Angular's `HttpResource` API (Angular 19+) bietet nativ Caching-Überlegungen mit `stale`-Strategien – vergleiche deinen Interceptor-Ansatz damit.
- RFC 7234 (HTTP Caching) definiert `Cache-Control`-Header; ein produktionsreifer Interceptor sollte `max-age`, `no-cache` und `ETag`/`If-None-Match` aus den Response-Headern auslesen und respektieren.
