# Angular Dojo: Global Error Handling
**Datum:** 2026-08-02
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du baust eine robuste, zentrale Fehlerbehandlungs-Strategie für eine Angular-App: globaler `ErrorHandler`, HTTP-Fehler mit Interceptor und reaktive Fehlerstreams mit RxJS – alles sauber getrennt und testbar.

## Hintergrund & Theorie

Angular bietet drei Ebenen der Fehlerbehandlung:

**1. Globaler `ErrorHandler`**
Die Klasse `ErrorHandler` aus `@angular/core` fängt alle unbehandelten Fehler der Anwendung ab (Laufzeitfehler in Templates, unbehandelte Promises, etc.). Durch das Ersetzen des Standard-Providers mit `{provide: ErrorHandler, useClass: MyErrorHandler}` lassen sich Fehler zentral loggen oder an einen Monitoring-Dienst (z. B. Sentry) weiterleiten.

**2. HTTP-Fehlerbehandlung mit Interceptors**
`HttpInterceptorFn` (funktionaler Ansatz) erlaubt, HTTP-Fehler global abzufangen, Statuscodes zu klassifizieren (401 → Redirect zu Login, 5xx → Toast-Meldung) und optional Requests zu wiederholen.

**3. Reaktive Fehlerstreams**
In Services können Fehler über ein `Subject<AppError>` als Observable veröffentlicht werden. Komponenten subscriben darauf und zeigen nutzerspezifische Meldungen an – entkoppelt und testbar.

Das Zusammenspiel dieser drei Ebenen ergibt eine vollständige Fehlerarchitektur ohne redundanten try/catch-Boilerplate in jedem Service.

## Aufgabe

Implementiere eine zentrale Fehlerbehandlungs-Architektur:

1. Einen globalen `AppErrorHandler` (ersetzt Angulars Standard-`ErrorHandler`)
2. Einen funktionalen HTTP-Interceptor `errorInterceptor`, der 4xx/5xx klassifiziert
3. Einen `ErrorNotificationService` mit einem reaktiven Fehlerstream als Signal
4. Eine `ErrorBannerComponent`, die den aktuellen Fehler als Alert anzeigt

### Schritte

1. **`AppErrorHandler` erstellen** – implementiert `ErrorHandler`, loggt den Fehler und pusht ihn in den `ErrorNotificationService`
2. **`ErrorNotificationService` erstellen** – hält einen `BehaviorSubject<AppError | null>` und exponiert ihn als `readonly Signal` via `toSignal()`; bietet `report(error)` und `clear()` Methoden
3. **`errorInterceptor` erstellen** – funktionaler Interceptor (`HttpInterceptorFn`), der auf `HttpErrorResponse` reagiert; unterscheidet 401, 403, 404, 5xx und ruft `ErrorNotificationService.report()` auf
4. **`ErrorBannerComponent` erstellen** – liest den aktuellen Fehler über das Signal, zeigt einen dismissierbaren Alert an (Template Control Flow `@if`)
5. **Alles in `app.config.ts` registrieren** – `provideHttpClient(withInterceptors([errorInterceptor]))`, `{provide: ErrorHandler, useClass: AppErrorHandler}`

## Hints

<details>
<summary>Hint 1 – AppErrorHandler Grundstruktur</summary>

```typescript
import { ErrorHandler, inject, Injectable } from '@angular/core';
import { ErrorNotificationService } from './error-notification.service';

@Injectable()
export class AppErrorHandler implements ErrorHandler {
  private errorService = inject(ErrorNotificationService);

  handleError(error: unknown): void {
    console.error('[AppErrorHandler]', error);
    const message = error instanceof Error ? error.message : String(error);
    this.errorService.report({ message, source: 'runtime' });
  }
}
```

</details>

<details>
<summary>Hint 2 – Funktionaler HTTP-Interceptor</summary>

```typescript
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const errorService = inject(ErrorNotificationService);
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      const message = classifyHttpError(err);
      errorService.report({ message, source: 'http', status: err.status });
      return throwError(() => err);
    })
  );
};

function classifyHttpError(err: HttpErrorResponse): string {
  if (err.status === 401) return 'Nicht authentifiziert – bitte einloggen.';
  if (err.status === 403) return 'Zugriff verweigert.';
  if (err.status === 404) return 'Ressource nicht gefunden.';
  if (err.status >= 500) return 'Serverfehler – bitte später erneut versuchen.';
  return `Unbekannter Fehler (${err.status})`;
}
```

</details>

<details>
<summary>Hint 3 – ErrorNotificationService mit Signal</summary>

```typescript
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { toSignal } from '@angular/core/rxjs-interop';

export interface AppError {
  message: string;
  source: 'runtime' | 'http';
  status?: number;
}

@Injectable({ providedIn: 'root' })
export class ErrorNotificationService {
  private readonly _error$ = new BehaviorSubject<AppError | null>(null);
  readonly currentError = toSignal(this._error$, { initialValue: null });

  report(error: AppError): void {
    this._error$.next(error);
  }

  clear(): void {
    this._error$.next(null);
  }
}
```

</details>

## Beispiellösung

```typescript
// error-notification.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { toSignal } from '@angular/core/rxjs-interop';

export interface AppError {
  message: string;
  source: 'runtime' | 'http';
  status?: number;
}

@Injectable({ providedIn: 'root' })
export class ErrorNotificationService {
  private readonly _error$ = new BehaviorSubject<AppError | null>(null);
  readonly currentError = toSignal(this._error$, { initialValue: null });

  report(error: AppError): void {
    this._error$.next(error);
  }

  clear(): void {
    this._error$.next(null);
  }
}

// app-error-handler.ts
import { ErrorHandler, inject, Injectable } from '@angular/core';
import { ErrorNotificationService } from './error-notification.service';

@Injectable()
export class AppErrorHandler implements ErrorHandler {
  private errorService = inject(ErrorNotificationService);

  handleError(error: unknown): void {
    console.error('[AppErrorHandler]', error);
    const message = error instanceof Error ? error.message : String(error);
    this.errorService.report({ message, source: 'runtime' });
  }
}

// error.interceptor.ts
import { HttpInterceptorFn, HttpErrorResponse } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, throwError } from 'rxjs';
import { ErrorNotificationService } from './error-notification.service';

export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const errorService = inject(ErrorNotificationService);
  return next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err instanceof HttpErrorResponse) {
        const message = classifyHttpError(err);
        errorService.report({ message, source: 'http', status: err.status });
      }
      return throwError(() => err);
    })
  );
};

function classifyHttpError(err: HttpErrorResponse): string {
  if (err.status === 401) return 'Nicht authentifiziert – bitte einloggen.';
  if (err.status === 403) return 'Zugriff verweigert.';
  if (err.status === 404) return 'Ressource nicht gefunden.';
  if (err.status >= 500) return 'Serverfehler – bitte später erneut versuchen.';
  return `HTTP-Fehler ${err.status}`;
}

// error-banner.component.ts
import { Component, inject } from '@angular/core';
import { ErrorNotificationService } from './error-notification.service';

@Component({
  selector: 'app-error-banner',
  standalone: true,
  template: `
    @if (errorService.currentError(); as error) {
      <div class="error-banner" role="alert">
        <span>{{ error.message }}</span>
        @if (error.status) {
          <small> (Status {{ error.status }})</small>
        }
        <button (click)="errorService.clear()" aria-label="Schließen">✕</button>
      </div>
    }
  `,
  styles: [`
    .error-banner {
      display: flex;
      align-items: center;
      gap: 1rem;
      padding: 0.75rem 1rem;
      background: #fee2e2;
      border-left: 4px solid #ef4444;
      color: #991b1b;
    }
    button { margin-left: auto; cursor: pointer; border: none; background: none; }
  `],
})
export class ErrorBannerComponent {
  protected errorService = inject(ErrorNotificationService);
}

// app.config.ts
import { ApplicationConfig, ErrorHandler } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { AppErrorHandler } from './app-error-handler';
import { errorInterceptor } from './error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([errorInterceptor])),
    { provide: ErrorHandler, useClass: AppErrorHandler },
  ],
};
```

## Weiterführendes

- **Sentry-Integration:** `@sentry/angular` stellt einen eigenen `ErrorHandler` bereit, der sich nahtlos in dieses Muster einfügt – einfach `AppErrorHandler` daraus ableiten und `Sentry.captureException(error)` aufrufen.
- **Retry-Strategie:** Der HTTP-Interceptor kann mit `retryWhen` oder `retry({ count: 3, delay: 1000 })` für transiente Netzwerkfehler (408, 503) erweitert werden, bevor er die Fehlermeldung weitergibt.
- **Offizielle Docs:** [angular.dev/api/core/ErrorHandler](https://angular.dev/api/core/ErrorHandler)
