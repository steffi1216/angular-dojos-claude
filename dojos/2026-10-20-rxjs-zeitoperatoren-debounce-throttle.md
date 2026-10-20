# Angular Dojo: RxJS Zeitoperatoren – debounce, throttle, audit, sample
**Datum:** 2026-10-20
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die vier zentralen RxJS-Zeitoperatoren (`debounceTime`, `throttleTime`, `auditTime`, `sampleTime`) zu unterscheiden und setzt sie gezielt in einer Angular-Suchkomponente und einem Scroll-Event-Handler ein.

## Hintergrund & Theorie

In Angular-Anwendungen entstehen schnell hochfrequente Ereignisströme – Tastatureingaben, Scroll-Events, Resize-Events, Mouse-Moves. Ohne Drosselung feuern API-Anfragen oder DOM-Updates hundertfach pro Sekunde. Die vier Zeitoperatoren lösen dieses Problem auf unterschiedliche Weise:

| Operator | Gibt den Wert aus… |
|---|---|
| `debounceTime(ms)` | …erst nach `ms` Millisekunden Stille (kein neuer Wert) |
| `throttleTime(ms)` | …sofort, ignoriert dann alle Werte für `ms` ms |
| `auditTime(ms)` | …den _letzten_ Wert nach Ablauf des `ms`-Timers |
| `sampleTime(ms)` | …den letzten Wert in festen Intervallen von `ms` ms |

**Wann was benutzen?**
- **debounceTime** → Sucheingaben, Autocomplete (warte auf Tipp-Pause)
- **throttleTime** → Button-Klicks, Scroll-Events (sofort reagieren, dann einfrieren)
- **auditTime** → Resize-Handler, Scroll-Position (immer letzten Wert wollen)
- **sampleTime** → Echtzeit-Dashboards, Progress-Bars (regelmäßige Updates)

`throttleTime` hat eine `config`-Option (`leading`/`trailing`), die steuert, ob der erste oder letzte Wert eines Zeitfensters emittiert wird.

## Aufgabe

Erstelle eine Standalone-Komponente `SmartSearchComponent`, die drei Features kombiniert:

1. **Suchfeld mit debounceTime**: API-Anfragen erst nach 400 ms Tipp-Pause senden
2. **Scroll-Event mit auditTime**: Scroll-Position alle 200 ms auslesen
3. **Save-Button mit throttleTime**: Doppelklick-Schutz (500 ms Sperre nach erstem Klick)

Die Komponente soll das Ergebnis in der UI anzeigen (kein echtes Backend nötig – simuliere die API-Antwort mit `of(...).pipe(delay(100))`).

### Schritte

1. Generiere die Komponente: `ng generate component smart-search --standalone`
2. Erstelle im Template ein Eingabefeld, ein Scroll-Container-`<div>` (100 vh hoch mit overflow-y: scroll und langem Inhalt) und einen Save-Button
3. Verbinde das `input`-Event des Feldes mit `fromEvent`, pipe durch `debounceTime(400)`, dann `distinctUntilChanged()` und `switchMap` zu einer simulierten API
4. Verbinde `scroll`-Events des Containers mit `fromEvent`, pipe durch `auditTime(200)` und extrahiere `scrollTop`
5. Verbinde den Button-`click` mit `fromEvent`, pipe durch `throttleTime(500)` und logge / zeige die Aktion an
6. Verwende `takeUntilDestroyed()` für alle Subscriptions
7. Zeige in der Template die aktuelle Suchquery, die Scroll-Position und die Anzahl der gespeicherten Aktionen an

## Hints

<details>
<summary>Hint 1 – fromEvent und Elementreferenzen</summary>

Nutze `viewChild` (Signal-API) für die Elementreferenzen:

```typescript
searchInput = viewChild.required<ElementRef>('searchInput');
scrollContainer = viewChild.required<ElementRef>('scrollContainer');
saveButton = viewChild.required<ElementRef>('saveButton');
```

Initialisiere die Observables in `afterNextRender` oder im `ngAfterViewInit`-Lifecycle:

```typescript
ngAfterViewInit() {
  fromEvent<InputEvent>(this.searchInput().nativeElement, 'input')
    .pipe(
      debounceTime(400),
      // ...
    )
    .subscribe();
}
```

</details>

<details>
<summary>Hint 2 – throttleTime mit trailing config</summary>

Standardmäßig emittiert `throttleTime` den _ersten_ Wert (leading: true, trailing: false). Um auch den letzten Wert eines Zeitfensters zu erhalten:

```typescript
import { throttleTime } from 'rxjs/operators';
import { asyncScheduler } from 'rxjs';

fromEvent(button, 'click').pipe(
  throttleTime(500, asyncScheduler, { leading: true, trailing: false })
)
```

Für einen Doppelklick-Schutz (nur erster Klick zählt) reicht die Standardkonfiguration.

</details>

<details>
<summary>Hint 3 – auditTime vs. debounceTime beim Scroll</summary>

`debounceTime` würde beim Scrollen erst feuern, wenn der Nutzer **aufgehört** hat zu scrollen – das ist für die Live-Scroll-Position unerwünscht.
`auditTime(200)` feuert hingegen alle 200 ms den _letzten_ Wert, solange Events ankommen:

```typescript
fromEvent(container, 'scroll').pipe(
  auditTime(200),
  map((event) => (event.target as HTMLElement).scrollTop)
)
```

</details>

## Beispiellösung

```typescript
import {
  Component, ElementRef, viewChild, AfterViewInit, signal, inject
} from '@angular/core';
import { CommonModule } from '@angular/common';
import { fromEvent, of } from 'rxjs';
import {
  debounceTime, distinctUntilChanged, switchMap, delay,
  auditTime, throttleTime, map
} from 'rxjs/operators';
import { takeUntilDestroyed, DestroyRef } from '@angular/core/rxjs-interop';

@Component({
  selector: 'app-smart-search',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="container">
      <h2>Smart Search Demo</h2>

      <input #searchInput type="text" placeholder="Suche..." />
      <p>Query: {{ currentQuery() }}</p>
      <p>Ergebnis: {{ searchResult() }}</p>

      <hr />

      <div #scrollContainer class="scroll-box">
        @for (item of items; track item) {
          <p>{{ item }}</p>
        }
      </div>
      <p>Scroll-Position: {{ scrollTop() }}px</p>

      <hr />

      <button #saveButton type="button">Speichern</button>
      <p>Gespeichert: {{ saveCount() }}x</p>
    </div>
  `,
  styles: [`
    .scroll-box { height: 200px; overflow-y: scroll; border: 1px solid #ccc; padding: 8px; }
  `]
})
export class SmartSearchComponent implements AfterViewInit {
  searchInput = viewChild.required<ElementRef<HTMLInputElement>>('searchInput');
  scrollContainer = viewChild.required<ElementRef<HTMLDivElement>>('scrollContainer');
  saveButton = viewChild.required<ElementRef<HTMLButtonElement>>('saveButton');

  currentQuery = signal('');
  searchResult = signal('–');
  scrollTop = signal(0);
  saveCount = signal(0);

  items = Array.from({ length: 50 }, (_, i) => `Eintrag ${i + 1}`);

  private destroyRef = inject(DestroyRef);

  ngAfterViewInit(): void {
    // 1. Suchfeld – debounceTime
    fromEvent<InputEvent>(this.searchInput().nativeElement, 'input').pipe(
      map((e) => (e.target as HTMLInputElement).value),
      debounceTime(400),
      distinctUntilChanged(),
      switchMap((query) => {
        this.currentQuery.set(query);
        if (!query) return of('Keine Suche');
        return of(`Ergebnisse für "${query}"`).pipe(delay(100));
      }),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe((result) => this.searchResult.set(result));

    // 2. Scroll – auditTime
    fromEvent(this.scrollContainer().nativeElement, 'scroll').pipe(
      auditTime(200),
      map((e) => (e.target as HTMLElement).scrollTop),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe((top) => this.scrollTop.set(Math.round(top)));

    // 3. Button – throttleTime (Doppelklick-Schutz)
    fromEvent(this.saveButton().nativeElement, 'click').pipe(
      throttleTime(500),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(() => this.saveCount.update((n) => n + 1));
  }
}
```

## Weiterführendes

- **`debounce(()=> timer(400))`** statt `debounceTime`: Ermöglicht dynamische Verzögerungen basierend auf dem Wert selbst
- **`throttleTime` mit `{ leading: false, trailing: true }`**: Verhält sich wie `auditTime`, ist aber durch die explizite Konfiguration flexibler
- RxJS-Doku: [debounceTime](https://rxjs.dev/api/operators/debounceTime) | [throttleTime](https://rxjs.dev/api/operators/throttleTime) | [auditTime](https://rxjs.dev/api/operators/auditTime) | [sampleTime](https://rxjs.dev/api/operators/sampleTime)
- Für Signal-basierte Ansätze: `toSignal(source$.pipe(debounceTime(400)))` kombiniert RxJS-Drosselung mit Signals direkt im Template
