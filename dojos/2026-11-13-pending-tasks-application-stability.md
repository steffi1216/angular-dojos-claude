# Angular Dojo: PendingTasks & ApplicationRef.isStable
**Datum:** 2026-11-13
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie Angular die Rendering-Stabilität über den `PendingTasks`-Service und `ApplicationRef.isStable` steuert – und wie du mit diesen APIs eigene asynchrone Operationen (z. B. WebSockets, Custom Timers) korrekt in das SSR-Rendering und Testing einbindest.

## Hintergrund & Theorie

Angular muss wissen, wann eine Applikation „stabil" ist – d. h. wenn alle asynchronen Aufgaben abgeschlossen sind. Dies ist entscheidend für:

- **SSR (Server-Side Rendering):** Angular Universal / `@angular/ssr` serialisiert die Seite erst, wenn `isStable` den Wert `true` emittiert.
- **Angular Testing:** `ComponentFixture.whenStable()` wartet intern auf `isStable`.
- **Prerendering:** Statische Seiten werden erst nach Stabilität gerendert.

Seit Angular 18 stellt Angular den `PendingTasks`-Service bereit (vorher: `ApplicationRef.isStable` direkt via `NgZone`). Dieser Service erlaubt es, eigene „Pending Tasks" zu registrieren, die die Stabilität blockieren – unabhängig von Zone.js oder dem zoneless-Modus.

```typescript
// Injizierbar via inject()
const pendingTasks = inject(PendingTasks);

// Erstellt einen Cleanup-Handle
const cleanup = pendingTasks.add(); 

// Wenn die Aufgabe abgeschlossen ist:
cleanup();
```

Typische Anwendungsfälle:
- Eigene Timer (`setTimeout`, `setInterval`) außerhalb der Zone
- WebSocket-Verbindungen bei SSR
- Lazy-geladene Ressourcen oder externe Skripte
- Komplexe Initialisierungsroutinen

## Aufgabe

Erstelle einen Service `DataSyncService`, der beim Start eine simulierte Datensynchronisierung durchführt. Der Service soll Angular korrekt mitteilen, dass eine Initialisierungsaufgabe läuft, damit SSR und Tests korrekt auf den Abschluss warten.

### Schritte

1. Erstelle einen `DataSyncService` mit `providedIn: 'root'`, der `PendingTasks` injiziert und im Konstruktor eine Synchronisierung startet.

2. Implementiere eine Methode `syncData()`, die einen `cleanup`-Handle via `pendingTasks.add()` registriert, einen simulierten async-Vorgang (z. B. 2 Sekunden `setTimeout` **außerhalb** der Zone mit `NgZone.runOutsideAngular`) durchführt und danach `cleanup()` aufruft.

3. Erstelle eine `AppComponent`, die den Service injiziert und den aktuellen Status (`syncing`, `done`) als Signal anzeigt.

4. Schreibe einen Unit-Test, der mit `fakeAsync` / `tick` verifiziert, dass `ApplicationRef.isStable` erst nach Abschluss der Synchronisierung `true` emittiert.

### Schritte (Detail)

1. **Service erstellen:**
```typescript
// data-sync.service.ts
@Injectable({ providedIn: 'root' })
export class DataSyncService {
  private pendingTasks = inject(PendingTasks);
  private ngZone = inject(NgZone);

  readonly status = signal<'idle' | 'syncing' | 'done'>('idle');

  startSync(): void {
    this.status.set('syncing');
    // Schritt 2: cleanup handle registrieren
    // Schritt 2: außerhalb der Zone simulieren
    // Schritt 2: cleanup nach Abschluss aufrufen
  }
}
```

2. **`PendingTasks.add()` aufrufen** und den Cleanup-Handle speichern.

3. **`runOutsideAngular`** nutzen, damit der `setTimeout` Zone.js nicht triggert, aber `PendingTasks` trotzdem blockiert.

4. **Test:** Prüfe, dass `isStable` blockiert ist, während die Aufgabe läuft, und `true` wird, sobald `cleanup()` aufgerufen wurde.

## Hints

<details>
<summary>Hint 1 – PendingTasks API</summary>

`PendingTasks` wird direkt per `inject(PendingTasks)` bezogen (kein `forRoot` nötig). Die Methode `add()` gibt eine Funktion zurück – rufe diese auf, wenn die Aufgabe abgeschlossen ist:

```typescript
const cleanup = this.pendingTasks.add();
// ... async work ...
cleanup(); // Aufgabe als abgeschlossen markieren
```

</details>

<details>
<summary>Hint 2 – Test mit isStable</summary>

Für den Test nutze `ApplicationRef.isStable` als Observable und prüfe, ob der erste `true`-Wert erst nach dem `tick()` kommt:

```typescript
const appRef = TestBed.inject(ApplicationRef);
let isStable = false;
appRef.isStable.pipe(first(v => v)).subscribe(() => (isStable = true));

service.startSync();
expect(isStable).toBe(false); // noch nicht stabil

tick(2000); // Simuliert den setTimeout
expect(isStable).toBe(true); // jetzt stabil
```

</details>

## Beispiellösung

```typescript
// data-sync.service.ts
import { Injectable, NgZone, inject, signal } from '@angular/core';
import { PendingTasks } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class DataSyncService {
  private pendingTasks = inject(PendingTasks);
  private ngZone = inject(NgZone);

  readonly status = signal<'idle' | 'syncing' | 'done'>('idle');

  startSync(): void {
    this.status.set('syncing');
    const cleanup = this.pendingTasks.add();

    this.ngZone.runOutsideAngular(() => {
      setTimeout(() => {
        this.ngZone.run(() => {
          this.status.set('done');
          cleanup();
        });
      }, 2000);
    });
  }
}
```

```typescript
// app.component.ts
import { Component, inject, OnInit } from '@angular/core';
import { DataSyncService } from './data-sync.service';

@Component({
  selector: 'app-root',
  standalone: true,
  template: `
    <p>Status: {{ syncService.status() }}</p>
    <button (click)="syncService.startSync()">Sync starten</button>
  `,
})
export class AppComponent implements OnInit {
  syncService = inject(DataSyncService);

  ngOnInit(): void {
    this.syncService.startSync();
  }
}
```

```typescript
// data-sync.service.spec.ts
import { TestBed, fakeAsync, tick } from '@angular/core/testing';
import { ApplicationRef } from '@angular/core';
import { first } from 'rxjs';
import { DataSyncService } from './data-sync.service';

describe('DataSyncService', () => {
  let service: DataSyncService;
  let appRef: ApplicationRef;

  beforeEach(() => {
    TestBed.configureTestingModule({});
    service = TestBed.inject(DataSyncService);
    appRef = TestBed.inject(ApplicationRef);
  });

  it('sollte die App-Stabilität blockieren bis Sync abgeschlossen ist', fakeAsync(() => {
    let isStable = false;
    appRef.isStable.pipe(first(v => v)).subscribe(() => (isStable = true));

    service.startSync();
    expect(service.status()).toBe('syncing');
    expect(isStable).toBe(false);

    tick(2000);

    expect(service.status()).toBe('done');
    expect(isStable).toBe(true);
  }));
});
```

## Weiterführendes

- **Offiziell:** [Angular Docs – ApplicationRef](https://angular.dev/api/core/ApplicationRef) und die `isStable` Observable-Eigenschaft
- **Vertiefung:** `ExperimentalPendingTasks` wurde in Angular 17 eingeführt und als `PendingTasks` in Angular 18 stabilisiert – prüfe die CHANGELOG-Einträge für Breaking Changes
- **SSR-Integration:** Kombiniere `PendingTasks` mit `TransferState` (siehe Dojo `2026-09-11-transfer-state.md`), um Daten vom Server zu übertragen, die erst nach einer pending Task verfügbar sind
- **Zoneless-Kontext:** In einer zoneless App (`provideExperimentalZonelessChangeDetection`) ist `PendingTasks` die **einzige** Möglichkeit, Stabilität zu blockieren – Zone.js ist nicht mehr vorhanden
