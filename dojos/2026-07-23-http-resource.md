# Angular Dojo: httpResource() – Signal-basiertes HTTP-Datenladen
**Datum:** 2026-07-23
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, `httpResource()` aus `@angular/common/http` einzusetzen – den HTTP-spezifischen Resource-Helper, der `HttpClient`, Signals und automatische Request-Cancellation vereint und dabei deutlich weniger Boilerplate als `rxResource()` benötigt.

## Hintergrund & Theorie

Seit Angular 19.2 ergänzt `httpResource()` die generische `resource()`-API (bekannt aus dem Dojo vom 08.07.) um eine HTTP-native Variante. Statt einen generischen `loader` mit `HttpClient` zu verkabeln, übergibt man eine URL oder ein `HttpRequest`-Objekt direkt – Angular übernimmt den Rest.

**Kernmerkmale:**
- Direkte `HttpClient`-Integration: Interceptors, Auth-Header und Retry-Mechanismen greifen automatisch
- Reaktive URL-Parameter: ändert sich ein abhängiges Signal, wird der Request automatisch neu ausgelöst
- Automatische Cancellation: laufende Requests werden abgebrochen, sobald ein neuer Parameterwert vorliegt
- Typsicherheit: `httpResource<T>()` gibt ein `ResourceRef<T>` zurück
- Status-Signals: `.value()`, `.status()`, `.error()`, `.isLoading()`
- Varianten für Nicht-JSON-Responses: `httpResource.text()`, `httpResource.blob()`, `httpResource.arraybuffer()`

```typescript
const userId = signal(1);

const user = httpResource<User>(
  () => `/api/users/${userId()}`
);

// userId() ändert sich → laufender Request wird abgebrochen, neuer startet
```

`httpResource()` benötigt `provideHttpClient()` in der App-Konfiguration und ist seit Angular 19.2 als Developer Preview verfügbar.

## Aufgabe

Erstelle eine Komponente `GithubUserSearchComponent`, die:
1. Ein reaktives `username`-Signal als HTTP-Parameter nutzt
2. Die GitHub REST API abfragt: `https://api.github.com/users/{username}`
3. Lade-, Fehler- und Erfolgszustand im Template anzeigt
4. Benutzereingaben mit 300 ms Debounce verarbeitet, bevor ein Request gestartet wird

### Schritte
1. Erstelle `GithubUserSearchComponent` als Standalone Component und registriere `provideHttpClient()` in `app.config.ts`
2. Definiere ein `inputValue`-Signal für das aktuelle Texteingabe-Feld
3. Leite via `toSignal()` + `Subject` + `debounceTime(300)` ein geglättetes `username`-Signal ab
4. Binde `httpResource<GithubUser>(() => ...)` an das `username`-Signal
5. Zeige im Template: Ladeindikator (`.isLoading()`), Fehlermeldung (`.error()`), Avatar + Nutzerdaten (`.value()`)
6. Bonus: Deaktiviere den Request, wenn `username()` leer ist (return `undefined` im Loader)

## Hints

<details>
<summary>Hint 1 – Grundstruktur von httpResource()</summary>

```typescript
import { httpResource } from '@angular/common/http';

interface GithubUser {
  login: string;
  avatar_url: string;
  name: string | null;
  public_repos: number;
  followers: number;
}

// In der Komponente:
username = signal('angular');

userResource = httpResource<GithubUser>(
  () => `https://api.github.com/users/${this.username()}`
);

// Template:
// userResource.isLoading() → boolean
// userResource.error()     → unknown
// userResource.value()     → GithubUser | undefined
```

Die Loader-Funktion wird im Reaktivitätskontext ausgeführt. Jede Signal-Leseoperation (`this.username()`) registriert eine Abhängigkeit.
</details>

<details>
<summary>Hint 2 – Debounce mit Subject + toSignal</summary>

```typescript
import { Subject, debounceTime, distinctUntilChanged } from 'rxjs';
import { toSignal } from '@angular/core/rxjs-interop';

private inputSubject = new Subject<string>();

// Geglättetes Signal mit 300 ms Verzögerung:
username = toSignal(
  this.inputSubject.pipe(debounceTime(300), distinctUntilChanged()),
  { initialValue: 'angular' }
);

// Im Template (ngModelChange oder Input-Event):
onInput(value: string): void {
  this.inputSubject.next(value);
}
```
</details>

<details>
<summary>Hint 3 – Request unterdrücken (Bonus)</summary>

Gibt die Loader-Funktion `undefined` zurück, pausiert `httpResource()` und führt keinen Request aus:

```typescript
userResource = httpResource<GithubUser>(
  () => {
    const name = this.username();
    if (!name.trim()) return undefined; // kein Request
    return `https://api.github.com/users/${name}`;
  }
);
```
</details>

## Beispiellösung

```typescript
// github-user-search.component.ts
import { Component, signal } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { httpResource } from '@angular/common/http';
import { toSignal } from '@angular/core/rxjs-interop';
import { Subject, debounceTime, distinctUntilChanged } from 'rxjs';

interface GithubUser {
  login: string;
  avatar_url: string;
  name: string | null;
  public_repos: number;
  followers: number;
}

@Component({
  selector: 'app-github-user-search',
  standalone: true,
  imports: [FormsModule],
  template: `
    <input
      type="text"
      placeholder="GitHub-Username eingeben..."
      [ngModel]="inputValue()"
      (ngModelChange)="onInput($event)"
    />

    @if (userResource.isLoading()) {
      <p>Lade Benutzerdaten...</p>
    }

    @if (userResource.error()) {
      <p class="error">Benutzer „{{ username() }}" nicht gefunden.</p>
    }

    @if (userResource.value(); as user) {
      <div class="user-card">
        <img [src]="user.avatar_url" width="80" [alt]="user.login" />
        <div>
          <strong>{{ user.name ?? user.login }}</strong>
          <p>{{ user.public_repos }} Repos · {{ user.followers }} Follower</p>
        </div>
      </div>
    }
  `,
})
export class GithubUserSearchComponent {
  private inputSubject = new Subject<string>();

  inputValue = signal('angular');

  username = toSignal(
    this.inputSubject.pipe(debounceTime(300), distinctUntilChanged()),
    { initialValue: 'angular' }
  );

  userResource = httpResource<GithubUser>(() => {
    const name = this.username().trim();
    if (!name) return undefined;
    return `https://api.github.com/users/${name}`;
  });

  onInput(value: string): void {
    this.inputValue.set(value);
    this.inputSubject.next(value);
  }
}
```

```typescript
// app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withFetch } from '@angular/common/http';

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(withFetch()), // withFetch() empfohlen für SSR-Kompatibilität
  ],
};
```

**Erweiterung – POST-Request mit HttpRequest-Objekt:**

```typescript
import { HttpRequest } from '@angular/common/http';

payload = signal({ query: 'Angular' });

searchResource = httpResource<SearchResult[]>(
  () => new HttpRequest('POST', '/api/search', this.payload())
);
```

## Weiterführendes
- Nutze `httpResource.text()` für Plain-Text- und `httpResource.blob()` für Datei-Downloads – kein manuelles Response-Parsing nötig
- Interceptors (z. B. für Auth-Token) greifen automatisch, da `httpResource()` intern `HttpClient` verwendet
- Für optimistische Updates: Schreibe temporär in `.value` (via `ResourceRef.update()`), bevor der echte Request abgeschlossen ist
- [Angular Docs – httpResource](https://angular.dev/guide/signals/resource)
- [Angular Blog – HTTP Resource API](https://blog.angular.dev)
