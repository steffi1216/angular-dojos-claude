# Angular Dojo: Angular Router Advanced
**Datum:** 2026-06-17
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie man den Angular Router mit Guards, Resolvers, Child Routes und Lazy Loading professionell einsetzt, um sichere, performante und gut strukturierte Navigation zu bauen.

## Hintergrund & Theorie

Der Angular Router ist weit mehr als eine einfache URL-zu-Komponente-Zuordnung. Vier fortgeschrittene Konzepte sind besonders wichtig:

**Guards** kontrollieren den Zugang zu Routen. Mit `CanActivateFn` (functional guards seit Angular 15+) prüfst du z. B. ob ein User eingeloggt ist, bevor eine Route aktiviert wird. `CanDeactivateFn` verhindert versehentliches Verlassen bei ungespeicherten Daten.

**Resolvers** laden Daten *bevor* eine Komponente gerendert wird. Statt `ngOnInit` mit einem Lade-Spinner zu füllen, liefert der Resolver die Daten synchron über `ActivatedRoute.data`. Seit Angular 16 können Resolvers auch Signals zurückgeben.

**Child Routes** erlauben hierarchische Layouts: Eine Parent-Komponente mit `<router-outlet>` hostet Kind-Routen, die ihren eigenen URL-Pfad haben.

**Lazy Loading** teilt die App in separate Bundles auf. Mit `loadComponent()` (standalone) oder `loadChildren()` lädt Angular Module/Komponenten nur bei Bedarf. Das reduziert die initiale Bundle-Größe erheblich.

Seit Angular 14+ sind **functional guards** der bevorzugte Ansatz – keine Klassen, kein `implements`, nur reine Funktionen oder `inject()`-basierte Factories.

## Aufgabe

Baue ein Mini-Feature: eine geschützte Admin-Sektion mit Lazy Loading, einem Auth-Guard und einem Resolver, der Benutzerdaten vorab lädt.

### Schritte

1. **Auth-Service erstellen** – Simuliere einen `AuthService` mit einer `isLoggedIn()`-Methode (gibt ein `Signal<boolean>` zurück) und einer `currentUser()`-Methode.

2. **Functional Auth-Guard implementieren** – Erstelle einen Guard als pure Funktion mit `inject()`. Leite nicht eingeloggte User zu `/login` um.

3. **User-Resolver implementieren** – Erstelle einen Resolver, der (mit einer simulierten Verzögerung via `of(...).pipe(delay(500))`) Benutzerdaten zurückgibt.

4. **Lazy-loaded Admin-Feature konfigurieren** – Definiere eine Route mit `loadComponent()`, dem Guard und dem Resolver. Füge eine Child-Route für ein Dashboard hinzu.

5. **Admin-Komponente** – Lies die Resolver-Daten aus `ActivatedRoute.data` aus und zeige sie an. Nutze `CanDeactivate`, um beim Verlassen zu warnen, falls ein Formular dirty ist.

6. **Route-Konfiguration verdrahten** – Konfiguriere alles in der App-Route-Konfiguration und teste die Navigation.

## Hints

<details>
<summary>Hint 1 – Functional Guard mit inject()</summary>

```typescript
// auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn()) {
    return true;
  }
  // Speichere die ursprüngliche URL für den Redirect nach dem Login
  return router.createUrlTree(['/login'], {
    queryParams: { returnUrl: state.url }
  });
};
```

</details>

<details>
<summary>Hint 2 – Resolver als Funktion</summary>

```typescript
// user.resolver.ts
import { inject } from '@angular/core';
import { ResolveFn } from '@angular/router';
import { delay, of } from 'rxjs';
import { UserService } from './user.service';

export interface UserData {
  id: number;
  name: string;
  role: string;
}

export const userResolver: ResolveFn<UserData> = (route, state) => {
  const userId = route.paramMap.get('id') ?? '1';
  // Simulierte API – in der Praxis: inject(UserService).getUser(userId)
  return of({ id: +userId, name: 'Admin User', role: 'admin' }).pipe(delay(500));
};
```

</details>

<details>
<summary>Hint 3 – Lazy Route mit Child Routes</summary>

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { authGuard } from './guards/auth.guard';
import { userResolver } from './resolvers/user.resolver';

export const routes: Routes = [
  { path: 'login', loadComponent: () => import('./login/login.component').then(m => m.LoginComponent) },
  {
    path: 'admin',
    canActivate: [authGuard],
    loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent),
    resolve: { user: userResolver },
    children: [
      { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
      {
        path: 'dashboard',
        loadComponent: () => import('./admin/dashboard/dashboard.component').then(m => m.DashboardComponent)
      },
      {
        path: 'settings',
        loadComponent: () => import('./admin/settings/settings.component').then(m => m.SettingsComponent),
        canDeactivate: [unsavedChangesGuard]
      }
    ]
  },
  { path: '**', redirectTo: '/login' }
];
```

</details>

<details>
<summary>Hint 4 – Resolver-Daten in der Komponente lesen</summary>

```typescript
// admin.component.ts
import { Component, inject } from '@angular/core';
import { ActivatedRoute, RouterOutlet } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs';
import { UserData } from '../resolvers/user.resolver';

@Component({
  standalone: true,
  imports: [RouterOutlet],
  template: `
    <h1>Admin: {{ user()?.name }}</h1>
    <nav>
      <a routerLink="dashboard">Dashboard</a> |
      <a routerLink="settings">Settings</a>
    </nav>
    <router-outlet />
  `
})
export class AdminComponent {
  private route = inject(ActivatedRoute);

  user = toSignal(
    this.route.data.pipe(map(data => data['user'] as UserData))
  );
}
```

</details>

## Beispiellösung

```typescript
// ===== auth.service.ts =====
import { Injectable, signal } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private loggedIn = signal(false);

  isLoggedIn = this.loggedIn.asReadonly();

  login(username: string, password: string): boolean {
    // Simulation: jeder Login funktioniert
    this.loggedIn.set(true);
    return true;
  }

  logout(): void {
    this.loggedIn.set(false);
  }
}

// ===== auth.guard.ts =====
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (_route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn()
    ? true
    : router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

// ===== unsaved-changes.guard.ts =====
import { CanDeactivateFn } from '@angular/router';

export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('Du hast ungespeicherte Änderungen. Seite wirklich verlassen?');
  }
  return true;
};

// ===== user.resolver.ts =====
import { ResolveFn } from '@angular/router';
import { delay, of } from 'rxjs';

export interface UserData {
  id: number;
  name: string;
  role: string;
}

export const userResolver: ResolveFn<UserData> = (route) => {
  const id = Number(route.paramMap.get('id') ?? 1);
  return of({ id, name: 'Anna Admin', role: 'admin' }).pipe(delay(500));
};

// ===== app.routes.ts =====
import { Routes } from '@angular/router';
import { authGuard } from './auth.guard';
import { unsavedChangesGuard } from './unsaved-changes.guard';
import { userResolver } from './user.resolver';

export const routes: Routes = [
  {
    path: 'login',
    loadComponent: () => import('./login/login.component').then(m => m.LoginComponent)
  },
  {
    path: 'admin',
    canActivate: [authGuard],
    resolve: { user: userResolver },
    loadComponent: () => import('./admin/admin.component').then(m => m.AdminComponent),
    children: [
      { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
      {
        path: 'dashboard',
        loadComponent: () => import('./admin/dashboard/dashboard.component')
          .then(m => m.DashboardComponent)
      },
      {
        path: 'settings',
        canDeactivate: [unsavedChangesGuard],
        loadComponent: () => import('./admin/settings/settings.component')
          .then(m => m.SettingsComponent)
      }
    ]
  },
  { path: '**', redirectTo: '/login' }
];

// ===== admin.component.ts =====
import { Component, inject } from '@angular/core';
import { ActivatedRoute, RouterLink, RouterOutlet } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs';
import { UserData } from './user.resolver';

@Component({
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <header>
      <h1>Admin Panel – Willkommen, {{ user()?.name }}!</h1>
      <nav>
        <a routerLink="dashboard" routerLinkActive="active">Dashboard</a>
        <a routerLink="settings" routerLinkActive="active">Settings</a>
      </nav>
    </header>
    <main>
      <router-outlet />
    </main>
  `
})
export class AdminComponent {
  private route = inject(ActivatedRoute);
  user = toSignal(this.route.data.pipe(map(d => d['user'] as UserData)));
}

// ===== admin/settings/settings.component.ts =====
import { Component, signal } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { HasUnsavedChanges } from '../../unsaved-changes.guard';

@Component({
  standalone: true,
  imports: [FormsModule],
  template: `
    <h2>Einstellungen</h2>
    <input [(ngModel)]="displayName" (ngModelChange)="dirty.set(true)" placeholder="Anzeigename" />
    <p *ngIf="dirty()">Ungespeicherte Änderungen!</p>
    <button (click)="save()">Speichern</button>
  `
})
export class SettingsComponent implements HasUnsavedChanges {
  displayName = signal('');
  dirty = signal(false);

  save(): void {
    // API-Call hier
    this.dirty.set(false);
  }

  hasUnsavedChanges(): boolean {
    return this.dirty();
  }
}
```

## Weiterführendes

- **withComponentInputBinding()**: Seit Angular 16 kannst du Route-Parameter und Resolver-Daten direkt als `@Input()` in Komponenten injizieren – kein `ActivatedRoute` mehr nötig. Aktiviere es mit `provideRouter(routes, withComponentInputBinding())`.
- **withPreloading(PreloadAllModules)**: Lazy-loaded Bundles im Hintergrund vorladen, sobald die initiale Route gerendert wurde.
- **Linked Signals & Router**: Kombiniere `toSignal(route.params)` mit `linkedSignal()` für reaktive, signal-basierte Routing-Logik ohne manuelle Subscriptions.
- Offizielle Docs: [https://angular.dev/guide/routing](https://angular.dev/guide/routing)
