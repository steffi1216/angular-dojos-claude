# Angular Dojo: NgModule-zu-Standalone Migration
**Datum:** 2026-11-16
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du eine bestehende Angular-Anwendung systematisch von der NgModule-Architektur auf Standalone Components migrierst – sowohl manuell als auch mit dem offiziellen Migrations-Schematic.

## Hintergrund & Theorie
Seit Angular 15 sind Standalone Components stabil und empfohlen. Statt `NgModule` deklarieren Standalone Components ihre Abhängigkeiten direkt im `@Component`-Decorator über das `imports`-Array. Das Resultat: weniger Boilerplate, kürzere Lernkurve, besseres Tree-Shaking.

Eine Migration läuft typischerweise in drei Phasen ab:

**Phase 1 – Komponenten standalone machen:** `standalone: true` im `@Component`-Decorator setzen, benötigte Importe direkt in die Komponente verschieben.

**Phase 2 – Module ausdünnen:** Wenn alle Komponenten eines Moduls standalone sind, kann das Modul entfernt werden. Services wandern zu `providedIn: 'root'` oder in `app.config.ts`.

**Phase 3 – Bootstrapping umstellen:** `bootstrapModule(AppModule)` wird durch `bootstrapApplication(AppComponent, appConfig)` ersetzt; Router, HTTP-Client und andere Provider landen in `app.config.ts`.

Angular stellt das Schematic `@angular/core:standalone` bereit, das den Großteil dieser Arbeit automatisiert. Das Verständnis der manuellen Schritte ist trotzdem essenziell, um Migrationsartefakte zu verstehen und Grenzfälle zu lösen.

## Aufgabe
Du hast eine kleine NgModule-basierte App mit `AppModule`, `UserModule`, `UserListComponent` und `UserDetailComponent`. Migriere sie schrittweise auf Standalone.

### Ausgangslage (NgModule)

```typescript
// user.module.ts
@NgModule({
  declarations: [UserListComponent, UserDetailComponent],
  imports: [CommonModule, RouterModule, SharedModule],
  exports: [UserListComponent],
})
export class UserModule {}

// app.module.ts
@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule, RouterModule.forRoot(routes), UserModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
```

### Schritte

1. **UserListComponent standalone machen**
   - Setze `standalone: true` im `@Component`-Decorator.
   - Füge `imports: [CommonModule, RouterModule]` direkt zur Komponente hinzu.
   - Entferne `UserListComponent` aus `UserModule.declarations`.

2. **UserDetailComponent standalone machen**
   - Wiederhole den Vorgang für `UserDetailComponent`.
   - Importiere nur die Direktiven/Pipes, die diese Komponente wirklich braucht (z. B. `AsyncPipe`, `RouterLink` statt ganzer Module).

3. **UserModule entfernen**
   - Da nun keine Komponenten mehr im Modul deklariert sind, lösche `user.module.ts`.
   - Passe die Routen-Konfiguration an: Referenziere `UserListComponent` und `UserDetailComponent` direkt.

4. **AppComponent standalone machen & Bootstrapping umstellen**
   - Mache `AppComponent` standalone.
   - Erstelle `app.config.ts` mit `ApplicationConfig`:
     ```typescript
     export const appConfig: ApplicationConfig = {
       providers: [
         provideRouter(routes),
         provideHttpClient(),
       ],
     };
     ```
   - Ändere `main.ts` von `bootstrapModule(AppModule)` auf `bootstrapApplication(AppComponent, appConfig)`.
   - Lösche `app.module.ts`.

5. **Schematic als Vergleich ausführen (optional)**
   ```bash
   ng generate @angular/core:standalone --mode=convert-to-standalone
   ng generate @angular/core:standalone --mode=prune-ng-modules
   ng generate @angular/core:standalone --mode=standalone-bootstrap
   ```

## Hints

<details>
<summary>Hint 1 – Welche Importe braucht eine standalone Component?</summary>

Ersetze Module durch ihre konkreten Direktiven und Pipes. Statt `CommonModule` importiere nur das, was du wirklich nutzt:

```typescript
import { NgIf, NgFor, AsyncPipe, DatePipe } from '@angular/common';

@Component({
  standalone: true,
  imports: [NgIf, NgFor, AsyncPipe, RouterLink],
})
export class UserListComponent {}
```

Das verbessert das Tree-Shaking erheblich, weil der Bundler ungenutzte Teile weglassen kann.

</details>

<details>
<summary>Hint 2 – Lazy-Loading mit Standalone Components</summary>

Standalone Components können direkt lazy-geladen werden, ohne ein Wrapper-Modul:

```typescript
// routes.ts
export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () =>
      import('./user/user-list.component').then(m => m.UserListComponent),
  },
  {
    path: 'users/:id',
    loadChildren: () =>
      import('./user/user.routes').then(m => m.USER_ROUTES),
  },
];
```

`loadChildren` mit einem Array von Routen (kein Modul!) ist der moderne Weg für ganze Feature-Bereiche.

</details>

<details>
<summary>Hint 3 – Services und Providers</summary>

Services mit `providedIn: 'root'` funktionieren unverändert. Feature-spezifische Providers, die vorher im `NgModule.providers`-Array standen, kommen jetzt in die Route-Konfiguration:

```typescript
{
  path: 'admin',
  loadComponent: () => import('./admin.component'),
  providers: [AdminService, { provide: ADMIN_CONFIG, useValue: config }],
}
```

Oder für ganze Feature-Bereiche in `loadChildren` mit einer separaten Provider-Liste.

</details>

## Beispiellösung

```typescript
// main.ts (nach der Migration)
import { bootstrapApplication } from '@angular/platform-browser';
import { AppComponent } from './app/app.component';
import { appConfig } from './app/app.config';

bootstrapApplication(AppComponent, appConfig).catch(console.error);

// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { provideHttpClient, withFetch } from '@angular/common/http';
import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withComponentInputBinding()),
    provideHttpClient(withFetch()),
  ],
};

// app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `<router-outlet />`,
})
export class AppComponent {}

// user-list.component.ts
import { Component, inject } from '@angular/core';
import { AsyncPipe, NgFor } from '@angular/common';
import { RouterLink } from '@angular/router';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-list',
  standalone: true,
  imports: [AsyncPipe, NgFor, RouterLink],
  template: `
    <ul>
      @for (user of users$ | async; track user.id) {
        <li><a [routerLink]="['/users', user.id]">{{ user.name }}</a></li>
      }
    </ul>
  `,
})
export class UserListComponent {
  private userService = inject(UserService);
  users$ = this.userService.getAll();
}

// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () =>
      import('./user/user-list.component').then(m => m.UserListComponent),
  },
  {
    path: 'users/:id',
    loadComponent: () =>
      import('./user/user-detail.component').then(m => m.UserDetailComponent),
  },
  { path: '', redirectTo: 'users', pathMatch: 'full' },
];
```

## Weiterführendes
- Führe das Migrations-Schematic in einem echten Projekt aus (`--mode=convert-to-standalone`, dann `--mode=prune-ng-modules`, dann `--mode=standalone-bootstrap`) und prüfe anschließend mit `ng lint` und `ng build`, ob alles sauber ist.
- Lies die offizielle Migrationsanleitung: [angular.dev/reference/migrations/standalone](https://angular.dev/reference/migrations/standalone)
- Für Bibliotheken gilt eine eigene Strategie: Prüfe, ob du `secondary entry points` und `NgModule` als Kompatibilitätsschicht noch brauchst.
