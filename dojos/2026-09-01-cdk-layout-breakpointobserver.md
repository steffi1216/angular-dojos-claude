# Angular Dojo: CDK Layout & BreakpointObserver
**Datum:** 2026-09-01
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem Angular CDK `BreakpointObserver` reaktiv auf Viewport-Änderungen reagierst – ohne direkte Media-Query-Strings in Templates und ohne `window.matchMedia` manuell zu verwalten.

## Hintergrund & Theorie
Das Angular CDK stellt im Paket `@angular/cdk/layout` zwei zentrale Bausteine bereit:

**`BreakpointObserver`** ist ein Service, der intern `window.matchMedia` kapselt und für jeden registrierten Breakpoint einen `Observable<BreakpointState>` liefert. `BreakpointState` enthält ein `matches`-Flag und eine `breakpoints`-Map, die anzeigt, welche der beobachteten Queries aktiv sind.

**`Breakpoints`** ist eine Konstantensammlung mit Standard-Breakpoints nach Material Design (z. B. `Breakpoints.Handset`, `Breakpoints.Tablet`, `Breakpoints.Web`). Diese orientieren sich an gängigen Geräteklassen und ersparen das manuelle Aufschreiben von Pixel-Werten.

Im Zusammenspiel mit der `async`-Pipe oder `toSignal()` ergibt sich ein vollständig reaktiver, RxJS-basierter Ansatz für Responsive Design – ohne `ngIf`-Hacks auf Basis von `window.innerWidth` und ohne Lifecycle-Hook-Boilerplate.

Wichtig: `BreakpointObserver` ist **plattformunabhängig** – in SSR-Umgebungen fällt er gracefully auf `false` zurück, was ihn produktionssicher macht.

## Aufgabe
Erstelle eine Standalone-Komponente `ResponsiveLayoutComponent`, die ihr Layout reaktiv an den Viewport anpasst:

- Auf **Handset** (Mobilgerät): Einspaltiges Layout, Navigation als Bottom-Bar
- Auf **Tablet**: Zweispaltiges Layout, Navigation als Sidebar
- Auf **Web / Desktop**: Dreispaltiges Layout, Navigation als erweiterter Sidebar-Bereich

Nutze `BreakpointObserver` mit `Breakpoints`-Konstanten und `toSignal()`, damit die Komponente vollständig signalbasiert arbeitet.

### Schritte
1. Installiere `@angular/cdk` falls noch nicht vorhanden: `npm install @angular/cdk`
2. Erstelle `ResponsiveLayoutComponent` als Standalone-Komponente und importiere `BreakpointObserver` über `inject()`
3. Beobachte `[Breakpoints.Handset, Breakpoints.Tablet, Breakpoints.Web]` und leite aus dem `BreakpointState` ein `layoutMode`-Signal ab (`'mobile' | 'tablet' | 'desktop'`)
4. Erstelle ein zweites Signal `navPosition` (`'bottom' | 'sidebar' | 'sidebar-wide'`) basierend auf `layoutMode`
5. Rendere im Template unterschiedliche Layout-Strukturen mit `@switch` auf `layoutMode()`
6. Füge eine Resize-Demo hinzu: Zeige im Template dynamisch den aktiven Breakpoint-Namen an

## Hints
<details>
<summary>Hint 1 – BreakpointObserver abonnieren und in Signal umwandeln</summary>

```typescript
import { inject } from '@angular/core';
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';

const breakpointObserver = inject(BreakpointObserver);

const layout$ = breakpointObserver
  .observe([Breakpoints.Handset, Breakpoints.Tablet, Breakpoints.Web])
  .pipe(
    map(state => {
      if (state.breakpoints[Breakpoints.Handset]) return 'mobile';
      if (state.breakpoints[Breakpoints.Tablet]) return 'tablet';
      return 'desktop';
    })
  );

const layoutMode = toSignal(layout$, { initialValue: 'desktop' as const });
```
</details>

<details>
<summary>Hint 2 – computed() für abgeleitete Signale und @switch im Template</summary>

```typescript
import { computed } from '@angular/core';

const navPosition = computed(() => {
  const mode = layoutMode();
  if (mode === 'mobile') return 'bottom';
  if (mode === 'tablet') return 'sidebar';
  return 'sidebar-wide';
});
```

Im Template:
```html
@switch (layoutMode()) {
  @case ('mobile') {
    <div class="layout-mobile">
      <main>Inhalt</main>
      <nav class="bottom-nav">Navigation</nav>
    </div>
  }
  @case ('tablet') {
    <div class="layout-tablet">
      <nav class="sidebar">Navigation</nav>
      <main>Inhalt</main>
    </div>
  }
  @default {
    <div class="layout-desktop">
      <nav class="sidebar-wide">Navigation</nav>
      <main>Inhalt</main>
      <aside>Extras</aside>
    </div>
  }
}
```
</details>

## Beispiellösung
```typescript
import { Component, computed, inject } from '@angular/core';
import { BreakpointObserver, Breakpoints } from '@angular/cdk/layout';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs/operators';

type LayoutMode = 'mobile' | 'tablet' | 'desktop';
type NavPosition = 'bottom' | 'sidebar' | 'sidebar-wide';

@Component({
  selector: 'app-responsive-layout',
  standalone: true,
  template: `
    <div class="debug-bar">
      Aktiver Modus: <strong>{{ layoutMode() }}</strong> |
      Navigation: <strong>{{ navPosition() }}</strong>
    </div>

    @switch (layoutMode()) {
      @case ('mobile') {
        <div class="layout layout-mobile">
          <main class="content">
            <h2>Mobile Layout</h2>
            <p>Einspaltig, Navigation unten</p>
          </main>
          <nav class="nav bottom-nav">
            <span>Home</span><span>Suche</span><span>Profil</span>
          </nav>
        </div>
      }
      @case ('tablet') {
        <div class="layout layout-tablet">
          <nav class="nav sidebar">
            <ul><li>Home</li><li>Suche</li><li>Profil</li></ul>
          </nav>
          <main class="content">
            <h2>Tablet Layout</h2>
            <p>Zweispaltig, Navigation als Sidebar</p>
          </main>
        </div>
      }
      @default {
        <div class="layout layout-desktop">
          <nav class="nav sidebar-wide">
            <ul>
              <li>Home</li><li>Suche</li><li>Nachrichten</li>
              <li>Profil</li><li>Einstellungen</li>
            </ul>
          </nav>
          <main class="content">
            <h2>Desktop Layout</h2>
            <p>Dreispaltig, breite Navigation</p>
          </main>
          <aside class="aside">
            <h3>Extras</h3>
            <p>Empfehlungen, Widgets…</p>
          </aside>
        </div>
      }
    }
  `,
  styles: [`
    .debug-bar { padding: 8px 16px; background: #333; color: #fff; font-size: 13px; }
    .layout { display: flex; min-height: calc(100vh - 32px); }
    .layout-mobile { flex-direction: column; }
    .layout-tablet { flex-direction: row; }
    .layout-desktop { flex-direction: row; }
    .content { flex: 1; padding: 16px; }
    .bottom-nav { display: flex; justify-content: space-around; padding: 12px; background: #eee; }
    .sidebar { width: 200px; background: #f5f5f5; padding: 16px; }
    .sidebar-wide { width: 280px; background: #f0f0f0; padding: 16px; }
    .aside { width: 240px; background: #fafafa; padding: 16px; border-left: 1px solid #ddd; }
  `],
})
export class ResponsiveLayoutComponent {
  private breakpointObserver = inject(BreakpointObserver);

  private layout$ = this.breakpointObserver
    .observe([Breakpoints.Handset, Breakpoints.Tablet, Breakpoints.Web])
    .pipe(
      map(state => {
        if (state.breakpoints[Breakpoints.HandsetPortrait] ||
            state.breakpoints[Breakpoints.HandsetLandscape]) return 'mobile' as LayoutMode;
        if (state.breakpoints[Breakpoints.TabletPortrait] ||
            state.breakpoints[Breakpoints.TabletLandscape]) return 'tablet' as LayoutMode;
        return 'desktop' as LayoutMode;
      })
    );

  layoutMode = toSignal(this.layout$, { initialValue: 'desktop' as LayoutMode });

  navPosition = computed((): NavPosition => {
    const mode = this.layoutMode();
    if (mode === 'mobile') return 'bottom';
    if (mode === 'tablet') return 'sidebar';
    return 'sidebar-wide';
  });
}
```

Einbinden in `app.config.ts`:
```typescript
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';

export const appConfig: ApplicationConfig = {
  providers: [provideAnimationsAsync()],
};
```

## Weiterführendes
- **`LayoutModule`**: Neben `BreakpointObserver` enthält das CDK `MediaMatcher` für den direkten Zugriff auf `MediaQueryList`-Objekte – nützlich für imperative Szenarien außerhalb von Komponenten.
- **Custom Breakpoints**: Statt `Breakpoints.*` kannst du eigene Strings übergeben: `observe(['(min-width: 1440px)'])` – nützlich für Design-System-spezifische Breakpoints.
- **SSR-Kompatibilität**: In Server-Rendering-Umgebungen gibt `BreakpointObserver` immer `matches: false` zurück. Nutze `initialValue` in `toSignal()` bewusst, um das initiale Server-Rendering-Layout festzulegen.
- **Offizielle Docs**: [angular.io/cdk/layout](https://material.angular.io/cdk/layout/overview)
