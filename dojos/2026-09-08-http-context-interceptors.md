# Angular Dojo: HttpContext – Anfrage-spezifische Metadaten für Interceptors
**Datum:** 2026-09-08
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `HttpContext` und `HttpContextToken` pro HTTP-Request eigene Metadaten mitgeben kannst, die Interceptors gezielt lesen und auswerten – ohne globale Zustände oder Request-URL-Hacks.

## Hintergrund & Theorie

Vor `HttpContext` (eingeführt in Angular 12) mussten Entwickler oft Tricks wie URL-Pattern-Matching oder globale Flags verwenden, um einem Interceptor mitzuteilen, dass ein bestimmter Request anders behandelt werden soll – z.B. „diesen Request nicht authentifizieren" oder „diesen Request bis zu 3-mal wiederholen".

`HttpContext` löst dieses Problem elegant: Jeder `HttpRequest` trägt eine eigene, typsichere `HttpContext`-Map. Ein `HttpContextToken<T>` definiert dabei den Schlüssel und den Default-Wert. Interceptors lesen den Token aus dem Context, ohne die restliche Logik zu berühren.

**Wichtige Klassen:**
- `HttpContextToken<T>` – definiert einen typisierten Token mit Default-Wert
- `HttpContext` – eine Map von Tokens zu Werten, die an einen Request gehängt wird
- `request.context.get(TOKEN)` – liest den Wert im Interceptor
- `context.set(TOKEN, value)` – setzt den Wert beim Erstellen des Requests

`HttpContext` ist **mutable** – Interceptors können Werte auch verändern, z.B. um nachgelagerte Interceptors zu informieren.

## Aufgabe

Baue ein System mit zwei Interceptors und `HttpContext`:

1. **`AuthInterceptor`** – hängt einen `Authorization`-Header an alle Requests, **außer** wenn der Request mit `SKIP_AUTH` markiert ist.
2. **`RetryInterceptor`** – wiederholt fehlgeschlagene Requests, wobei die Anzahl der Wiederholungen per `RETRY_COUNT`-Token konfigurierbar ist (Default: 1).

Nutze diese Interceptors in einem Service, der zwei verschiedene HTTP-Calls macht: einen öffentlichen (ohne Auth, mit 3 Retries) und einen privaten (mit Auth, mit Standard-Retry).

### Schritte

1. Erstelle eine Datei `http-context-tokens.ts` und definiere darin `SKIP_AUTH` (Token mit Default `false`) und `RETRY_COUNT` (Token mit Default `1`).

2. Implementiere `AuthInterceptor` als funktionalen Interceptor: Lies `SKIP_AUTH` aus dem Context – wenn `true`, leite den Request unverändert weiter; sonst füge den `Authorization`-Header hinzu.

3. Implementiere `RetryInterceptor` als funktionalen Interceptor: Lies `RETRY_COUNT` aus dem Context und nutze den RxJS-Operator `retry()` mit der entsprechenden Anzahl.

4. Registriere beide Interceptors in `app.config.ts` via `provideHttpClient(withInterceptors([...]))`.

5. Erstelle einen `ApiService` mit zwei Methoden:
   - `getPublicData()` – setzt `SKIP_AUTH: true` und `RETRY_COUNT: 3`
   - `getPrivateData()` – nutzt die Defaults (Auth aktiv, 1 Retry)

6. Teste das Verhalten, indem du die Requests in der Browser-Konsole oder den Angular DevTools beobachtest.

## Hints

<details>
<summary>Hint 1 – HttpContextToken definieren</summary>

```typescript
import { HttpContextToken } from '@angular/common/http';

export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
export const RETRY_COUNT = new HttpContextToken<number>(() => 1);
```

Der Konstruktor von `HttpContextToken` erwartet eine Factory-Funktion, die den Default-Wert liefert.
</details>

<details>
<summary>Hint 2 – Context beim Request-Aufruf setzen</summary>

```typescript
import { HttpContext } from '@angular/common/http';
import { SKIP_AUTH, RETRY_COUNT } from './http-context-tokens';

// Im Service:
getPublicData() {
  const context = new HttpContext()
    .set(SKIP_AUTH, true)
    .set(RETRY_COUNT, 3);

  return this.http.get('/api/public', { context });
}
```
</details>

<details>
<summary>Hint 3 – Token im funktionalen Interceptor lesen</summary>

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { SKIP_AUTH } from './http-context-tokens';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.context.get(SKIP_AUTH)) {
    return next(req);
  }
  const authReq = req.clone({
    setHeaders: { Authorization: 'Bearer my-token' }
  });
  return next(authReq);
};
```
</details>

## Beispiellösung

```typescript
// http-context-tokens.ts
import { HttpContextToken } from '@angular/common/http';

export const SKIP_AUTH = new HttpContextToken<boolean>(() => false);
export const RETRY_COUNT = new HttpContextToken<number>(() => 1);


// auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { SKIP_AUTH } from './http-context-tokens';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  if (req.context.get(SKIP_AUTH)) {
    return next(req);
  }
  return next(req.clone({
    setHeaders: { Authorization: 'Bearer super-secret-token' },
  }));
};


// retry.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { retry } from 'rxjs';
import { RETRY_COUNT } from './http-context-tokens';

export const retryInterceptor: HttpInterceptorFn = (req, next) => {
  const retries = req.context.get(RETRY_COUNT);
  return next(req).pipe(retry(retries));
};


// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './auth.interceptor';
import { retryInterceptor } from './retry.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(
      withInterceptors([authInterceptor, retryInterceptor])
    ),
  ],
};


// api.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpContext } from '@angular/common/http';
import { SKIP_AUTH, RETRY_COUNT } from './http-context-tokens';

@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);

  getPublicData() {
    const context = new HttpContext()
      .set(SKIP_AUTH, true)
      .set(RETRY_COUNT, 3);
    return this.http.get('/api/public', { context });
  }

  getPrivateData() {
    // Defaults: SKIP_AUTH=false, RETRY_COUNT=1
    return this.http.get('/api/private');
  }
}
```

## Weiterführendes

- **Mutable Context nutzen:** Interceptors können `req.context.set(TOKEN, newValue)` aufrufen, um nachgelagerten Interceptors Informationen zu übergeben – z.B. „Ich habe bereits einen Retry durchgeführt".
- **Offizielle Docs:** [Angular HttpContext](https://angular.dev/api/common/http/HttpContext)
- **Pattern:** Kombiniere `HttpContext` mit einem `LoadingInterceptor`, der via Token entscheidet, ob ein globaler Lade-Indikator gezeigt werden soll – sauberer als URL-Whitelists.
- **Tipp:** `HttpContext` eignet sich auch dafür, Request-IDs für Tracing mitzugeben, die dann im `ErrorHandler` ausgelesen werden können.
