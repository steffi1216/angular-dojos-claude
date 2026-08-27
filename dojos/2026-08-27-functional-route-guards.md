# Angular Dojo: Functional Route Guards
**Datum:** 2026-08-27
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Lerne, wie du in Angular 15+ Route Guards als einfache Funktionen schreibst – ohne Klassen und `implements CanActivate`. Verstehe dabei die Kombination mit `inject()`, `CanActivateFn`, `CanDeactivateFn` und `CanMatchFn`.

## Hintergrund & Theorie

Bis Angular 14 wurden Route Guards als Klassen implementiert, die Interfaces wie `CanActivate` oder `CanDeactivate` implementierten und als Services registriert werden mussten. Ab Angular 15 sind Guards als einfache Funktionen (`CanActivateFn`, `CanDeactivateFn`, `CanMatchFn`) bevorzugt – die Klassen-basierten Guards sind seit Angular 15.1 als **deprecated** markiert.

**Vorteile funktionaler Guards:**
- Kein Boilerplate (keine Klasse, kein `Injectable`, kein `providedIn`)
- Nutzung von `inject()` innerhalb der Funktion für DI
- Besser testbar und tree-shakeable
- Komponierfähig: Guards lassen sich mit `pipe()` oder einem Helper kombinieren

**Wichtige Typen:**
```typescript
type CanActivateFn = (route: ActivatedRouteSnapshot, state: RouterStateSnapshot) 
  => Observable<boolean|UrlTree> | Promise<boolean|UrlTree> | boolean | UrlTree;

type CanDeactivateFn<T> = (component: T, ...) => ...;

type CanMatchFn = (route: Route, segments: UrlSegment[]) => ...;
```

`inject()` ist innerhalb eines Guards direkt nutzbar, weil Guards im **Injection Context** aufgerufen werden.

## Aufgabe

Baue ein Mini-Angular-Routing-Setup mit drei Guards:

1. **`authGuard`** – prüft, ob ein User eingeloggt ist (via `AuthService`). Nicht eingeloggte User werden auf `/login` umgeleitet.
2. **`roleGuard`** – prüft, ob der User eine bestimmte Rolle besitzt (Rolle wird als `data`-Property der Route übergeben). Fehlende Rolle → Redirect auf `/forbidden`.
3. **`unsavedChangesGuard`** – ein `CanDeactivateFn`, das ein Interface `HasUnsavedChanges` überprüft und den User bei ungespeicherten Änderungen um Bestätigung bittet.

### Schritte

1. Erstelle einen `AuthService` mit einem `isLoggedIn`-Signal und einer `userRole`-Property.
2. Implementiere `authGuard` als `CanActivateFn`-Funktion – nutze `inject(AuthService)` und `inject(Router)`.
3. Implementiere `roleGuard` als `CanActivateFn`-Funktion – lies die erforderliche Rolle aus `route.data['role']`.
4. Definiere das Interface `HasUnsavedChanges` und implementiere `unsavedChangesGuard` als `CanDeactivateFn<HasUnsavedChanges>`.
5. Registriere alle Guards in der Route-Konfiguration (ohne sie als Provider anzumelden – das ist der Clou!).
6. Schreibe einen Unit-Test für `authGuard`, der `TestBed.runInInjectionContext()` verwendet.

## Hints

<details>
<summary>Hint 1 – inject() in Guards</summary>

Du kannst `inject()` direkt im Funktionskörper eines Guards verwenden, weil Angular den Guard im Injection Context ausführt:

```typescript
export const authGuard: CanActivateFn = (route, state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.createUrlTree(['/login']);
};
```

Kein `constructor`, kein `@Injectable()` notwendig!
</details>

<details>
<summary>Hint 2 – CanDeactivateFn mit Interface</summary>

Definiere ein Interface, das deine Komponente implementiert, und nutze es als Typparameter:

```typescript
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('Änderungen verwerfen?');
  }
  return true;
};
```

In der Route: `canDeactivate: [unsavedChangesGuard]`
</details>

<details>
<summary>Hint 3 – Unit-Test mit runInInjectionContext</summary>

```typescript
it('should redirect when not logged in', () => {
  TestBed.configureTestingModule({
    providers: [provideRouter([]), AuthService],
  });

  const result = TestBed.runInInjectionContext(() =>
    authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
  );

  expect(result).toBeInstanceOf(UrlTree);
});
```

`TestBed.runInInjectionContext()` simuliert den Injection Context, in dem Guards ausgeführt werden.
</details>

## Beispiellösung

```typescript
// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  isLoggedIn = signal(false);
  userRole = signal<'admin' | 'user' | null>(null);

  login(role: 'admin' | 'user') {
    this.isLoggedIn.set(true);
    this.userRole.set(role);
  }

  logout() {
    this.isLoggedIn.set(false);
    this.userRole.set(null);
  }
}

// auth.guard.ts
export const authGuard: CanActivateFn = (_route, _state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isLoggedIn() ? true : router.createUrlTree(['/login']);
};

// role.guard.ts
export const roleGuard: CanActivateFn = (route, _state) => {
  const auth = inject(AuthService);
  const router = inject(Router);
  const requiredRole = route.data['role'] as 'admin' | 'user';

  if (auth.userRole() === requiredRole) {
    return true;
  }
  return router.createUrlTree(['/forbidden']);
};

// unsaved-changes.guard.ts
export interface HasUnsavedChanges {
  hasUnsavedChanges(): boolean;
}

export const unsavedChangesGuard: CanDeactivateFn<HasUnsavedChanges> = (component) => {
  if (component.hasUnsavedChanges()) {
    return confirm('Du hast ungespeicherte Änderungen. Wirklich verlassen?');
  }
  return true;
};

// app.routes.ts
export const routes: Routes = [
  { path: 'login', component: LoginComponent },
  { path: 'forbidden', component: ForbiddenComponent },
  {
    path: 'dashboard',
    component: DashboardComponent,
    canActivate: [authGuard],        // Einfache Funktion – kein Provider nötig!
  },
  {
    path: 'admin',
    component: AdminComponent,
    canActivate: [authGuard, roleGuard],
    data: { role: 'admin' },
  },
  {
    path: 'editor',
    component: EditorComponent,
    canActivate: [authGuard],
    canDeactivate: [unsavedChangesGuard],
  },
];

// auth.guard.spec.ts
describe('authGuard', () => {
  it('gibt true zurück wenn eingeloggt', () => {
    TestBed.configureTestingModule({
      providers: [provideRouter([]), AuthService],
    });
    TestBed.inject(AuthService).login('user');

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
    );
    expect(result).toBe(true);
  });

  it('leitet auf /login weiter wenn nicht eingeloggt', () => {
    TestBed.configureTestingModule({
      providers: [provideRouter([]), AuthService],
    });

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as ActivatedRouteSnapshot, {} as RouterStateSnapshot)
    );
    expect(result).toBeInstanceOf(UrlTree);
    expect((result as UrlTree).toString()).toBe('/login');
  });
});
```

## Weiterführendes

- **Guards kombinieren mit `pipe()`-Helper**: Nutze einen `combineGuards(...fns)` Helper, der mehrere `CanActivateFn`s zu einer zusammenfasst – nützlich für wiederverwendbare Guard-Kombinationen.
- **Offiziell dokumentiert**: [angular.dev/guide/routing/common-router-tasks#preventing-unauthorized-access](https://angular.dev/guide/routing/common-router-tasks) für die aktuellen Empfehlungen zu funktionalen Guards.
- **`CanMatchFn`** ist besonders interessant für Lazy Loading: Der Guard entscheidet, ob ein Lazy-Route-Match überhaupt versucht wird – noch bevor der Bundle geladen wird.
