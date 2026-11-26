# Angular Dojo: `provideHttpClient` – Funktionale HTTP-Konfiguration
**Datum:** 2026-11-26
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die funktionale `provideHttpClient`-API (Angular 15+) kennen und weißt, wie du HTTP-Features wie `withFetch`, `withInterceptors`, `withRequestsMadeViaParent` und `withNoXsrfProtection` gezielt konfigurierst.

## Hintergrund & Theorie

Seit Angular 15 ersetzt `provideHttpClient()` das klassische `HttpClientModule`. Die neue API folgt dem funktionalen Provider-Pattern und erlaubt feingranulare Konfiguration über sogenannte **HTTP-Features** (`HttpFeature`).

**Wichtigste Features:**

| Feature | Beschreibung |
|---|---|
| `withFetch()` | Nutzt `fetch` statt `XMLHttpRequest` als Transport |
| `withInterceptors([...])` | Registriert funktionale Interceptors (kein `Injectable`) |
| `withInterceptorsFromDi()` | Bindet klassische DI-basierte Interceptors ein |
| `withRequestsMadeViaParent()` | Requests werden über den Eltern-Injector geleitet (für lazy features) |
| `withNoXsrfProtection()` | Deaktiviert den automatischen XSRF-Schutz |
| `withJsonpSupport()` | Aktiviert JSONP-Requests |
| `withXsrfConfiguration({...})` | Passt Cookie-/Header-Namen für XSRF an |

Funktionale Interceptors sind einfache Funktionen mit der Signatur `(req, next) => Observable<HttpEvent<unknown>>`. Sie lassen sich leicht testen, direkt mit `inject()` in ihrem Kontext nutzen und müssen nicht als `Injectable`-Klasse deklariert werden.

**Kombination in `app.config.ts`:**
```typescript
provideHttpClient(
  withFetch(),
  withInterceptors([authInterceptor, loggingInterceptor]),
)
```

## Aufgabe

Baue eine Mini-Applikation mit zwei Lazy-Loaded Features (`admin` und `public`), die **unterschiedliche HTTP-Interceptors** verwenden. Der `admin`-Bereich soll einen Auth-Token-Interceptor nutzen; der `public`-Bereich bleibt ohne Auth. Beide nutzen jedoch einen gemeinsamen Logging-Interceptor aus dem Root-Injector.

### Schritte

1. **Root-Konfiguration** (`app.config.ts`): Registriere `provideHttpClient(withFetch(), withInterceptors([loggingInterceptor]))` auf Root-Ebene.

2. **Logging-Interceptor erstellen** (`logging.interceptor.ts`): Schreibe eine funktionale Interceptor-Funktion, die Start und Ende jedes Requests mit `console.log` protokolliert (URL + Dauer in ms).

3. **Auth-Interceptor erstellen** (`auth.interceptor.ts`): Schreibe einen Interceptor, der einen `Authorization: Bearer <token>`-Header anfügt. Lese den Token aus einem `AuthService` via `inject()`.

4. **Admin-Feature-Providers** (`admin/admin.config.ts`): Konfiguriere für das Admin-Feature `provideHttpClient(withInterceptors([authInterceptor]), withRequestsMadeViaParent())`. Das `withRequestsMadeViaParent()` sorgt dafür, dass der Root-Logging-Interceptor **ebenfalls** aktiv bleibt.

5. **Demo-Komponente**: Erstelle eine Komponente, die einen HTTP-Request an `https://jsonplaceholder.typicode.com/todos/1` sendet und das Ergebnis in einem Signal speichert. Nutze die Komponente einmal im Admin- und einmal im Public-Kontext und beobachte die unterschiedlichen Request-Header in den Browser-DevTools.

## Hints

<details>
<summary>Hint 1 – Funktionale Interceptor-Signatur</summary>

```typescript
import { HttpInterceptorFn } from '@angular/common/http';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const start = Date.now();
  console.log(`[HTTP] --> ${req.url}`);
  return next(req).pipe(
    tap(() => console.log(`[HTTP] <-- ${req.url} (${Date.now() - start}ms)`))
  );
};
```

`next` gibt ein `Observable<HttpEvent<unknown>>` zurück. Du kannst `pipe` direkt darauf anwenden.
</details>

<details>
<summary>Hint 2 – inject() im Interceptor-Kontext</summary>

Funktionale Interceptors werden im Injection-Context ausgeführt, deshalb kannst du `inject()` direkt im Funktionskörper verwenden:

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.token();          // z.B. ein Signal
  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(authReq);
};
```

Achte darauf, dass `inject()` nur synchron im Funktionsrumpf, **nicht** innerhalb eines Callbacks oder Operators, aufgerufen werden darf.
</details>

<details>
<summary>Hint 3 – withRequestsMadeViaParent()</summary>

Ohne `withRequestsMadeViaParent()` wird für das Feature ein **eigener** `HttpClient`-Handler erstellt, der Root-Interceptors **nicht** ausführt. Mit diesem Feature werden Requests an den Parent-Injector weitergeleitet, sodass Root-Interceptors zusätzlich greifen:

```typescript
// admin/admin.config.ts
export const adminProviders: Provider[] = [
  provideHttpClient(
    withInterceptors([authInterceptor]),
    withRequestsMadeViaParent(),   // ← Logging-Interceptor aus Root bleibt aktiv
  ),
];
```

Die Reihenfolge der Interceptors ist: zuerst die Feature-Interceptors, dann die Root-Interceptors.
</details>

## Beispiellösung

```typescript
// logging.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { tap } from 'rxjs/operators';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const start = Date.now();
  console.log(`[HTTP] --> ${req.method} ${req.url}`);
  return next(req).pipe(
    tap({
      next: event => {
        if (event.type === 4 /* Response */) {
          console.log(`[HTTP] <-- ${req.url} (${Date.now() - start}ms)`);
        }
      },
    })
  );
};

// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  return next(
    token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req
  );
};

// app.config.ts
import { provideHttpClient, withFetch, withInterceptors } from '@angular/common/http';
import { loggingInterceptor } from './logging.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withFetch(),
      withInterceptors([loggingInterceptor]),
    ),
    provideRouter(routes),
  ],
};

// admin/admin.config.ts
import { provideHttpClient, withInterceptors, withRequestsMadeViaParent } from '@angular/common/http';
import { authInterceptor } from '../auth.interceptor';

export const adminProviders = [
  provideHttpClient(
    withInterceptors([authInterceptor]),
    withRequestsMadeViaParent(),
  ),
];

// admin/admin.routes.ts
export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    providers: adminProviders,
    component: AdminDashboardComponent,
  },
];

// demo.component.ts
@Component({
  standalone: true,
  imports: [AsyncPipe],
  template: `
    @if (todo()) {
      <pre>{{ todo() | json }}</pre>
    } @else {
      <p>Laden...</p>
    }
  `,
})
export class DemoComponent {
  private http = inject(HttpClient);
  todo = signal<unknown>(null);

  constructor() {
    this.http.get('https://jsonplaceholder.typicode.com/todos/1')
      .pipe(takeUntilDestroyed())
      .subscribe(data => this.todo.set(data));
  }
}
```

## Weiterführendes

- **Offizielles Guide:** [angular.dev – HTTP Client](https://angular.dev/guide/http) – vollständige Dokumentation zu allen `withXxx`-Features
- **Tipp:** Mit `HttpTestingController` (aus `provideHttpClientTesting()`) lassen sich funktionale Interceptors genauso einfach testen wie klassische; `withInterceptors([myInterceptor])` wird in der Testumgebung automatisch berücksichtigt.
- **Weiterdenken:** Kombiniere `withRequestsMadeViaParent()` mit einem Feature-spezifischen Retry-Interceptor, um unterschiedliche Fehlerbehandlungsstrategien pro Route zu implementieren – ohne globale State-Konflikte.
