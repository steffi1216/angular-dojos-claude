# Angular Dojo: Functional inject() Patterns
**Datum:** 2026-07-20
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, die `inject()`-Funktion in funktionalen Kontexten einzusetzen – in Guards, Resolvern, Interceptors und wiederverwendbaren „Inject-Functions" – und verstehst dabei, wann und warum dieser Ansatz dem klassischen Constructor-DI überlegen ist.

## Hintergrund & Theorie

Seit Angular 14 kann `inject()` überall aufgerufen werden, wo ein **Injection Context** aktiv ist:
- im Konstruktor einer Klasse (klassisch)
- in **Klassen-Feldern** (Field Initializer)
- in **funktionalen Guards, Resolvern und Interceptors** (Funktion direkt als Provider)
- in **`runInInjectionContext()`** (manuell erzeugter Kontext)

Der größte Vorteil: Wiederverwendbare **Inject-Functions** – einfache Funktionen, die `inject()` intern nutzen und so Abhängigkeiten kapseln, ohne eine eigene Klasse zu erfordern.

```typescript
// Klassisch (Klasse notwendig)
@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private auth: AuthService, private router: Router) {}
  canActivate() { … }
}

// Modern (einfache Funktion)
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.parseUrl('/login');
};
```

Diese Kompositionsstrategie (ähnlich wie React Hooks) macht Code deutlich testbarer und lesbarer. `inject()` ist zur Laufzeit auf den Injection Context angewiesen – außerhalb dieses Kontexts wirft es einen Fehler.

## Aufgabe

Baue ein Mini-Routing-Feature mit ausschließlich funktionalen DI-Patterns:

1. Eine wiederverwendbare **Inject-Function** `injectCurrentUser()`, die den eingeloggten User als Signal zurückgibt.
2. Einen funktionalen **`authGuard`**, der unangemeldete Nutzer auf `/login` umleitet.
3. Einen funktionalen **`Resolver`** (`userResolver`), der Nutzerdaten vor dem Route-Rendering lädt.
4. Einen funktionalen **HTTP-Interceptor** (`authInterceptor`), der den Bearer-Token aus dem `AuthService` automatisch anhängt.

### Schritte

1. **Inject-Function erstellen**
   Schreibe eine Funktion `injectCurrentUser()` in `auth/inject-current-user.ts`. Sie soll intern `inject(AuthService)` verwenden und ein `computed`-Signal auf Basis von `authService.user` zurückgeben.

2. **Funktionalen Guard erstellen**
   Erstelle `auth/auth.guard.ts` mit einer `CanActivateFn`. Nutze `injectCurrentUser()` und `inject(Router)`. Ist kein User vorhanden, leite auf `/login` um; sonst gib `true` zurück.

3. **Funktionalen Resolver erstellen**
   Erstelle `auth/user.resolver.ts` als `ResolveFn<User>`. Injiziere `UserService` und `ActivatedRouteSnapshot` – lies die `id`-Route-Parameter aus und lade den User asynchron.

4. **Funktionalen Interceptor erstellen**
   Erstelle `auth/auth.interceptor.ts` als `HttpInterceptorFn`. Injiziere `AuthService`, lese den Token aus und klone den Request mit gesetztem `Authorization`-Header.

5. **Alles in `app.config.ts` verdrahten**
   Registriere Guard, Resolver und Interceptor über `provideRouter(routes)` und `provideHttpClient(withInterceptors([authInterceptor]))`.

## Hints

<details>
<summary>Hint 1 – Inject-Function Grundstruktur</summary>

```typescript
// auth/inject-current-user.ts
import { inject } from '@angular/core';
import { AuthService } from './auth.service';
import { computed } from '@angular/core';

export function injectCurrentUser() {
  const auth = inject(AuthService);
  // Annahme: auth.user ist ein Signal<User | null>
  return computed(() => auth.user());
}
```

Diese Funktion kann in jedem Injection Context aufgerufen werden – also in Guards, Resolvern, Komponenten-Feldern usw.

</details>

<details>
<summary>Hint 2 – Funktionaler Guard mit UrlTree</summary>

```typescript
// auth/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { injectCurrentUser } from './inject-current-user';

export const authGuard: CanActivateFn = () => {
  const currentUser = injectCurrentUser();
  const router = inject(Router);
  // Signal lesen – kein Subscription nötig
  return currentUser() ? true : router.parseUrl('/login');
};
```

</details>

<details>
<summary>Hint 3 – Funktionaler HTTP-Interceptor</summary>

```typescript
// auth/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (!token) return next(req);

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(authReq);
};
```

In `app.config.ts`: `provideHttpClient(withInterceptors([authInterceptor]))`

</details>

<details>
<summary>Hint 4 – Funktionaler Resolver</summary>

```typescript
// auth/user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { UserService } from './user.service';
import { User } from './user.model';

export const userResolver: ResolveFn<User> = (route) => {
  const id = route.paramMap.get('id')!;
  return inject(UserService).getUserById(id);
  // Observable oder Promise – Angular wartet automatisch
};
```

Im Route-Config: `{ path: 'user/:id', resolve: { user: userResolver }, … }`

</details>

## Beispiellösung

```typescript
// auth/auth.service.ts
import { Injectable, signal } from '@angular/core';

export interface User { id: string; name: string; }

@Injectable({ providedIn: 'root' })
export class AuthService {
  readonly user = signal<User | null>(null);
  private token = signal<string | null>(null);

  login(user: User, token: string) {
    this.user.set(user);
    this.token.set(token);
  }

  logout() {
    this.user.set(null);
    this.token.set(null);
  }

  getToken() { return this.token(); }
  isLoggedIn() { return this.user() !== null; }
}

// auth/inject-current-user.ts
import { inject, computed } from '@angular/core';
import { AuthService } from './auth.service';

export function injectCurrentUser() {
  return computed(() => inject(AuthService).user());
}

// auth/auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { injectCurrentUser } from './inject-current-user';

export const authGuard: CanActivateFn = () => {
  const currentUser = injectCurrentUser();
  const router = inject(Router);
  return currentUser() ? true : router.parseUrl('/login');
};

// auth/user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { UserService } from './user.service';
import { User } from './auth.service';

export const userResolver: ResolveFn<User> = (route) =>
  inject(UserService).getUserById(route.paramMap.get('id')!);

// auth/auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (!token) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { authInterceptor } from './auth/auth.interceptor';
import { authGuard } from './auth/auth.guard';
import { userResolver } from './auth/user.resolver';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withInterceptors([authInterceptor])),
    provideRouter([
      {
        path: 'dashboard',
        canActivate: [authGuard],
        loadComponent: () => import('./dashboard/dashboard.component')
          .then(m => m.DashboardComponent),
      },
      {
        path: 'user/:id',
        canActivate: [authGuard],
        resolve: { user: userResolver },
        loadComponent: () => import('./user/user.component')
          .then(m => m.UserComponent),
      },
      { path: 'login', loadComponent: () => import('./login/login.component')
          .then(m => m.LoginComponent) },
    ]),
  ],
};
```

## Weiterführendes

- **`runInInjectionContext()`**: Damit kannst du manuell einen Injection Context öffnen, z. B. in setTimeout-Callbacks oder in Libraries – `environmentInjector.runInContext(() => inject(MyService))`.
- **Composable Inject-Functions als Pattern**: Ähnlich wie React Custom Hooks lassen sich inject-Funktionen beliebig kombinieren, z. B. `injectAuthHeaders()` ruft intern `injectCurrentUser()` auf.
- [Angular Docs – Injection Context](https://angular.dev/guide/di/dependency-injection-context)
- [Angular Docs – Functional Guards](https://angular.dev/guide/routing/common-router-tasks#canactivate-functional)
