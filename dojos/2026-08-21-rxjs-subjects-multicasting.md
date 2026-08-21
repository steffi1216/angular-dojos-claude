# Angular Dojo: RxJS Subjects & Multicasting
**Datum:** 2026-08-21
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du verstehst die Unterschiede zwischen `Subject`, `BehaviorSubject`, `ReplaySubject` und `AsyncSubject` und weißt, wann du welchen einsetzt. Du lernst außerdem, wie `shareReplay` Observables multicastet und teure HTTP-Aufrufe vermeidet.

## Hintergrund & Theorie
Normale RxJS-Observables sind **kalt (cold)** – jede Subscription startet eine eigene unabhängige Ausführung (z. B. ein separater HTTP-Request). **Subjects** sind hingegen **heiß (hot)**: Sie fungieren gleichzeitig als Observable *und* als Observer. Mehrere Subscriber teilen denselben Datenstrom.

| Typ | Startwert | Replay für neue Subscriber |
|---|---|---|
| `Subject` | keiner | keiner (verpasste Werte gehen verloren) |
| `BehaviorSubject` | erforderlich | letzter Wert |
| `ReplaySubject(n)` | keiner | letzte *n* Werte |
| `AsyncSubject` | keiner | letzter Wert **nach** `complete()` |

**`shareReplay(1)`** verwandelt ein kaltes Observable in ein heißes: Es teilt die Ausführung unter allen Subscribern und speichert den letzten emittierten Wert für Späteinsteigende. Das ist der idiomatische Weg, teure Operationen (HTTP, komplexe Berechnungen) zu cachen.

Typische Einsatzgebiete:
- `BehaviorSubject` → Zustandsverwaltung (aktueller User, Theme, Filter)
- `ReplaySubject` → Event-Bus, bei dem neue Subscriber den Verlauf brauchen
- `shareReplay(1)` → gecachte HTTP-Requests in Services

## Aufgabe
Baue einen **Notification Service**, der Benachrichtigungen über einen `ReplaySubject` verbreitet, und einen **User Service**, der den aktuell eingeloggten User als `BehaviorSubject` hält. Zeige beide in einer Komponente an und optimiere einen HTTP-Aufruf mit `shareReplay`.

### Schritte

1. **NotificationService** erstellen:
   - `ReplaySubject<string>(5)` als private Property (letzte 5 Notifications merken)
   - Methode `notify(message: string)` zum Emittieren
   - Property `notifications$` als publisches Observable (per `.asObservable()`)

2. **UserService** erstellen:
   - `BehaviorSubject<User | null>(null)` für den aktuellen User
   - Methode `login(user: User)` und `logout()`
   - Property `currentUser$` als publisches Observable
   - HTTP-Aufruf `getProfile(userId: string)` mit `shareReplay(1)` cachen (simuliere mit `HttpClient.get`)

3. **DashboardComponent** (standalone) erstellen:
   - Injiziere beide Services
   - Zeige alle Notifications mit `| async` im Template an
   - Zeige den aktuellen User-Namen an (oder „Gast" wenn `null`)
   - Füge zwei Buttons ein: „Login simulieren" und „Notification senden"
   - Abonniere `getProfile` **zweimal** und logge in der Console, dass nur **ein** HTTP-Request stattfindet

4. **Bonus:** Ersetze `| async` durch `toSignal()` und beobachte, wie sich das Teardown-Verhalten ändert.

## Hints

<details>
<summary>Hint 1 – Subject vs. Observable</summary>

Ein Subject ist direkt beschreibbar. Nutze `.next()` zum Emittieren und `.asObservable()`, um das Subject nach außen nur als Observable zu exponieren – so kann kein externer Code unbeabsichtigt Werte emittieren:

```typescript
private readonly _state$ = new BehaviorSubject<string>('initial');
readonly state$ = this._state$.asObservable();

update(val: string) { this._state$.next(val); }
```
</details>

<details>
<summary>Hint 2 – shareReplay für HTTP-Caching</summary>

Ohne `shareReplay` startet jede Subscription einen neuen HTTP-Request. Mit `shareReplay(1)` teilen alle Subscriber denselben Request und erhalten sofort den gecachten Wert, wenn sie sich später subscriben:

```typescript
private profile$ = this.http.get<Profile>('/api/profile').pipe(
  shareReplay(1)
);

// Beide Subscriptions: nur EIN HTTP-Request
this.profile$.subscribe(p => console.log('Sub 1', p));
this.profile$.subscribe(p => console.log('Sub 2', p));
```

Achtung: `shareReplay({ bufferSize: 1, refCount: true })` beendet das Multicasting, wenn kein Subscriber mehr existiert – sinnvoll, um Memory Leaks zu vermeiden.
</details>

<details>
<summary>Hint 3 – ReplaySubject Initialwert für Späteinsteigende</summary>

Im Gegensatz zu `Subject` erhalten Subscriber eines `ReplaySubject(n)` bis zu *n* vergangene Werte sofort beim Subscribe:

```typescript
const replay$ = new ReplaySubject<string>(3);
replay$.next('A');
replay$.next('B');
replay$.next('C');
replay$.next('D');

// Neuer Subscriber erhält sofort: B, C, D (die letzten 3)
replay$.subscribe(v => console.log(v));
```
</details>

## Beispiellösung

```typescript
// models/user.model.ts
export interface User {
  id: string;
  name: string;
}

// services/notification.service.ts
import { Injectable } from '@angular/core';
import { ReplaySubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  private readonly _notifications$ = new ReplaySubject<string>(5);
  readonly notifications$ = this._notifications$.asObservable();

  notify(message: string): void {
    this._notifications$.next(message);
  }
}

// services/user.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { BehaviorSubject } from 'rxjs';
import { shareReplay } from 'rxjs/operators';
import { User } from '../models/user.model';

@Injectable({ providedIn: 'root' })
export class UserService {
  private readonly http = inject(HttpClient);
  private readonly _currentUser$ = new BehaviorSubject<User | null>(null);
  readonly currentUser$ = this._currentUser$.asObservable();

  // Gecachter HTTP-Aufruf: nur ein Request, egal wie viele Subscriber
  private readonly profile$ = this.http
    .get<User>('/api/profile/1')
    .pipe(shareReplay({ bufferSize: 1, refCount: true }));

  login(user: User): void {
    this._currentUser$.next(user);
  }

  logout(): void {
    this._currentUser$.next(null);
  }

  getProfile() {
    return this.profile$;
  }
}

// components/dashboard.component.ts
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';
import { NotificationService } from '../services/notification.service';
import { UserService } from '../services/user.service';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h2>Dashboard</h2>

    <p>Eingeloggt als: <strong>{{ userName() }}</strong></p>

    <button (click)="login()">Login simulieren</button>
    <button (click)="sendNotification()">Notification senden</button>

    <h3>Benachrichtigungen</h3>
    <ul>
      @for (msg of notifications(); track msg) {
        <li>{{ msg }}</li>
      }
    </ul>
  `,
})
export class DashboardComponent {
  private readonly notificationService = inject(NotificationService);
  private readonly userService = inject(UserService);

  readonly notifications = toSignal(this.notificationService.notifications$, {
    initialValue: [] as string[],
  });

  readonly userName = toSignal(
    this.userService.currentUser$.pipe(map(u => u?.name ?? 'Gast')),
    { initialValue: 'Gast' }
  );

  constructor() {
    // Zwei Subscriptions – dank shareReplay nur ein HTTP-Request
    this.userService.getProfile().subscribe(p => console.log('Sub 1:', p));
    this.userService.getProfile().subscribe(p => console.log('Sub 2:', p));
  }

  login(): void {
    this.userService.login({ id: '1', name: 'Ada Lovelace' });
  }

  sendNotification(): void {
    this.notificationService.notify(`Ereignis um ${new Date().toLocaleTimeString()}`);
  }
}
```

## Weiterführendes
- **`switchMap` + `BehaviorSubject`** kombinieren: Wenn sich der User-State ändert, automatisch neue Daten laden – klassisches Pattern für nutzerbezogene API-Calls:
  ```typescript
  currentUser$.pipe(switchMap(user => user ? fetchData(user.id) : EMPTY))
  ```
- Offizielle Doku zu Multicasting: [RxJS – Higher-order observables](https://rxjs.dev/guide/subject)
- Tipp: Für neue Angular-Projekte prüfen, ob `toSignal()` + `resource()` / `httpResource()` die Subject-basierte State-Verwaltung ersetzen kann – Signals bieten dasselbe „letzter Wert sofort" Verhalten wie `BehaviorSubject`, sind aber reaktiver und einfacher zu debuggen.
