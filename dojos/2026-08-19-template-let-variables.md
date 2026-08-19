# Angular Dojo: `@let` Template-Variablen
**Datum:** 2026-08-19
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `@let` lokale Variablen direkt im Angular-Template deklarierst, um redundante Ausdrücke zu vermeiden, die Lesbarkeit zu verbessern und komplexe Typinferenz in Templates sauber zu handhaben.

## Hintergrund & Theorie
Angular 18 führte `@let` als offiziellen Teil der neuen Template-Syntax ein. Damit kannst du innerhalb eines Templates eine lokale Variable deklarieren, die im restlichen Template (unterhalb der Deklaration, im gleichen Block und in Kind-Blöcken) verfügbar ist.

```html
@let greeting = 'Hallo, ' + user.name + '!';
<h1>{{ greeting }}</h1>
```

**Wichtige Eigenschaften:**
- Eine `@let`-Variable ist **read-only** – sie kann nicht aus der Komponente heraus gesetzt werden.
- Sie wird bei jedem Change-Detection-Zyklus neu ausgewertet.
- Der Wert kann jeder Template-Ausdruck sein: Strings, Objekte, Ergebnisse von Pipes, Async-Ausdrücke, Signals – alles ohne `.value`-Aufruf.
- `@let` ist **scoped**: Innerhalb von `@if`, `@for`, `@switch`-Blöcken ist die Variable nur innerhalb dieses Blocks sichtbar.

**Typischer Anwendungsfall:** Async-Pipes oder Signals, die mehrfach im Template referenziert werden, sauber einer einzigen Variable zuweisen, anstatt die Pipe/Signal-Expression zu wiederholen.

```html
@let user = currentUser$ | async;
@if (user) {
  <p>{{ user.name }}</p>
  <p>{{ user.email }}</p>
}
```

## Aufgabe
Baue eine `UserDashboardComponent`, die Daten von zwei Observables kombiniert und `@let` nutzt, um das Template sauber und typ-sicher zu halten.

### Schritte
1. Erstelle eine `UserDashboardComponent` als Standalone-Komponente.
2. Injiziere einen `UserService` (mock), der zwei Observables bereitstellt:
   - `currentUser$: Observable<{ name: string; role: 'admin' | 'viewer' }>`
   - `stats$: Observable<{ totalItems: number; lastLogin: Date }>`
3. Kombiniere beide Observables im Service oder in der Komponente mit `combineLatest`.
4. Nutze im Template `@let` + `async`-Pipe, um die Daten einmalig zu entpacken.
5. Zeige bedingt einen Admin-Bereich (mit einem extra `@let` für den formatierten Datumswert) und einen Viewer-Bereich an.
6. Nutze zusätzlich `@let` für einen berechneten Begrüßungstext, der den Usernamen und die Rolle kombiniert.

**Bonus:** Schreibe eine `OnPush`-Komponente und überzeuge dich davon, dass `@let` mit `OnPush` und Signals genauso funktioniert.

### Schritte
1. `ng generate component user-dashboard --standalone`
2. Mock-Service mit `of(...)` oder `BehaviorSubject` bauen.
3. Template mit `@let` für `async`-Pipe-Ergebnis und berechnete Variablen schreiben.
4. `@if (userData)` mit einem inneren `@let lastLoginFormatted = ...` ergänzen.
5. Prüfen, dass der Template-Compiler Typen korrekt inferiert (kein `any`, kein Non-null-Assertion nötig).

## Hints
<details>
<summary>Hint 1 – async-Pipe mit @let</summary>

Statt das Ergebnis der `async`-Pipe in ein `*ngIf="... as foo"` zu verpacken, schreibe:

```html
@let data = (userData$ | async);
@if (data) {
  <!-- data ist hier garantiert non-null -->
  <p>{{ data.user.name }}</p>
}
```

Das erspart die doppelte Typ-Assertion und ist im neuen Control Flow lesbarer.
</details>

<details>
<summary>Hint 2 – Scoping von @let</summary>

`@let` ist auf den Block beschränkt, in dem es deklariert wird. Deklariere es **oberhalb** des `@if`-Blocks, wenn du es danach noch benötigst. Innerhalb eines `@for`-Blocks hast du Zugriff auf Loop-Variablen wie `$index` und kannst sie mit `@let` kombinieren:

```html
@for (item of items; track item.id; let i = $index) {
  @let label = i + 1 + '. ' + item.title;
  <li>{{ label }}</li>
}
```
</details>

## Beispiellösung
```typescript
// user-dashboard.component.ts
import { Component, inject, ChangeDetectionStrategy } from '@angular/core';
import { AsyncPipe, DatePipe } from '@angular/common';
import { combineLatest } from 'rxjs';
import { UserService } from './user.service';

@Component({
  selector: 'app-user-dashboard',
  standalone: true,
  imports: [AsyncPipe, DatePipe],
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    @let data = (userData$ | async);

    @if (data) {
      @let greeting = 'Willkommen, ' + data.user.name + ' (' + data.user.role + ')';
      @let lastLogin = data.stats.lastLogin | date:'medium';

      <h1>{{ greeting }}</h1>
      <p>Letzter Login: {{ lastLogin }}</p>
      <p>Gesamt-Items: {{ data.stats.totalItems }}</p>

      @if (data.user.role === 'admin') {
        @let adminInfo = 'Admin-Bereich – ' + data.stats.totalItems + ' Einträge verwaltbar';
        <section class="admin-panel">
          <h2>{{ adminInfo }}</h2>
          <button>Einträge verwalten</button>
        </section>
      } @else {
        <section class="viewer-panel">
          <p>Du hast nur Lesezugriff.</p>
        </section>
      }
    } @else {
      <p>Lade Benutzerdaten…</p>
    }
  `,
})
export class UserDashboardComponent {
  private userService = inject(UserService);

  userData$ = combineLatest({
    user: this.userService.currentUser$,
    stats: this.userService.stats$,
  });
}
```

```typescript
// user.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject, of } from 'rxjs';
import { delay } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class UserService {
  currentUser$ = of({ name: 'Alex Müller', role: 'admin' as const }).pipe(delay(500));
  stats$ = of({ totalItems: 42, lastLogin: new Date() }).pipe(delay(300));
}
```

## Weiterführendes
- **Offizielle Docs:** [angular.dev/guide/templates/let-template-variables](https://angular.dev/guide/templates/let-template-variables) – vollständige Spezifikation inkl. Scoping-Regeln
- **Kombination mit Signals:** `@let count = mySignal();` funktioniert genauso – Angular tracked die Dependency automatisch, kein `.value`-Aufruf nötig im Template.
- **Typ-Narrowing:** Der Compiler narrowt den Typ innerhalb eines `@if (data)` korrekt, sodass nach der Null-Prüfung auf alle Properties typsicher zugegriffen werden kann – ein großer Vorteil gegenüber der alten `*ngIf="x as y"`-Syntax.
