# Angular Dojo: Named Router Outlets & Auxiliary Routes
**Datum:** 2026-08-26
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mehrere benannte `<router-outlet>`-Elemente gleichzeitig nutzt, um parallele Navigation in einem Layout zu ermöglichen – z. B. für persistente Seitenleisten, Detailpanels oder modale Ansichten, die unabhängig vom Hauptinhalt navigierbar sind.

## Hintergrund & Theorie

Angular erlaubt es, mehrere Router-Outlets in einer Komponente zu haben. Neben dem primären (namenlosen) Outlet können **benannte Outlets** (Named Outlets) über das `outlet`-Property einer Route definiert werden:

```ts
{ path: 'detail', component: DetailComponent, outlet: 'side' }
```

Im Template entspricht das einem Outlet mit dem passenden Namen:

```html
<router-outlet name="side"></router-outlet>
```

Solche Routen werden als **Auxiliary Routes** bezeichnet. Sie erscheinen in der URL als eigenständiges Segment:

```
/dashboard(side:detail)
```

Das bedeutet: Primäre Route `/dashboard` läuft im primären Outlet, gleichzeitig läuft `detail` im Outlet `side`.

Navigation zu Auxiliary Routes erfolgt über ein `outlets`-Objekt im `routerLink` oder per `Router.navigate()`:

```ts
router.navigate([{ outlets: { side: ['detail'] } }]);
```

Zum **Schließen** wird `null` übergeben:

```ts
router.navigate([{ outlets: { side: null } }]);
```

Auxiliary Routes sind ideal für Slide-over-Panels, Tabs, Preview-Fenster oder alle UI-Elemente, die parallel zur Hauptnavigation existieren sollen, ohne den primären Zustand zu verlieren.

## Aufgabe

Baue eine kleine Angular-App mit einem persistenten Seitenpanel, das unabhängig vom Hauptinhalt navigierbar ist.

### Schritte

1. **Routen konfigurieren:** Definiere primäre Routen (`/home`, `/about`) und eine Auxiliary Route mit `outlet: 'panel'` für eine `ProfileComponent`.

2. **Template aufbauen:** Füge in `AppComponent` zwei Outlets ein: das primäre (namenlos) und ein benanntes mit `name="panel"`.

3. **Navigation implementieren:** Erstelle Buttons, die
   - das Haupt-Outlet zu `/home` oder `/about` navigieren,
   - das Panel-Outlet zu `profile` öffnen und wieder schließen.

4. **URL beobachten:** Prüfe in der Adressleiste, wie die URL bei kombinierter Navigation aussieht (z. B. `/home(panel:profile)`).

5. **Guard hinzufügen (Bonus):** Schütze die Panel-Route mit einem funktionalen `canActivate`-Guard, der prüft, ob der User eingeloggt ist (simuliere es mit einem `AuthService`, der ein Signal hält).

### Schritte – Details

```
AppComponent
├── <nav> (Links für primäre und panel-Navigation)
├── <router-outlet>          ← primary
└── <router-outlet name="panel">  ← auxiliary
```

Routenkonfiguration:
```
/home      → HomeComponent      (primary)
/about     → AboutComponent     (primary)
profile    → ProfileComponent   (outlet: 'panel')
```

## Hints

<details>
<summary>Hint 1 – Routenkonfiguration</summary>

```ts
export const routes: Routes = [
  { path: '', redirectTo: 'home', pathMatch: 'full' },
  { path: 'home', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'profile', component: ProfileComponent, outlet: 'panel' },
];
```

Beachte: Der `path` einer Auxiliary Route muss kein `/` enthalten. Er ist relativ zum Router-Outlets, nicht zum URL-Segment.

</details>

<details>
<summary>Hint 2 – Navigation mit outlets-Objekt</summary>

```ts
// Panel öffnen
this.router.navigate([{ outlets: { panel: ['profile'] } }]);

// Panel schließen
this.router.navigate([{ outlets: { panel: null } }]);

// In einer Komponente, die auf der Route /home ist:
// Beide gleichzeitig setzen
this.router.navigate(['home', { outlets: { panel: ['profile'] } }]);
```

Als `routerLink`-Direktive:
```html
<a [routerLink]="[{ outlets: { panel: ['profile'] } }]">Profil öffnen</a>
<a [routerLink]="[{ outlets: { panel: null } }]">Profil schließen</a>
```

</details>

<details>
<summary>Hint 3 – Bonus Guard mit Signal</summary>

```ts
// auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  readonly isLoggedIn = signal(false);
  login() { this.isLoggedIn.set(true); }
  logout() { this.isLoggedIn.set(false); }
}

// auth.guard.ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  return auth.isLoggedIn() || inject(Router).createUrlTree(['/home']);
};

// In der Route:
{ path: 'profile', component: ProfileComponent, outlet: 'panel', canActivate: [authGuard] }
```

</details>

## Beispiellösung

```typescript
// app.routes.ts
import { Routes } from '@angular/router';
import { HomeComponent } from './home.component';
import { AboutComponent } from './about.component';
import { ProfileComponent } from './profile.component';

export const routes: Routes = [
  { path: '', redirectTo: 'home', pathMatch: 'full' },
  { path: 'home', component: HomeComponent },
  { path: 'about', component: AboutComponent },
  { path: 'profile', component: ProfileComponent, outlet: 'panel' },
];

// app.component.ts
import { Component, inject } from '@angular/core';
import { Router, RouterLink, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink],
  template: `
    <nav>
      <a routerLink="/home">Home</a> |
      <a routerLink="/about">About</a> |
      <button (click)="openPanel()">Profil öffnen</button>
      <button (click)="closePanel()">Profil schließen</button>
    </nav>

    <div class="layout">
      <main>
        <router-outlet />
      </main>
      <aside>
        <router-outlet name="panel" />
      </aside>
    </div>
  `,
  styles: [`
    .layout { display: flex; gap: 1rem; }
    main { flex: 1; }
    aside { width: 300px; border-left: 1px solid #ccc; padding: 1rem; }
  `]
})
export class AppComponent {
  private router = inject(Router);

  openPanel(): void {
    this.router.navigate([{ outlets: { panel: ['profile'] } }]);
  }

  closePanel(): void {
    this.router.navigate([{ outlets: { panel: null } }]);
  }
}

// profile.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-profile',
  standalone: true,
  template: `
    <h2>Benutzerprofil</h2>
    <p>Name: Max Mustermann</p>
    <p>E-Mail: max@example.com</p>
  `
})
export class ProfileComponent {}
```

**Ergebnis-URL bei offenem Panel auf der Home-Route:**
```
http://localhost:4200/home(panel:profile)
```

Das Panel bleibt beim Wechsel zwischen `/home` und `/about` geöffnet, da es ein eigenes, unabhängiges Outlet hat.

## Weiterführendes

- **Mehrere Auxiliary Outlets:** Eine App kann beliebig viele benannte Outlets haben – z. B. `chat`, `notifications`, `detail` – alle unabhängig navigierbar in einer URL.
- **State-Synchronisation:** Nutze `Router.events` und filtere `NavigationEnd`, um auf Outlet-Zustandsänderungen zu reagieren und bspw. den `isLoggedIn`-Guard-State mit der URL zu synchronisieren.
- **Angular Docs:** [Named Outlets](https://angular.dev/guide/routing/secondary-routes) – die offizielle Dokumentation zu Secondary/Auxiliary Routes.
- **Tipp:** Auxiliary Routes eignen sich hervorragend für Split-View-Layouts (Master-Detail), wo beide Seiten unabhängig navigierbar sein sollen – in Kombination mit `@defer` kann das Detail-Panel sogar lazy geladen werden.
