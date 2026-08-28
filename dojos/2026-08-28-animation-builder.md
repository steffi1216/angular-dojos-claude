# Angular Dojo: AnimationBuilder – Programmatische Animationen
**Datum:** 2026-08-28
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du mit dem `AnimationBuilder`-Service Animationen imperativ und dynamisch zur Laufzeit erstellst und steuerst – unabhängig von den deklarativen `@Component`-Animations-Metadaten.

## Hintergrund & Theorie

Angular bietet zwei Wege für Animationen: Die **deklarative API** über die `animations`-Property im `@Component`-Decorator eignet sich für Zustands-Übergänge, die vorab bekannt sind. Der **`AnimationBuilder`-Service** hingegen ermöglicht es, Animationen vollständig imperativ zu erstellen und auszuführen – zur Laufzeit, mit dynamischen Werten und vollständiger Kontrolle über den `AnimationPlayer`.

`AnimationBuilder` erzeugt über `build()` eine `AnimationFactory`, aus der sich ein `AnimationPlayer` ableiten lässt. Der Player bietet die Methoden `play()`, `pause()`, `reset()`, `finish()` und `destroy()` sowie Event-Callbacks wie `onDone()` und `onStart()`.

**Typischer Anwendungsfall:**
- Animationen für dynamisch geladene Komponenten
- Interaktive Animationen, die auf Benutzereingaben (z. B. Mausposition) reagieren
- Programmatisches Abspielen von Highlight-Effekten (z. B. bei Suchergebnissen)

**Wichtig:** `BrowserAnimationsModule` (oder `provideAnimations()` in standalone Apps) muss importiert sein. `AnimationBuilder` wird über `inject()` oder den Konstruktor bezogen.

```typescript
import { AnimationBuilder, animate, style } from '@angular/animations';
```

## Aufgabe

Erstelle eine standalone Komponente `HighlightListComponent`, die eine Liste von Einträgen anzeigt. Beim Klick auf einen Eintrag soll dieser per `AnimationBuilder` animiert hervorgehoben werden (kurzes Aufleuchten in Gelb, dann zurück zur ursprünglichen Farbe). Die Animation soll imperativ ausgelöst werden, ohne `@Component`-Animations-Metadaten zu verwenden.

### Schritte

1. Erstelle eine neue standalone Komponente `HighlightListComponent` mit `signal`-basiertem Input für eine Liste von Strings.
2. Injiziere `AnimationBuilder` in die Komponente.
3. Schreibe eine Methode `highlightItem(element: HTMLElement)`, die mit `AnimationBuilder` eine Keyframe-Animation (gelber Hintergrund → transparenter Hintergrund) erstellt, einen `AnimationPlayer` erzeugt, diesen abspielt und nach Abschluss (`onDone`) wieder zerstört.
4. Binde die Klick-Events im Template an `highlightItem()`, übergib dabei das geklickte `$event.currentTarget` als `HTMLElement`.
5. Stelle sicher, dass mehrere schnelle Klicks korrekt funktionieren (Player des vorherigen Klicks ggf. stoppen).

## Hints

<details>
<summary>Hint 1 – AnimationFactory und Player erzeugen</summary>

```typescript
private animationBuilder = inject(AnimationBuilder);

private buildHighlightAnimation(): AnimationFactory {
  return this.animationBuilder.build([
    style({ backgroundColor: 'transparent' }),
    animate('200ms ease-in', style({ backgroundColor: '#fef08a' })),
    animate('600ms ease-out', style({ backgroundColor: 'transparent' })),
  ]);
}

highlightItem(el: HTMLElement): void {
  const factory = this.buildHighlightAnimation();
  const player = factory.create(el);
  player.play();
  player.onDone(() => player.destroy());
}
```

</details>

<details>
<summary>Hint 2 – Mehrere Klicks abfangen mit einer Map</summary>

```typescript
// Speichere den aktiven Player pro Element, um Überschneidungen zu vermeiden
private activePlayers = new Map<HTMLElement, AnimationPlayer>();

highlightItem(el: HTMLElement): void {
  // Laufende Animation auf diesem Element stoppen
  this.activePlayers.get(el)?.destroy();

  const factory = this.buildHighlightAnimation();
  const player = factory.create(el);
  this.activePlayers.set(el, player);

  player.play();
  player.onDone(() => {
    player.destroy();
    this.activePlayers.delete(el);
  });
}
```

</details>

## Beispiellösung

```typescript
// highlight-list.component.ts
import { Component, input, inject, OnDestroy } from '@angular/core';
import { AnimationBuilder, AnimationFactory, AnimationPlayer, animate, style } from '@angular/animations';

@Component({
  selector: 'app-highlight-list',
  standalone: true,
  template: `
    <ul>
      @for (item of items(); track item) {
        <li
          (click)="highlightItem($event.currentTarget as HTMLElement)"
          style="cursor: pointer; padding: 8px 12px; border-radius: 4px;"
        >
          {{ item }}
        </li>
      }
    </ul>
  `,
})
export class HighlightListComponent implements OnDestroy {
  items = input.required<string[]>();

  private animationBuilder = inject(AnimationBuilder);
  private activePlayers = new Map<HTMLElement, AnimationPlayer>();

  private buildHighlightAnimation(): AnimationFactory {
    return this.animationBuilder.build([
      style({ backgroundColor: 'transparent' }),
      animate('200ms ease-in', style({ backgroundColor: '#fef08a' })),
      animate('600ms 200ms ease-out', style({ backgroundColor: 'transparent' })),
    ]);
  }

  highlightItem(el: HTMLElement): void {
    this.activePlayers.get(el)?.destroy();

    const factory = this.buildHighlightAnimation();
    const player = factory.create(el);
    this.activePlayers.set(el, player);

    player.play();
    player.onDone(() => {
      player.destroy();
      this.activePlayers.delete(el);
    });
  }

  ngOnDestroy(): void {
    this.activePlayers.forEach(player => player.destroy());
    this.activePlayers.clear();
  }
}

// app.component.ts (Verwendung)
import { Component } from '@angular/core';
import { HighlightListComponent } from './highlight-list.component';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [HighlightListComponent],
  template: `
    <h2>Klicke auf einen Eintrag zum Hervorheben</h2>
    <app-highlight-list [items]="fruits" />
  `,
})
export class AppComponent {
  fruits = ['Apfel', 'Banane', 'Kirsche', 'Mango', 'Orange'];
}

// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideAnimations } from '@angular/platform-browser/animations';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, {
  providers: [provideAnimations()],
});
```

## Weiterführendes

- **`AnimationPlayer`-Lifecycle tiefer erkunden:** Kombiniere `onStart()`, `onDone()` und `getPosition()` für Progress-Tracking von Animationen.
- **Sequenzen und Gruppen:** Nutze `sequence()` und `group()` aus `@angular/animations` auch innerhalb des `AnimationBuilder` für komplexe Choreographien.
- **Offizielle Docs:** [Angular – Complex Animation Sequences](https://angular.dev/guide/animations/complex-sequences) für weiterführende Patterns.
