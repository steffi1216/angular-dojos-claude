# Angular Dojo: RouterTestingHarness
**Datum:** 2026-09-18
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du Angular-Komponenten, die auf Routing angewiesen sind, mit dem modernen `RouterTestingHarness` aus `@angular/router/testing` präzise und komfortabel testest – ohne echten Browser und ohne den älteren `RouterTestingModule`.

## Hintergrund & Theorie

Seit Angular 15 gibt es `RouterTestingHarness` als empfohlenen Ersatz für `RouterTestingModule`. Das Harness startet einen vollständigen In-Memory-Router, der mit `provideRouter()` konfiguriert wird, und navigiert dann zu einer bestimmten Route. Dabei wird die aktivierte Komponente direkt gerendert und ist über `harness.fixture` oder `harness.routeNativeElement` zugänglich.

**Vorteile gegenüber dem alten Ansatz:**
- Kein `RouterTestingModule` nötig – funktioniert mit `provideRouter()` (standalone API)
- Navigation erfolgt synchron innerhalb von `fakeAsync` / `await`
- Route-Parameter, Guards, Resolvers und Lazy Routes werden vollständig unterstützt
- Einfaches Zugreifen auf die gerenderte Komponente ohne manuelles `fixture.detectChanges()`

**Grundprinzip:**
```typescript
const harness = await RouterTestingHarness.create('/products/42');
const component = harness.routeDebugElement!.componentInstance as ProductDetailComponent;
```

Die Navigation ist `async`, da der Router Guards und Resolvers auflöst. Das Harness kümmert sich intern um `detectChanges()`.

## Aufgabe

Baue eine `ProductDetailComponent`, die einen Produkt-Resolver verwendet, und teste sie vollständig mit `RouterTestingHarness`. Der Resolver liefert Produktdaten basierend auf dem `:id`-Parameter aus der URL.

### Schritte

1. **Erstelle einen einfachen `ProductResolver`**, der anhand der Route-Param `id` ein Produkt zurückgibt. Nutze eine `ProductService`-Stub-Implementierung mit fixen Testdaten.

2. **Erstelle `ProductDetailComponent`** als Standalone-Komponente. Sie empfängt das aufgelöste Produkt über `ActivatedRoute.data` (oder als Signal via `input()` mit Router Input Bindings) und zeigt Name und Preis an.

3. **Schreibe zwei Unit-Tests** mit `RouterTestingHarness`:
   - Test 1: Navigiere zu `/products/1` – prüfe, dass der Produktname korrekt im DOM erscheint.
   - Test 2: Navigiere zu `/products/999` (unbekannte ID) – prüfe, dass die Komponente auf einen Redirect oder einen Fehlerzustand reagiert (Resolver gibt `null` zurück und ein Guard leitet um).

4. **Bonus:** Lade die Route lazy (mit `loadComponent`) und stelle sicher, dass der Test weiterhin besteht.

## Hints

<details>
<summary>Hint 1 – TestBed-Konfiguration</summary>

Verwende `provideRouter()` mit `withComponentInputBinding()`, damit Resolver-Daten direkt als `@Input()` in die Komponente fließen:

```typescript
TestBed.configureTestingModule({
  providers: [
    provideRouter(
      [
        {
          path: 'products/:id',
          component: ProductDetailComponent,
          resolve: { product: productResolver },
        },
      ],
      withComponentInputBinding()
    ),
    { provide: ProductService, useValue: mockProductService },
  ],
});
```

</details>

<details>
<summary>Hint 2 – Harness erstellen und navigieren</summary>

```typescript
const harness = await RouterTestingHarness.create();
// Erste Navigation:
await harness.navigateByUrl('/products/1');

// Danach kannst du auf das native Element zugreifen:
const el = harness.routeNativeElement!;
expect(el.querySelector('h1')?.textContent).toContain('Laptop');

// Oder auf die Komponenteninstanz:
const comp = harness.routeDebugElement!.componentInstance as ProductDetailComponent;
expect(comp.product.name).toBe('Laptop');
```

Wichtig: `harness.navigateByUrl` ist asynchron. In `fakeAsync`-Tests rufe danach `tick()` auf; in `async`-Tests genügt `await`.

</details>

<details>
<summary>Hint 3 – Redirect bei unbekanntem Produkt testen</summary>

Im Resolver kannst du bei `null`-Produkt mit `inject(Router).navigate(['/not-found'])` umleiten und `null` zurückgeben. Im Guard-Ansatz liefert der Resolver ein `RedirectCommand`:

```typescript
export const productResolver: ResolveFn<Product | null> = (route) => {
  const service = inject(ProductService);
  const router = inject(Router);
  const product = service.getById(+route.paramMap.get('id')!);
  if (!product) {
    return new RedirectCommand(router.parseUrl('/not-found'));
  }
  return product;
};
```

Im Test:
```typescript
await harness.navigateByUrl('/products/999');
expect(TestBed.inject(Router).url).toBe('/not-found');
```

</details>

## Beispiellösung

```typescript
// product.service.ts
@Injectable({ providedIn: 'root' })
export class ProductService {
  private products = [
    { id: 1, name: 'Laptop', price: 999 },
    { id: 2, name: 'Maus', price: 29 },
  ];

  getById(id: number) {
    return this.products.find(p => p.id === id) ?? null;
  }
}

// product.resolver.ts
export const productResolver: ResolveFn<Product | null> = (route) => {
  const service = inject(ProductService);
  const router = inject(Router);
  const product = service.getById(+route.paramMap.get('id')!);
  if (!product) {
    return new RedirectCommand(router.parseUrl('/not-found'));
  }
  return product;
};

// product-detail.component.ts
@Component({
  standalone: true,
  selector: 'app-product-detail',
  template: `
    @if (product) {
      <h1>{{ product.name }}</h1>
      <p>Preis: {{ product.price | currency:'EUR' }}</p>
    }
  `,
  imports: [CurrencyPipe],
})
export class ProductDetailComponent {
  product = input<Product | null>(null);
}

// product-detail.component.spec.ts
describe('ProductDetailComponent', () => {
  let harness: RouterTestingHarness;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      providers: [
        provideRouter(
          [
            {
              path: 'products/:id',
              component: ProductDetailComponent,
              resolve: { product: productResolver },
            },
            {
              path: 'not-found',
              component: NotFoundComponent,
            },
          ],
          withComponentInputBinding()
        ),
      ],
    }).compileComponents();
  });

  it('zeigt den Produktnamen an', async () => {
    harness = await RouterTestingHarness.create('/products/1');
    const el = harness.routeNativeElement!;
    expect(el.querySelector('h1')?.textContent?.trim()).toBe('Laptop');
  });

  it('leitet bei unbekannter ID auf /not-found um', async () => {
    harness = await RouterTestingHarness.create('/products/999');
    expect(TestBed.inject(Router).url).toBe('/not-found');
  });
});
```

## Weiterführendes

- Kombiniere `RouterTestingHarness` mit `ActivatedRouteSnapshot`-Mocking, um komplexe Guard-Ketten zu testen ohne den gesamten Router-Stack hochzufahren.
- Seit Angular 17 lässt sich `RouterTestingHarness` mit `TestBed.runInInjectionContext` kombinieren, um Resolver-Funktionen isoliert zu testen – ideal, wenn der Resolver eigenständige Logik enthält.
- Offizielle Doku: [angular.dev – RouterTestingHarness](https://angular.dev/api/router/testing/RouterTestingHarness)
