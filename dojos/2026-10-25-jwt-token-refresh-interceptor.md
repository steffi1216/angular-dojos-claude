# Angular Dojo: JWT Token Refresh Interceptor mit Queue-Mechanismus
**Datum:** 2026-10-25
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du implementierst einen funktionalen HTTP-Interceptor, der bei einem 401-Fehler automatisch den JWT-Access-Token erneuert und dabei alle parallel eintreffenden Anfragen in einer Queue puffert – so wird ein "Token Refresh Thundering Herd" verhindert.

## Hintergrund & Theorie

In SPAs läuft ein JWT-Access-Token typischerweise nach kurzer Zeit ab (5–15 Minuten). Wenn mehrere HTTP-Anfragen gleichzeitig mit einem abgelaufenen Token gesendet werden, würden ohne Schutz mehrere parallele Refresh-Aufrufe starten.

Das klassische Lösungsmuster:
1. Die erste 401-Antwort triggert den Token-Refresh.
2. Alle weiteren Anfragen, die in dieser Zeit ankommen, werden in einer **Queue** gehalten.
3. Sobald der neue Token da ist, werden alle gepufferten Anfragen mit dem neuen Token wiederholt.
4. Schlägt der Refresh fehl (z. B. Refresh-Token abgelaufen), werden alle gepufferten Anfragen abgebrochen und der User wird ausgeloggt.

In Angular wird das elegant mit einem `BehaviorSubject` als Sperr-Signal und dem RxJS-Operator `switchMap` / `filter` umgesetzt. Seit Angular v15+ bevorzugen wir **funktionale Interceptoren** (`HttpInterceptorFn`).

## Aufgabe

Implementiere einen funktionalen Interceptor `authInterceptor`, der:
- Gültige Anfragen mit einem `Authorization: Bearer <token>` Header versieht.
- Bei einer 401-Antwort einen Token-Refresh auslöst.
- Alle weiteren eintreffenden Anfragen währenddessen buffert (Queue via `BehaviorSubject`).
- Nach erfolgreichem Refresh alle Anfragen mit dem neuen Token wiederholt.
- Bei einem fehlgeschlagenen Refresh alle Anfragen abbricht und den User ausloggt.

### Schritte

1. Erstelle einen `AuthService` mit einem `BehaviorSubject<string | null>` für den aktuellen Access-Token und einer `refreshToken()`-Methode, die einen neuen Token zurückgibt (Mock: nach 500ms).

2. Erstelle ein `BehaviorSubject<boolean>` namens `isRefreshing$` sowie ein `Subject<string | null>` namens `tokenRefreshed$` als modul-globale Variablen im Interceptor-File (außerhalb des Interceptors, damit alle Anfragen denselben State teilen).

3. Implementiere die `HttpInterceptorFn`:
   - Hänge den aktuellen Token als Header an.
   - Fange `HttpErrorResponse` mit Status 401 ab.
   - Wenn `isRefreshing$.value === false`: setze auf `true`, rufe `authService.refreshToken()` auf, emittiere bei Erfolg den neuen Token auf `tokenRefreshed$`, setze `isRefreshing$` zurück auf `false`.
   - Wenn `isRefreshing$.value === true`: warte via `tokenRefreshed$.pipe(filter(Boolean), take(1))` auf den neuen Token und wiederhole die Anfrage.

4. Registriere den Interceptor in `app.config.ts` mit `withInterceptors([authInterceptor])`.

5. Schreibe einen einfachen Test (oder Demo-Component), der zwei gleichzeitige Anfragen sendet, während der Token abläuft, und überprüfe, dass der Refresh nur einmal aufgerufen wird.

## Hints

<details>
<summary>Hint 1 – BehaviorSubject als Sperre</summary>

```typescript
// Außerhalb des Interceptors – geteilt über alle Instanzen
let isRefreshing = false;
const tokenRefreshed$ = new Subject<string | null>();

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  // ...
};
```

`isRefreshing` als einfaches `boolean` (kein Subject nötig) reicht, weil Interceptors synchron entscheiden, ob sie in die Queue müssen.

</details>

<details>
<summary>Hint 2 – Anfrage mit neuem Token wiederholen</summary>

```typescript
function addToken(req: HttpRequest<unknown>, token: string) {
  return req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
}

// Im catch-Zweig, wenn bereits ein Refresh läuft:
return tokenRefreshed$.pipe(
  filter(token => token !== null),
  take(1),
  switchMap(token => next(addToken(req, token!)))
);
```

</details>

<details>
<summary>Hint 3 – Fehlerfall: Refresh schlägt fehl</summary>

```typescript
catchError(refreshError => {
  isRefreshing = false;
  tokenRefreshed$.next(null); // signalisiert allen Wartenden: abbrechen
  authService.logout();
  return throwError(() => refreshError);
})
```

</details>

## Beispiellösung

```typescript
// auth.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject, Observable, tap } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);
  private accessToken = new BehaviorSubject<string | null>(
    localStorage.getItem('access_token')
  );

  getToken(): string | null {
    return this.accessToken.value;
  }

  refreshToken(): Observable<{ access_token: string }> {
    return this.http.post<{ access_token: string }>('/api/auth/refresh', {}).pipe(
      tap(({ access_token }) => {
        localStorage.setItem('access_token', access_token);
        this.accessToken.next(access_token);
      })
    );
  }

  logout(): void {
    localStorage.removeItem('access_token');
    this.accessToken.next(null);
    // Router-Navigation zur Login-Seite hier
  }
}
```

```typescript
// auth.interceptor.ts
import {
  HttpErrorResponse,
  HttpInterceptorFn,
  HttpRequest
} from '@angular/common/http';
import { inject } from '@angular/core';
import { Subject, catchError, filter, switchMap, take, throwError } from 'rxjs';
import { AuthService } from './auth.service';

// Modul-globaler State – wird von allen parallelen Anfragen geteilt
let isRefreshing = false;
const tokenRefreshed$ = new Subject<string | null>();

function addAuthHeader(req: HttpRequest<unknown>, token: string) {
  return req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
}

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getToken();

  const authReq = token ? addAuthHeader(req, token) : req;

  return next(authReq).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status !== 401) {
        return throwError(() => error);
      }

      if (!isRefreshing) {
        // Erster 401: Refresh starten
        isRefreshing = true;

        return authService.refreshToken().pipe(
          switchMap(({ access_token }) => {
            isRefreshing = false;
            tokenRefreshed$.next(access_token); // alle Wartenden benachrichtigen
            return next(addAuthHeader(req, access_token));
          }),
          catchError(refreshError => {
            isRefreshing = false;
            tokenRefreshed$.next(null); // alle Wartenden abbrechen
            authService.logout();
            return throwError(() => refreshError);
          })
        );
      }

      // Weiterer 401 während Refresh läuft: in Queue einreihen
      return tokenRefreshed$.pipe(
        filter(newToken => newToken !== null),
        take(1),
        switchMap(newToken => next(addAuthHeader(req, newToken!)))
      );
    })
  );
};
```

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import {
  provideHttpClient,
  withInterceptors
} from '@angular/common/http';
import { authInterceptor } from './auth/auth.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor]))
  ]
};
```

## Weiterführendes
- **Race-Condition-Schutz**: Für reaktivere Ansätze kann `tokenRefreshed$` durch ein `ReplaySubject(1)` ersetzt werden, das nach jedem Refresh-Zyklus mit `reset()` zurückgesetzt wird.
- **Doku**: [Angular HttpClient Interceptors](https://angular.dev/guide/http/interceptors) und das Muster wird in der Angular-Community auch als *"Interceptor Token Refresh Queue"* bezeichnet.
- **Tipp**: Bei OAuth 2.0 / PKCE-Flows bietet die Bibliothek `angular-auth-oidc-client` dieses Muster bereits out-of-the-box.
