# Angular Dojo: HTTP Interceptors
**Datum:** 2026-06-14
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit funktionalen HTTP Interceptors in Angular (ab v15) HTTP-Requests und -Responses zentral abfangen, manipulieren und überwachen kannst – ohne dabei Komponenten oder Services direkt anzupassen.

## Hintergrund & Theorie

HTTP Interceptors sind Middleware-Schichten für den `HttpClient`. Sie können **jeden ausgehenden Request** und **jede eingehende Response** abfangen, bevor diese die eigentliche Logik erreicht.

Seit Angular 15 gibt es neben den klassischen klassenbasierten Interceptors auch **funktionale Interceptors** – deutlich schlanker und besser mit Standalone-APIs kombinierbar:

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  const authReq = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  return next(authReq);
};
```

Interceptors werden in `provideHttpClient(withInterceptors([...]))` registriert und in der Reihenfolge ihrer Registrierung ausgeführt. Jeder Interceptor gibt ein `Observable<HttpEvent<any>>` zurück – dadurch lassen sich mit RxJS auch Responses transformieren, Fehler behandeln oder Requests wiederholen.

Typische Anwendungsfälle:
- **Auth-Token hinzufügen** (Bearer Token in Header)
- **Globale Fehlerbehandlung** (z. B. 401 → Logout)
- **Request-Logging** (Dauer messen, URL loggen)
- **Caching** (Responses zwischenspeichern)
- **Loading-Spinner** (zentral steuern)

## Aufgabe

Implementiere **zwei funktionale Interceptors** in einer kleinen Standalone-Angular-App:

1. **`authInterceptor`** – Hängt einen Bearer-Token an jeden ausgehenden Request, der *nicht* zur Login-URL geht.
2. **`loggingInterceptor`** – Loggt URL, Methode und Dauer jedes HTTP-Requests in die Konsole.

Teste beide Interceptors, indem du einen einfachen `HttpClient.get()`-Aufruf absetzt und überprüfst, ob der Header gesetzt wird und das Logging funktioniert.

### Schritte

1. Erstelle eine neue Standalone-App (`ng new`) oder nutze ein bestehendes Projekt. Stelle sicher, dass `provideHttpClient()` in `app.config.ts` registriert ist.

2. Erstelle die Datei `src/app/interceptors/auth.interceptor.ts` und implementiere den `authInterceptor`:
   - Injiziere einen `AuthService` (kann ein einfacher Service mit einem hartcodierten Token sein).
   - Clone den Request und setze den `Authorization`-Header – aber nur, wenn die URL nicht `/api/auth/login` enthält.

3. Erstelle `src/app/interceptors/logging.interceptor.ts` und implementiere den `loggingInterceptor`:
   - Notiere den Startzeitpunkt mit `Date.now()`.
   - Rufe `next(req)` auf und pipe das Observable.
   - Nutze den RxJS-Operator `tap`, um beim Event-Typ `HttpResponse` die Dauer zu berechnen und zu loggen.

4. Registriere beide Interceptors in `app.config.ts`:
   ```typescript
   provideHttpClient(withInterceptors([authInterceptor, loggingInterceptor]))
   ```

5. Rufe in einer Komponente per `HttpClient.get('https://jsonplaceholder.typicode.com/todos/1')` einen Test-Request ab und beobachte Konsole und Network-Tab.

## Hints

<details>
<summary>Hint 1 – Request clonen mit gesetztem Header</summary>

`HttpRequest` ist immutable. Nutze `.clone()` um eine modifizierte Kopie zu erzeugen:

```typescript
const cloned = req.clone({
  setHeaders: {
    Authorization: `Bearer ${token}`
  }
});
return next(cloned);
```

</details>

<details>
<summary>Hint 2 – Dauer im loggingInterceptor messen</summary>

Verwende `tap` aus RxJS und filtere auf `HttpResponse`:

```typescript
import { tap } from 'rxjs/operators';
import { HttpEventType } from '@angular/common/http';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = Date.now();
  return next(req).pipe(
    tap(event => {
      if (event.type === HttpEventType.Response) {
        const elapsed = Date.now() - started;
        console.log(`[HTTP] ${req.method} ${req.url} – ${elapsed}ms`);
      }
    })
  );
};
```

</details>

<details>
<summary>Hint 3 – Login-URL ausschließen</summary>

Prüfe die URL des Requests, bevor du den Header setzt:

```typescript
if (req.url.includes('/api/auth/login')) {
  return next(req); // unverändert weiterleiten
}
```

</details>

## Beispiellösung

```typescript
// src/app/services/auth.service.ts
import { Injectable } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthService {
  getToken(): string {
    return 'my-secret-jwt-token';
  }
}
```

```typescript
// src/app/interceptors/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.url.includes('/api/auth/login')) {
    return next(req);
  }

  const token = inject(AuthService).getToken();
  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(authReq);
};
```

```typescript
// src/app/interceptors/logging.interceptor.ts
import { HttpEventType, HttpInterceptorFn } from '@angular/common/http';
import { tap } from 'rxjs/operators';

export const loggingInterceptor: HttpInterceptorFn = (req, next) => {
  const started = Date.now();
  console.log(`[HTTP] ▶ ${req.method} ${req.url}`);

  return next(req).pipe(
    tap(event => {
      if (event.type === HttpEventType.Response) {
        const elapsed = Date.now() - started;
        console.log(`[HTTP] ✓ ${req.method} ${req.url} – Status: ${event.status} – ${elapsed}ms`);
      }
    })
  );
};
```

```typescript
// src/app/app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './interceptors/auth.interceptor';
import { loggingInterceptor } from './interceptors/logging.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, loggingInterceptor])
    )
  ]
};
```

```typescript
// src/app/app.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `<pre>{{ data | json }}</pre>`
})
export class AppComponent implements OnInit {
  private http = inject(HttpClient);
  data: unknown;

  ngOnInit() {
    this.http.get('https://jsonplaceholder.typicode.com/todos/1').subscribe(res => {
      this.data = res;
    });
  }
}
```

## Weiterführendes

- **Fehlerbehandlung zentral**: Erweitere den `loggingInterceptor` mit `catchError`, um 4xx/5xx-Fehler global zu behandeln und z. B. bei 401 einen Logout auszulösen.
- **Retry-Logik**: Kombiniere mit RxJS `retry` oder `retryWhen`, um fehlgeschlagene Requests automatisch zu wiederholen.
- **Offizielle Doku**: [Angular – HTTP Interceptors](https://angular.dev/guide/http/interceptors)
- **Tipp**: Mehrere Interceptors lassen sich gezielt testen, indem du `HttpClientTestingModule` oder `provideHttpClientTesting()` nutzt und `HttpTestingController` verwendest.
