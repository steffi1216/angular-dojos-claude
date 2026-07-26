# Angular Dojo: Output API – Die neue funktionale Alternative zu @Output()
**Datum:** 2026-07-26
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst die neue `output()`-Funktion aus `@angular/core` kennen und verstehst, wie sie `@Output()` mit `EventEmitter` ablöst. Außerdem übst du `outputFromObservable()` und `outputToObservable()` für die nahtlose Interoperabilität zwischen der neuen Output-API und RxJS.

## Hintergrund & Theorie

Mit Angular 17.3 wurde die neue **Output API** eingeführt – als Teil der vollständigen signal-basierten Komponenten-API neben `input()`, `model()` und den Signal Queries. Ziel war eine konsistente, deklarative Alternative zu `@Output()` + `EventEmitter`.

### Vergleich

```typescript
// Alt: EventEmitter + @Output-Decorator
@Output() itemSelected = new EventEmitter<string>();

// Neu: output()-Funktion
itemSelected = output<string>();
```

Die `output()`-Funktion gibt ein `OutputEmitterRef<T>` zurück, das die Methode `.emit(value)` besitzt. Das Ergebnis ist **kein Signal**, sondern ein reines Event-System ohne reactive State.

### Zusätzliche Utilities

- `outputFromObservable(obs$)` – Wraps ein Observable als Output: Emits werden direkt aus dem Observable-Stream getriggert.
- `outputToObservable(outputRef)` – Konvertiert einen Output zurück in ein Observable (nützlich für Tests und Parent-Komponenten).

### Aliasing & required

```typescript
itemSelected = output<string>({ alias: 'selected' });
```

Outputs sind immer optional (kein `required`-Equivalent), da sie nur Events senden, keinen Wert halten.

## Aufgabe

Baue eine `SearchBoxComponent`, die:

1. Einen Suchbegriff via `input()` entgegennimmt
2. Eine interne Suche mit Debounce (RxJS) simuliert
3. Die Suchergebnisse über einen `output()` nach oben meldet
4. Einen weiteren Output `searchCleared` via `outputFromObservable()` erzeugt, der feuert, wenn das Input-Feld geleert wird

Die Parent-Komponente soll die Ergebnisse anzeigen und `outputToObservable()` nutzen, um auf Ereignisse zu reagieren.

### Schritte

1. Erstelle `SearchBoxComponent` als Standalone Component mit `input<string>('query')` und zwei Outputs: `searchResult = output<string[]>()` und `searchCleared` via `outputFromObservable()`.
2. Implementiere intern ein `Subject<string>` für den Suchbegriff, das mit `debounceTime(300)` und `distinctUntilChanged()` arbeitet. Mappe die Query auf simulierte Ergebnisse (z. B. ein Array gefilterter Strings) und rufe `searchResult.emit(results)` auf.
3. Erzeuge ein Observable `cleared$`, das aus dem Subject filtert (`filter(q => q === '')`), und übergebe es an `outputFromObservable(cleared$)`.
4. In der Parent-Komponente: Binde `(searchResult)` ans Template und nutze in `ngOnInit` die Funktion `outputToObservable(childRef.searchResult)` um als Observable zu lauschen und die Ergebnisliste in einem separaten Signal zu speichern.
5. Teste die Komponente: Tippe einen Begriff ein und beobachte die verzögerten Ergebnisse; lösche die Eingabe und beobachte, dass `searchCleared` feuert.

## Hints

<details>
<summary>Hint 1 – outputFromObservable Import & Struktur</summary>

```typescript
import { output, outputFromObservable, outputToObservable } from '@angular/core';
import { outputFromObservable } from '@angular/core/rxjs-interop';

// In der Komponente:
private query$ = new Subject<string>();
searchCleared = outputFromObservable(
  this.query$.pipe(filter(q => q === ''))
);
```

`outputFromObservable` und `outputToObservable` kommen aus `@angular/core/rxjs-interop`, nicht direkt aus `@angular/core`.

</details>

<details>
<summary>Hint 2 – Subject mit Debounce verbinden</summary>

```typescript
constructor() {
  this.query$.pipe(
    debounceTime(300),
    distinctUntilChanged(),
    filter(q => q.length > 0),
    map(q => MOCK_DATA.filter(item => item.toLowerCase().includes(q.toLowerCase())))
  ).subscribe(results => this.searchResult.emit(results));
}
```

Denke daran, den `Subject` in `ngOnDestroy` (oder via `takeUntilDestroyed()`) zu beenden, um Memory Leaks zu vermeiden.

</details>

## Beispiellösung

```typescript
// search-box.component.ts
import { Component, input, output, OnInit, inject, DestroyRef } from '@angular/core';
import { outputFromObservable } from '@angular/core/rxjs-interop';
import { Subject, debounceTime, distinctUntilChanged, filter, map } from 'rxjs';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { FormsModule } from '@angular/forms';

const MOCK_DATA = ['Angular', 'Signals', 'RxJS', 'TypeScript', 'NgRx', 'Standalone', 'Deferrable', 'Zoneless'];

@Component({
  selector: 'app-search-box',
  standalone: true,
  imports: [FormsModule],
  template: `
    <input
      type="text"
      [ngModel]="query()"
      (ngModelChange)="onInput($event)"
      placeholder="Suche..."
    />
  `
})
export class SearchBoxComponent {
  query = input<string>('');

  searchResult = output<string[]>();

  private query$ = new Subject<string>();

  searchCleared = outputFromObservable(
    this.query$.pipe(filter(q => q === ''))
  );

  constructor() {
    this.query$.pipe(
      debounceTime(300),
      distinctUntilChanged(),
      filter(q => q.length > 0),
      map(q => MOCK_DATA.filter(item =>
        item.toLowerCase().includes(q.toLowerCase())
      )),
      takeUntilDestroyed()
    ).subscribe(results => this.searchResult.emit(results));
  }

  onInput(value: string): void {
    this.query$.next(value);
  }
}
```

```typescript
// app.component.ts
import { Component, viewChild, OnInit, signal } from '@angular/core';
import { outputToObservable } from '@angular/core/rxjs-interop';
import { SearchBoxComponent } from './search-box.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [SearchBoxComponent],
  template: `
    <app-search-box (searchResult)="results.set($event)" (searchCleared)="results.set([])" />
    <ul>
      @for (result of results(); track result) {
        <li>{{ result }}</li>
      }
    </ul>
    <p>Ergebnisse via outputToObservable: {{ streamCount() }}</p>
  `
})
export class AppComponent implements OnInit {
  searchBox = viewChild.required(SearchBoxComponent);
  results = signal<string[]>([]);
  streamCount = signal(0);

  ngOnInit() {
    // Alternative: outputToObservable für Observables-basierte Verarbeitung
    outputToObservable(this.searchBox().searchResult).subscribe(() => {
      this.streamCount.update(n => n + 1);
    });
  }
}
```

## Weiterführendes

- `output()` mit Alias: `output<string>({ alias: 'externalName' })` – nützlich für Migrationspfade von `@Output('alias')`.
- Die offizielle Angular-Migrationsmöglichkeit: `ng generate @angular/core:output-migration` konvertiert automatisch alle `@Output` in der Codebasis auf die neue API.
- Lies: [Angular Output API RFC & Docs](https://angular.dev/guide/components/outputs) für die vollständige Spezifikation und Unterschiede zu `EventEmitter`.
- Kombination mit `linkedSignal()`: Ein Output kann indirekt einen internen State im Parent via Callback aktualisieren, während ein `linkedSignal` darauf reagiert – ein mächtiges Muster für entkoppelte Komponenten.
