# Angular Dojo: fakeAsync & tick – Präzises Testen von asynchronem Code
**Datum:** 2026-11-19
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit `fakeAsync`, `tick`, `flush` und `discardPeriodicTasks` asynchronen Angular-Code (Timeouts, Intervalle, Promises, `debounceTime`) deterministisch und ohne echte Wartezeiten testest.

## Hintergrund & Theorie

Beim Testen von asynchronem Code gibt es zwei Hauptprobleme: Der Test muss auf echte Zeit warten, und das Verhalten ist schwer vorhersagbar. Angular löst das mit der **`fakeAsync`-Zone**, die eine simulierte Uhr bereitstellt.

**Kernkonzepte:**

- **`fakeAsync(fn)`** – Wrapper, der die Test-Funktion in einer kontrollierten Zone ausführt. Alle `setTimeout`/`setInterval`-Aufrufe werden eingefroren, bis du explizit Zeit vorspulst.
- **`tick(millis?)`** – Springt die simulierte Uhr um `millis` Millisekunden vorwärts. Standard: 0ms (leert nur die Microtask-Queue).
- **`flush()`** – Führt alle ausstehenden Makrotasks aus, egal wie lange sie dauern würden. Nützlich, wenn die genaue Zeit egal ist.
- **`flushMicrotasks()`** – Leert nur die Microtask-Queue (z. B. aufgelöste Promises).
- **`discardPeriodicTasks()`** – Verwirft laufende `setInterval`-Tasks am Ende des Tests, um Fehlermeldungen zu vermeiden.

`fakeAsync` funktioniert hervorragend mit RxJS-Operatoren wie `debounceTime`, `delay`, `timer` – solange diese auf dem `AsyncScheduler` laufen, was der Standard ist.

## Aufgabe

Du hast einen `SearchService`, der HTTP-Anfragen mit einem `debounceTime(300)` verzögert, und eine `TypeaheadComponent`, die diesen Service nutzt. Schreibe vollständige, deterministisch laufende Tests mit `fakeAsync`.

### Schritte

1. **Service erstellen:** Implementiere einen `SearchService` mit einer Methode `search(query: string): Observable<string[]>`, die intern `debounceTime(300)` und `switchMap` auf ein `HttpClient.get()`-Call anwendet.

2. **Komponente erstellen:** Baue eine einfache `TypeaheadComponent`, die ein `<input>`-Feld hat. Bei jedem Keystroke wird `service.search(value)` aufgerufen und das Ergebnis in `results` gespeichert.

3. **Tests schreiben:** Schreibe Tests mit `fakeAsync` und `tick`:
   - Teste, dass bei einer Eingabe *vor* dem 300ms-Debounce **kein** HTTP-Call ausgelöst wird.
   - Teste, dass *nach* `tick(300)` der HTTP-Call gefeuert wird und `results` korrekt befüllt ist.
   - Teste den Fall mehrerer schneller Eingaben (nur der letzte Call landet dank `switchMap`).
   - Stelle sicher, dass kein `discardPeriodicTasks`-Fehler auftritt.

## Hints

<details>
<summary>Hint 1 – fakeAsync mit RxJS-Scheduler</summary>

RxJS-Operatoren wie `debounceTime` verwenden intern `AsyncScheduler`, der auf `setTimeout` aufbaut. In einer `fakeAsync`-Zone werden diese von `tick()` gesteuert. Du brauchst keinen speziellen Test-Scheduler zu injizieren.

```typescript
it('debounce fires after 300ms', fakeAsync(() => {
  component.onSearch('angular');
  tick(299); // Noch kein Call
  expect(httpMock.expectNone('/api/search?q=angular')).toBeUndefined();
  tick(1);   // Jetzt: 300ms erreicht
  httpMock.expectOne('/api/search?q=angular');
}));
```

</details>

<details>
<summary>Hint 2 – HTTP-Mock zurücksetzen</summary>

Mit `HttpClientTestingModule` und `HttpTestingController` kannst du ausstehende HTTP-Anfragen im Test kontrolliert beantworten. Rufe am Ende `httpMock.verify()` auf.

```typescript
afterEach(() => {
  httpMock.verify();
});

// Im Test: Anfrage manuell beantworten
const req = httpMock.expectOne('/api/search?q=angular');
req.flush(['Angular', 'Angular Material', 'AngularFire']);
```

</details>

<details>
<summary>Hint 3 – Mehrere Eingaben mit switchMap testen</summary>

`switchMap` abonniert nur den letzten Inner-Observable. Simuliere zwei Eingaben, beantworte nur die zweite HTTP-Anfrage und prüfe, dass die erste verworfen wurde.

```typescript
it('cancels previous request on new input', fakeAsync(() => {
  component.onSearch('a');
  tick(150); // Noch im Debounce
  component.onSearch('an');
  tick(300); // Zweiter Debounce abgelaufen

  // Nur eine Anfrage sollte offen sein (die zweite)
  const requests = httpMock.match('/api/search');
  expect(requests.length).toBe(1);
  expect(requests[0].request.params.get('q')).toBe('an');
  requests[0].flush(['Angular']);
}));
```

</details>

## Beispiellösung

```typescript
// search.service.ts
@Injectable({ providedIn: 'root' })
export class SearchService {
  private http = inject(HttpClient);

  search(query: string): Observable<string[]> {
    return this.http.get<string[]>('/api/search', {
      params: { q: query }
    });
  }
}

// typeahead.component.ts
@Component({
  selector: 'app-typeahead',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <input [formControl]="searchControl" placeholder="Suche..." />
    <ul>
      @for (result of results(); track result) {
        <li>{{ result }}</li>
      }
    </ul>
  `
})
export class TypeaheadComponent implements OnInit {
  private searchService = inject(SearchService);

  searchControl = new FormControl('');
  results = signal<string[]>([]);

  ngOnInit() {
    this.searchControl.valueChanges.pipe(
      debounceTime(300),
      switchMap(query => query
        ? this.searchService.search(query)
        : of([])
      ),
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(res => this.results.set(res));
  }

  private destroyRef = inject(DestroyRef);
}

// typeahead.component.spec.ts
describe('TypeaheadComponent', () => {
  let component: TypeaheadComponent;
  let fixture: ComponentFixture<TypeaheadComponent>;
  let httpMock: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [TypeaheadComponent, HttpClientTestingModule],
    }).compileComponents();

    fixture = TestBed.createComponent(TypeaheadComponent);
    component = fixture.componentInstance;
    httpMock = TestBed.inject(HttpTestingController);
    fixture.detectChanges();
  });

  afterEach(() => {
    httpMock.verify();
  });

  it('should not fire HTTP request before debounce', fakeAsync(() => {
    component.searchControl.setValue('ang');
    tick(299);
    httpMock.expectNone('/api/search');
  }));

  it('should fire HTTP request after 300ms debounce', fakeAsync(() => {
    component.searchControl.setValue('angular');
    tick(300);

    const req = httpMock.expectOne(r => r.url === '/api/search');
    expect(req.request.params.get('q')).toBe('angular');
    req.flush(['Angular', 'Angular Material']);

    fixture.detectChanges();
    expect(component.results()).toEqual(['Angular', 'Angular Material']);
  }));

  it('should cancel previous request via switchMap', fakeAsync(() => {
    component.searchControl.setValue('a');
    tick(150);
    component.searchControl.setValue('an');
    tick(300);

    // switchMap: nur die zweite Anfrage ist offen
    const reqs = httpMock.match(r => r.url === '/api/search');
    expect(reqs.length).toBe(1);
    expect(reqs[0].request.params.get('q')).toBe('an');
    reqs[0].flush(['Angular', 'AngularFire']);

    fixture.detectChanges();
    expect(component.results()).toEqual(['Angular', 'AngularFire']);
  }));

  it('should return empty results for empty input', fakeAsync(() => {
    component.searchControl.setValue('angular');
    tick(300);
    const req = httpMock.expectOne(r => r.url === '/api/search');
    req.flush(['Angular']);

    component.searchControl.setValue('');
    tick(300);
    httpMock.expectNone('/api/search'); // of([]) → kein HTTP-Call

    fixture.detectChanges();
    expect(component.results()).toEqual([]);
  }));
});
```

## Weiterführendes

- **`jasmine.clock()`** ist eine Alternative zu `fakeAsync`, aber weniger kompatibel mit Angulars Change Detection – bevorzuge `fakeAsync` in Angular-Tests.
- Für Tests mit `Promise`-basierten APIs (z. B. `async/await` in Services) kombiniere `fakeAsync` mit `flushMicrotasks()` statt `tick()`.
- Lese die offizielle Angular-Doku zu [Testing-Utility APIs](https://angular.dev/guide/testing/utility-apis) für weitere Methoden wie `getTestBed()` und `resetTestingModule()`.
- Für komplexe RxJS-Streams bieten sich ergänzend **Marble Tests** mit `TestScheduler` an (Dojo: `2026-08-10-rxjs-marble-testing.md`).
