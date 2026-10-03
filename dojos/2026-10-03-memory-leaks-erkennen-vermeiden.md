# Angular Dojo: Memory Leaks erkennen und vermeiden
**Datum:** 2026-10-03
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die häufigsten Ursachen von Memory Leaks in Angular-Anwendungen kennen und übst systematisch, sie zu erkennen und mit modernen Angular-Patterns (Signals, `DestroyRef`, `takeUntilDestroyed`) zu beheben.

## Hintergrund & Theorie

Memory Leaks entstehen in Angular typischerweise durch Subscriptions, die nach dem Zerstören einer Komponente weiterhin aktiv sind. Jede nicht geschlossene Subscription hält Referenzen auf Komponenten-Instanzen — und damit auf alles, was die Komponente referenziert.

**Häufige Leak-Quellen:**

1. **RxJS-Subscriptions ohne Unsubscribe** – `interval()`, `fromEvent()`, globale Services
2. **`setInterval` / `setTimeout` ohne Cleanup** – insbesondere außerhalb der Angular-Zone
3. **DOM-Event-Listener via `addEventListener`** ohne zugehöriges `removeEventListener`
4. **Globale Store-Subscriptions** (NgRx, Akita) in Komponenten-Klassen
5. **`Subject` / `BehaviorSubject` in Services**, die nie `complete()` aufgerufen bekommen
6. **`ViewContainerRef.createComponent`** ohne explizites `destroy()`

**Moderne Gegenmittel:**

- `takeUntilDestroyed(destroyRef)` (Angular 16+) – automatisches Unsubscribe über `DestroyRef`
- `toObservable()` + Signals – reaktiver ohne manuelle Subscriptions
- `DestroyRef.onDestroy()` – expliziter Cleanup-Hook
- `AsyncPipe` – unsubscribed automatisch beim Zerstören der View

Das Chrome DevTools Memory-Tab (Heap-Snapshots, Timeline) und die Angular DevTools ermöglichen es, Leaks zur Laufzeit zu identifizieren.

## Aufgabe

Du hast eine `DashboardComponent`, die mehrere häufige Memory-Leak-Muster enthält. Deine Aufgabe ist es, alle Leaks zu identifizieren und mit modernen Angular-Patterns zu beheben.

### Ausgangscode

```typescript
// dashboard.component.ts (FEHLERHAFT – enthält Memory Leaks)
@Component({
  selector: 'app-dashboard',
  standalone: true,
  template: `
    <div>Live Counter: {{ counter }}</div>
    <div>Mouse Position: {{ mouseX }}, {{ mouseY }}</div>
    <div>Last Update: {{ lastUpdate }}</div>
    <button (click)="startPolling()">Start Polling</button>
  `,
})
export class DashboardComponent implements OnInit, OnDestroy {
  counter = 0;
  mouseX = 0;
  mouseY = 0;
  lastUpdate = '';

  private pollingActive = false;

  constructor(
    private dataService: DataService,
    private notificationService: NotificationService,
  ) {}

  ngOnInit() {
    // Leak 1: interval-Subscription wird nie beendet
    interval(1000).subscribe((n) => {
      this.counter = n;
    });

    // Leak 2: globaler DOM-Event-Listener ohne Cleanup
    document.addEventListener('mousemove', (e: MouseEvent) => {
      this.mouseX = e.clientX;
      this.mouseY = e.clientY;
    });

    // Leak 3: Service-Observable wird direkt subscribed
    this.dataService.updates$.subscribe((update) => {
      this.lastUpdate = update;
    });

    // Leak 4: globale Notifications werden subscribed, egal ob Route verlassen wird
    this.notificationService.notifications$.subscribe((msg) => {
      console.log('Notification:', msg);
    });
  }

  startPolling() {
    if (this.pollingActive) return;
    this.pollingActive = true;

    // Leak 5: weiterer setInterval ohne clearInterval
    setInterval(() => {
      this.dataService.refresh();
    }, 5000);
  }

  ngOnDestroy() {
    // Cleanup fehlt komplett!
  }
}
```

```typescript
// data.service.ts
@Injectable({ providedIn: 'root' })
export class DataService {
  updates$ = new Subject<string>();

  refresh() {
    // simuliert Daten-Refresh
    this.updates$.next(`Refresh at ${new Date().toLocaleTimeString()}`);
  }
}
```

### Schritte

1. **Identifiziere alle 5 Memory Leaks** und kommentiere, warum jeder problematisch ist.

2. **Refactore `DashboardComponent`** mit `DestroyRef` und `takeUntilDestroyed`:
   - Ersetze manuelle Subscriptions durch `takeUntilDestroyed(this.destroyRef)`
   - Verwende `fromEvent()` statt `addEventListener` — ebenfalls mit `takeUntilDestroyed`
   - Räume `setInterval` über `DestroyRef.onDestroy()` auf

3. **Wandle `counter` und `mouseX/Y` in Signals um** (optional, wenn Zeit):
   - Nutze `toSignal()` für das `interval`-Observable
   - Verwende ein `signal()` für die Mausposition, das per `fromEvent` + `takeUntilDestroyed` befüllt wird

4. **Schreibe einen Test**, der prüft, dass nach `fixture.destroy()` keine weiteren Emissions stattfinden (nutze `fakeAsync` und `tick`).

## Hints

<details>
<summary>Hint 1 – DestroyRef injizieren</summary>

```typescript
export class DashboardComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    interval(1000)
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((n) => (this.counter = n));
  }
}
```

`takeUntilDestroyed` kann auch ohne `destroyRef` genutzt werden, wenn es im Konstruktor oder in einem Injection Context aufgerufen wird — dann holt Angular den `DestroyRef` automatisch.

</details>

<details>
<summary>Hint 2 – DOM-Events ohne Leak</summary>

```typescript
fromEvent<MouseEvent>(document, 'mousemove')
  .pipe(takeUntilDestroyed(this.destroyRef))
  .subscribe((e) => {
    this.mouseX = e.clientX;
    this.mouseY = e.clientY;
  });
```

`fromEvent` unsubscribed das native Event automatisch, sobald die RxJS-Subscription geschlossen wird — also auch das `removeEventListener`.

</details>

<details>
<summary>Hint 3 – setInterval über DestroyRef aufräumen</summary>

```typescript
startPolling() {
  if (this.pollingActive) return;
  this.pollingActive = true;

  const id = setInterval(() => this.dataService.refresh(), 5000);
  this.destroyRef.onDestroy(() => clearInterval(id));
}
```

`DestroyRef.onDestroy()` ist der Angular-native Ersatz für `ngOnDestroy` ohne Interface — und funktioniert auch in Funktionen, die nach dem Konstruktor aufgerufen werden.

</details>

<details>
<summary>Hint 4 – toSignal() für Observables</summary>

```typescript
counter = toSignal(interval(1000), { initialValue: 0 });
```

`toSignal()` nutzt intern `takeUntilDestroyed` — vollständig leak-safe, wenn es im Injection Context (Konstruktor oder Klassenfeld-Initialisierung) aufgerufen wird.

</details>

## Beispiellösung

```typescript
import {
  Component,
  DestroyRef,
  OnInit,
  inject,
  signal,
} from '@angular/core';
import { takeUntilDestroyed, toSignal } from '@angular/core/rxjs-interop';
import { fromEvent, interval } from 'rxjs';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  template: `
    <div>Live Counter: {{ counter() }}</div>
    <div>Mouse Position: {{ mouseX() }}, {{ mouseY() }}</div>
    <div>Last Update: {{ lastUpdate() }}</div>
    <button (click)="startPolling()">Start Polling</button>
  `,
})
export class DashboardComponent implements OnInit {
  private destroyRef = inject(DestroyRef);
  private dataService = inject(DataService);
  private notificationService = inject(NotificationService);

  // Signal direkt aus Observable — takeUntilDestroyed ist intern
  counter = toSignal(interval(1000), { initialValue: 0 });

  // Mutable Signals für DOM-Events
  mouseX = signal(0);
  mouseY = signal(0);
  lastUpdate = signal('');

  private pollingActive = false;

  constructor() {
    // fromEvent mit takeUntilDestroyed — kein removeEventListener nötig
    fromEvent<MouseEvent>(document, 'mousemove')
      .pipe(takeUntilDestroyed()) // DestroyRef wird automatisch aus dem Injection Context geholt
      .subscribe((e) => {
        this.mouseX.set(e.clientX);
        this.mouseY.set(e.clientY);
      });
  }

  ngOnInit() {
    this.dataService.updates$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((update) => this.lastUpdate.set(update));

    this.notificationService.notifications$
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe((msg) => console.log('Notification:', msg));
  }

  startPolling() {
    if (this.pollingActive) return;
    this.pollingActive = true;

    const id = setInterval(() => this.dataService.refresh(), 5000);
    // DestroyRef als sichere Alternative zu ngOnDestroy
    this.destroyRef.onDestroy(() => clearInterval(id));
  }

  // ngOnDestroy wird nicht mehr benötigt — alles läuft über DestroyRef
}
```

```typescript
// dashboard.component.spec.ts
import { fakeAsync, TestBed, tick } from '@angular/core/testing';
import { DashboardComponent } from './dashboard.component';

describe('DashboardComponent – kein Memory Leak', () => {
  it('sollte nach destroy keine weiteren counter-Updates erhalten', fakeAsync(() => {
    const fixture = TestBed.createComponent(DashboardComponent);
    fixture.detectChanges();

    tick(3000); // 3 Sekunden simulieren
    const counterAfter3s = fixture.componentInstance.counter();
    expect(counterAfter3s).toBeGreaterThan(0);

    fixture.destroy(); // Komponente zerstören

    tick(5000); // weitere 5 Sekunden — kein Update erwartet
    // counter() sollte sich nicht mehr ändern
    expect(fixture.componentInstance.counter()).toBe(counterAfter3s);
  }));
});
```

## Weiterführendes

- **Chrome DevTools Memory Tab**: Mit "Heap Snapshot" vor und nach dem Navigieren zu einer Komponente prüfen, ob Komponenten-Instanzen im Heap verbleiben — ein Zeichen für Leaks.
- **`ng-leak` und `@angular-extensions/toolkit`**: Community-Tools zur automatischen Leak-Erkennung in Tests.
- [`takeUntilDestroyed` API Docs](https://angular.dev/api/core/rxjs-interop/takeUntilDestroyed) – offizielle Dokumentation mit Injection-Context-Hinweisen.
- Muster für globale Services: Wenn ein Service `providedIn: 'root'` ist und Subjects exponiert, `complete()` nur aufrufen, wenn der gesamte App-Lifecycle endet — sonst Subjects **nie** aus Services completen, da andere Subscriber betroffen wären. Stattdessen Subscriptions in Konsumenten mit `takeUntilDestroyed` absichern.
