# Angular Dojo: CanMatch Route Guard
**Datum:** 2026-10-30
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie der `canMatch`-Guard in Angular funktioniert und wann er gegenüber `canActivate` eingesetzt werden sollte – insbesondere für Feature-Flags, Rollen-basiertes Routing und A/B-Testing.

## Hintergrund & Theorie
`canMatch` wurde in Angular 14 eingeführt und löst ein fundamentales Problem: Mit `canActivate` wird eine Route zwar geblockt, aber das Lazy-Loading-Modul wird dennoch geladen. Mit `canMatch` wird entschieden, ob eine Route überhaupt gematcht werden darf – schlägt der Guard fehl, sucht der Router nach der nächsten passenden Route.

**Wichtiger Unterschied:**
- `canActivate`: Route wird gematcht, Bundle wird geladen, dann wird geprüft ob Zugriff erlaubt.
- `canMatch`: Route wird nur gematcht (und das Bundle nur geladen), wenn der Guard `true` zurückgibt. Schlägt er fehl, probiert Angular die nächste Route mit gleichem Pfad.

Das ermöglicht folgendes Muster: Mehrere Routen mit gleichem `path`, aber unterschiedlichen `canMatch`-Guards – je nach Bedingung wird ein anderes Modul geladen:

```typescript
const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [isAdminGuard],
    loadComponent: () => import('./admin-dashboard.component')
      .then(m => m.AdminDashboardComponent),
  },
  {
    path: 'dashboard',
    loadComponent: () => import('./user-dashboard.component')
      .then(m => m.UserDashboardComponent),
  },
];
```

Guards können als Funktionen (functional) oder als Klassen mit `CanMatch`-Interface implementiert werden. Die funktionale Form ist seit Angular 15 bevorzugt.

## Aufgabe
Baue eine Angular-Anwendung mit zwei Versionen einer `dashboard`-Route: eine Admin-Version und eine Standard-Nutzer-Version. Welche geladen wird, hängt von einem `AuthService` ab, der die Rolle des Benutzers speichert.

### Schritte
1. Erstelle einen `AuthService` mit einem `userRole` Signal (`'admin' | 'user'`) und einer Methode `setRole()`.

2. Implementiere zwei funktionale `canMatch`-Guards:
   - `isAdminGuard`: gibt `true` zurück, wenn Rolle `'admin'` ist
   - `isUserGuard`: gibt `true` zurück, wenn Rolle `'user'` ist

3. Definiere in der `app.routes.ts` zwei Routen mit demselben `path: 'dashboard'`:
   - Die erste mit `canMatch: [isAdminGuard]` lädt `AdminDashboardComponent`
   - Die zweite mit `canMatch: [isUserGuard]` lädt `UserDashboardComponent`

4. Erstelle eine `AppComponent` mit zwei Buttons ("Als Admin anmelden" / "Als User anmelden"), die die Rolle setzen und dann zu `/dashboard` navigieren.

5. Überprüfe per Browser-DevTools (Network-Tab), dass wirklich nur das benötigte Bundle geladen wird – nicht beide.

## Hints
<details>
<summary>Hint 1 – Funktionaler Guard mit inject()</summary>

```typescript
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { AuthService } from './auth.service';

export const isAdminGuard: CanMatchFn = () => {
  const auth = inject(AuthService);
  return auth.userRole() === 'admin';
};
```

`CanMatchFn` hat die Signatur `(route: Route, segments: UrlSegment[]) => MaybeAsync<GuardResult>`.
Du kannst die Parameter auch weglassen, wenn du sie nicht brauchst.
</details>

<details>
<summary>Hint 2 – Mehrere Routen mit gleichem path</summary>

Angular durchsucht `routes`-Arrays sequenziell. Wenn `canMatch` `false` zurückgibt oder kein Guard passt, wird die nächste Route mit gleichem `path` ausprobiert. Eine Route ganz ohne `canMatch` am Ende dient als Fallback:

```typescript
export const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [isAdminGuard],
    loadComponent: () => import('./admin-dashboard/admin-dashboard.component')
      .then(m => m.AdminDashboardComponent),
  },
  {
    path: 'dashboard',
    // kein canMatch = immer gematcht, wenn vorherige Routes fehlschlugen
    loadComponent: () => import('./user-dashboard/user-dashboard.component')
      .then(m => m.UserDashboardComponent),
  },
];
```
</details>

## Beispiellösung

```typescript
// auth.service.ts
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthService {
  userRole = signal<'admin' | 'user'>('user');

  setRole(role: 'admin' | 'user') {
    this.userRole.set(role);
  }
}
```

```typescript
// guards/is-admin.guard.ts
import { inject } from '@angular/core';
import { CanMatchFn } from '@angular/router';
import { AuthService } from '../auth.service';

export const isAdminGuard: CanMatchFn = () => {
  return inject(AuthService).userRole() === 'admin';
};
```

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { isAdminGuard } from './guards/is-admin.guard';

export const routes: Routes = [
  {
    path: 'dashboard',
    canMatch: [isAdminGuard],
    loadComponent: () => import('./admin-dashboard/admin-dashboard.component')
      .then(m => m.AdminDashboardComponent),
  },
  {
    path: 'dashboard',
    loadComponent: () => import('./user-dashboard/user-dashboard.component')
      .then(m => m.UserDashboardComponent),
  },
  { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
];
```

```typescript
// app.component.ts
import { Component, inject } from '@angular/core';
import { Router, RouterOutlet } from '@angular/router';
import { AuthService } from './auth.service';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <button (click)="login('admin')">Als Admin anmelden</button>
    <button (click)="login('user')">Als User anmelden</button>
    <router-outlet />
  `,
})
export class AppComponent {
  private auth = inject(AuthService);
  private router = inject(Router);

  login(role: 'admin' | 'user') {
    this.auth.setRole(role);
    this.router.navigateByUrl('/dashboard');
  }
}
```

```typescript
// admin-dashboard/admin-dashboard.component.ts
import { Component } from '@angular/core';

@Component({
  standalone: true,
  template: `<h1>Admin Dashboard</h1><p>Nur für Admins sichtbar.</p>`,
})
export class AdminDashboardComponent {}
```

```typescript
// user-dashboard/user-dashboard.component.ts
import { Component } from '@angular/core';

@Component({
  standalone: true,
  template: `<h1>User Dashboard</h1><p>Standard-Ansicht für alle Nutzer.</p>`,
})
export class UserDashboardComponent {}
```

## Weiterführendes
- **Kombination mit `CanActivate`**: `canMatch` verhindert das Laden, `canActivate` den Zugriff auf eine bereits gematchte Route – beide Guards können kombiniert werden.
- **`canMatch` mit Guards die `UrlTree` zurückgeben**: Gibt ein Guard einen `UrlTree` zurück, leitet Angular zur angegebenen URL um, anstatt zur nächsten Route weiterzumachen.
- **Offizielle Doku**: [https://angular.dev/api/router/CanMatchFn](https://angular.dev/api/router/CanMatchFn)
- **Vertiefung**: Implementiere einen Guard, der eine HTTP-Anfrage macht (z.B. Permissions vom Backend lädt) und das `loadComponent` erst danach auslöst – nutze dafür `firstValueFrom(http.get(...))` im Guard.
