# Angular Dojo: Resource API – Asynchrones Datenladen mit Signals
**Datum:** 2026-07-08
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, die neue Angular Resource API (`resource()` und `rxResource()`) zu verwenden, um asynchrone Daten reaktiv und typsicher mit Signals zu laden – ohne manuelle Subscription-Verwaltung.

## Hintergrund & Theorie

Seit Angular 19 gibt es die `resource()`-Funktion als experimentelles Feature (Stabilisierung in 20+). Sie kombiniert die Reaktivität von Signals mit asynchronem Datenladen und ersetzt typische Patterns wie `ngOnInit + HttpClient + Subject`.

**Kernprinzipien:**
- `resource()` akzeptiert eine `loader`-Funktion und einen optionalen `request`-Signal als Eingabe.
- Bei jeder Änderung des `request`-Signals wird der `loader` automatisch neu ausgeführt.
- Das Resource-Objekt liefert `.value()`, `.status()`, `.error()` und `.isLoading()` als Signals.
- `rxResource()` ist die RxJS-Variante: der `loader` gibt ein `Observable` zurück statt eines `Promise`.
- Laufende Requests werden automatisch abgebrochen, wenn sich die Eingabe ändert (ähnlich `switchMap`).

**Status-Werte:** `idle`, `loading`, `refreshing`, `resolved`, `error`, `local`.

```typescript
import { resource, linkedSignal, signal } from '@angular/core';

const userId = signal(1);
const userResource = resource({
  request: userId,
  loader: ({ request: id }) => fetch(`/api/users/${id}`).then(r => r.json())
});
```

## Aufgabe

Baue eine `UserProfileComponent`, die Benutzerdaten dynamisch per `resource()` (oder `rxResource()`) lädt. Die Komponente soll:

1. Eine Auswahlliste mit 3 User-IDs (1, 2, 3) anzeigen
2. Bei Auswahl einer ID den Benutzer von `https://jsonplaceholder.typicode.com/users/{id}` laden
3. Den Ladezustand mit einem Spinner/Text anzeigen
4. Den Benutzer-Namen, E-Mail und Firma darstellen
5. Fehler anzeigen, wenn der Request fehlschlägt
6. Einen "Refresh"-Button implementieren, der die Daten neu lädt

### Schritte

1. Erstelle eine `standalone` Komponente `UserProfileComponent` mit `inject(HttpClient)`.
2. Definiere ein `Signal<number>` für die ausgewählte User-ID (`signal(1)`).
3. Verwende `rxResource()` mit `HttpClient.get()` als `loader` (RxJS-kompatibel).
4. Binde `.isLoading()`, `.value()`, `.error()` und `.status()` im Template via `@if`-Blöcke.
5. Implementiere den "Refresh"-Button via `.reload()` auf dem Resource-Objekt.
6. Nutze `OnPush` Change Detection – Signals machen das trivial.

## Hints

<details>
<summary>Hint 1 – rxResource Grundstruktur</summary>

```typescript
import { rxResource } from '@angular/core/rxjs-interop';
import { HttpClient } from '@angular/common/http';
import { signal, inject, Component, ChangeDetectionStrategy } from '@angular/core';

@Component({
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  // ...
})
export class UserProfileComponent {
  private http = inject(HttpClient);
  selectedId = signal(1);

  userResource = rxResource({
    request: this.selectedId,
    loader: ({ request: id }) =>
      this.http.get<User>(`https://jsonplaceholder.typicode.com/users/${id}`)
  });
}
```

</details>

<details>
<summary>Hint 2 – Template-Bindings</summary>

```html
@if (userResource.isLoading()) {
  <p>Lädt...</p>
} @else if (userResource.error()) {
  <p>Fehler: {{ userResource.error() }}</p>
} @else if (userResource.value(); as user) {
  <h2>{{ user.name }}</h2>
  <p>{{ user.email }}</p>
  <p>{{ user.company.name }}</p>
}

<button (click)="userResource.reload()">Aktualisieren</button>
```

</details>

<details>
<summary>Hint 3 – Typ-Interface und ID-Auswahl</summary>

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  company: { name: string };
}

// Im Template:
// <select (change)="selectedId.set(+$event.target.value)">
//   @for (id of [1, 2, 3]; track id) {
//     <option [value]="id">User {{ id }}</option>
//   }
// </select>
```

</details>

## Beispiellösung

```typescript
import { Component, ChangeDetectionStrategy, signal, inject } from '@angular/core';
import { HttpClient, provideHttpClient } from '@angular/common/http';
import { rxResource } from '@angular/core/rxjs-interop';

interface User {
  id: number;
  name: string;
  email: string;
  phone: string;
  company: { name: string };
}

@Component({
  selector: 'app-user-profile',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <h1>User-Profil Viewer</h1>

    <label for="user-select">Benutzer wählen:</label>
    <select id="user-select" (change)="selectedId.set(+$any($event.target).value)">
      @for (id of userIds; track id) {
        <option [value]="id" [selected]="id === selectedId()">User {{ id }}</option>
      }
    </select>

    <button (click)="userResource.reload()">
      Aktualisieren
    </button>

    <div class="status">Status: {{ userResource.status() }}</div>

    @if (userResource.isLoading()) {
      <div class="spinner">⏳ Lade Benutzerdaten...</div>
    }

    @if (userResource.error()) {
      <div class="error">
        ❌ Fehler beim Laden: {{ userResource.error() }}
      </div>
    }

    @if (userResource.value(); as user) {
      <div class="user-card">
        <h2>{{ user.name }}</h2>
        <p>📧 {{ user.email }}</p>
        <p>📞 {{ user.phone }}</p>
        <p>🏢 {{ user.company.name }}</p>
      </div>
    }
  `,
  styles: [`
    .user-card { border: 1px solid #ccc; padding: 1rem; margin-top: 1rem; border-radius: 4px; }
    .spinner { color: #666; margin-top: 1rem; }
    .error { color: red; margin-top: 1rem; }
    .status { font-size: 0.8rem; color: #999; margin-top: 0.5rem; }
    select, button { margin: 0.5rem; padding: 0.25rem 0.5rem; }
  `]
})
export class UserProfileComponent {
  private http = inject(HttpClient);

  readonly userIds = [1, 2, 3];
  selectedId = signal(1);

  userResource = rxResource({
    request: this.selectedId,
    loader: ({ request: id }) =>
      this.http.get<User>(`https://jsonplaceholder.typicode.com/users/${id}`)
  });
}

// In main.ts oder app.config.ts:
// bootstrapApplication(UserProfileComponent, {
//   providers: [provideHttpClient()]
// });
```

**Bonus-Aufgabe:** Verwende `linkedSignal()`, um aus `userResource.value()` direkt einen
abgeleiteten `Signal<string>` für den vollständigen Anzeigenamen zu erstellen:

```typescript
displayName = linkedSignal(() => this.userResource.value()?.name ?? 'Unbekannt');
```

## Weiterführendes

- **`resource()` vs `rxResource()`:** `resource()` erwartet ein `Promise`, `rxResource()` ein `Observable` – bevorzuge `rxResource()` mit `HttpClient`, da es Cancellation (Abbruch laufender Requests) via `AbortController` respektiert.
- **Lokale Updates:** Mit `userResource.set(newValue)` oder `userResource.update(fn)` können Werte optimistisch lokal gesetzt werden (Status wird `local`).
- **Offizielle Docs:** [angular.dev/guide/signals/resource](https://angular.dev/guide/signals/resource)
- **Nächste Schritte:** Kombiniere `resource()` mit `computed()` für abgeleitete Zustände, oder nutze mehrere Resources mit gegenseitiger Abhängigkeit via `request: computed(() => ...)`.
