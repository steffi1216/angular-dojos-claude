# Angular Dojo: RxJS `groupBy` und `partition` – Datenströme kategorisieren
**Datum:** 2026-10-06
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit den RxJS-Operatoren `groupBy` und `partition` einen einzigen Datenstrom in mehrere kategorisierte Teilströme aufteilst – ideal für Echtzeit-Szenarien mit gemischten Event-Typen in Angular.

## Hintergrund & Theorie
Oft empfangst du über einen einzigen Observable (z.B. WebSocket, HTTP-Polling) Nachrichten unterschiedlicher Typen. Statt alles mit `filter()` manuell aufzuteilen, bieten RxJS zwei elegante Operatoren:

**`groupBy(keySelector)`** teilt einen Observable in mehrere `GroupedObservable`s auf. Jede Gruppe ist selbst ein Observable mit einer `key`-Property. Wichtig: Du musst jeden Gruppen-Observable aktiv subscriben (z.B. mit `mergeMap`), sonst werden Elemente gebuffert und nie emittiert.

```typescript
source$.pipe(
  groupBy(item => item.category),
  mergeMap(group$ => group$.pipe(
    scan((acc, item) => [...acc, item], [] as Item[]),
    map(items => ({ key: group$.key, items }))
  ))
)
```

**`partition(predicate)`** gibt ein Tupel `[matching$, nonMatching$]` zurück – zwei Observables, aufgeteilt nach einem Prädikat. Perfekt für binäre Aufteilung wie Erfolg vs. Fehler.

```typescript
const [success$, error$] = partition(events$, e => e.status === 'ok');
```

`groupBy` eignet sich für N Kategorien, `partition` ist eleganter bei einer binären Trennung. Beide teilen sich die gleiche Source – bei mehrfachem Subscribe ist ein vorgeschaltetes `share()` wichtig.

## Aufgabe
Du entwickelst einen **Real-Time Order Monitor** in Angular. Über ein simuliertes Observable kommen Bestellungen mit unterschiedlichen Status (`pending`, `processing`, `completed`, `failed`) herein. Du gruppierst sie live und zeigst pro Statusgruppe einen Counter und die letzten 3 Bestellungen an.

### Schritte
1. Erstelle einen `OrderMonitorComponent` (standalone). Simuliere mit `interval(800)` einen Bestellungsstrom – jede Sekunde eine `Order` mit zufälligem Status.
2. Wende `share()` auf den Source-Stream an, damit mehrere Subscriber dieselbe Quelle nutzen.
3. Nutze `groupBy(order => order.status)` mit `mergeMap`, um je Gruppe die letzten 3 Bestellungen mit `scan` zu akkumulieren und in einem `signal`-basierten State zu speichern.
4. Teile den Stream mit `partition` in `active$` (pending/processing) und `terminal$` (completed/failed) auf und halte separate Gesamtzähler als Signals.
5. Stelle sicher, dass alle Subscriptions mit `takeUntilDestroyed()` sauber abgeräumt werden.

## Hints
<details>
<summary>Hint 1 – groupBy ohne mergeMap verliert Daten</summary>

`groupBy` allein subscribt die erzeugten Gruppen-Observables nicht automatisch. Ohne `mergeMap` werden Elemente intern gebuffert und nie weitergeleitet:

```typescript
orders$.pipe(
  groupBy(order => order.status),
  mergeMap(group$ =>
    group$.pipe(
      scan((acc, order) => [...acc, order].slice(-3), [] as Order[]),
      map(orders => ({ key: group$.key, orders }))
    )
  )
).subscribe(({ key, orders }) => {
  this.groups.update(g => ({ ...g, [key]: orders }));
});
```
</details>

<details>
<summary>Hint 2 – partition ist eine Funktion, kein pipeable Operator</summary>

`partition` wird nicht in einer `pipe()` verwendet, sondern direkt aufgerufen:

```typescript
import { partition } from 'rxjs';

const [active$, terminal$] = partition(
  orders$,
  order => order.status === 'pending' || order.status === 'processing'
);
```

Da `active$` und `terminal$` beide auf `orders$` subscriben, braucht `orders$` ein `share()`, um nicht zweimal die Source auszuführen.
</details>

## Beispiellösung
```typescript
import { Component, computed, DestroyRef, inject, OnInit, signal } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { CurrencyPipe } from '@angular/common';
import { interval, partition, share } from 'rxjs';
import { groupBy, map, mergeMap, scan } from 'rxjs/operators';

type OrderStatus = 'pending' | 'processing' | 'completed' | 'failed';

interface Order {
  id: number;
  status: OrderStatus;
  amount: number;
}

@Component({
  selector: 'app-order-monitor',
  standalone: true,
  imports: [CurrencyPipe],
  template: `
    <h2>Real-Time Order Monitor</h2>

    <div class="stats">
      <span>Aktiv: {{ activeCount() }}</span>
      <span>Abgeschlossen: {{ terminalCount() }}</span>
    </div>

    <div class="groups">
      @for (entry of groupEntries(); track entry.key) {
        <div class="group">
          <h3>{{ entry.key }}</h3>
          <ul>
            @for (order of entry.orders; track order.id) {
              <li>#{{ order.id }} – {{ order.amount | currency:'EUR' }}</li>
            }
          </ul>
        </div>
      }
    </div>
  `
})
export class OrderMonitorComponent implements OnInit {
  private readonly destroyRef = inject(DestroyRef);
  private readonly groups = signal<Record<string, Order[]>>({});

  readonly groupEntries = computed(() =>
    Object.entries(this.groups()).map(([key, orders]) => ({ key, orders }))
  );

  readonly activeCount = signal(0);
  readonly terminalCount = signal(0);

  ngOnInit() {
    let idCounter = 0;
    const statuses: OrderStatus[] = ['pending', 'processing', 'completed', 'failed'];

    const orders$ = interval(800).pipe(
      map((): Order => ({
        id: ++idCounter,
        status: statuses[Math.floor(Math.random() * statuses.length)],
        amount: Math.round(Math.random() * 490 + 10),
      })),
      share()
    );

    // groupBy für kategorisierten State – letzte 3 Bestellungen je Status
    orders$.pipe(
      groupBy(order => order.status),
      mergeMap(group$ =>
        group$.pipe(
          scan((acc, order) => [...acc, order].slice(-3), [] as Order[]),
          map(orders => ({ key: group$.key, orders }))
        )
      ),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(({ key, orders }) => {
      this.groups.update(g => ({ ...g, [key]: orders }));
    });

    // partition für binäre Aufteilung: aktiv vs. terminal
    const [active$, terminal$] = partition(
      orders$,
      order => order.status === 'pending' || order.status === 'processing'
    );

    active$.pipe(
      scan(count => count + 1, 0),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(count => this.activeCount.set(count));

    terminal$.pipe(
      scan(count => count + 1, 0),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(count => this.terminalCount.set(count));
  }
}
```

## Weiterführendes
- `groupBy` mit `windowTime` kombinieren, um Gruppen periodisch zurückzusetzen
- Bei mehr als 2 Kategorien: `partition` durch mehrfaches `filter` + `share` ersetzen
- Offizielle Doku: [groupBy](https://rxjs.dev/api/operators/groupBy), [partition](https://rxjs.dev/api/index/function/partition)
- Für persistenten Gruppen-State: `groupBy` mit dem NgRx Signal Store kombinieren
