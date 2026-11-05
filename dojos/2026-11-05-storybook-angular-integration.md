# Angular Dojo: Storybook Integration – Komponenten isoliert dokumentieren und testen

**Datum:** 2026-11-05
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel

Du lernst, wie du Storybook in ein Angular-Projekt integrierst, um Komponenten isoliert zu entwickeln, zu dokumentieren und visuell zu testen – unabhängig von der laufenden Applikation.

## Hintergrund & Theorie

**Storybook** ist ein Open-Source-Tool für komponentenbasierte UI-Entwicklung. Jede Komponente wird in sogenannten *Stories* beschrieben – verschiedene Zustände und Variationen, die direkt im Browser rendert werden können, ohne die ganze App starten zu müssen.

Für Angular nutzt man das offizielle `@storybook/angular` Paket, das tief in Angulars `TestBed`-Infrastruktur integriert ist. Stories werden in `*.stories.ts`-Dateien geschrieben und folgen dem **Component Story Format (CSF 3.0)**:

```typescript
// Meta-Objekt beschreibt die Komponente
const meta: Meta<ButtonComponent> = {
  component: ButtonComponent,
};

// Jede exportierte Konstante ist eine Story
export const Primary: Story = {
  args: { label: 'Click me', variant: 'primary' },
};
```

Storybook kann auch Mocks für Services und Injectiontokens per `applicationConfig` oder `moduleMetadata`-Decorator bereitstellen. Das macht es ideal für Komponenten mit Abhängigkeiten.

Wichtige Konzepte:
- **Args** – reaktive Eingaben für `@Input`-Properties, live im UI bearbeitbar
- **Decorators** – Wrapper für Story-Templates (z.B. Padding, Theme)
- **Controls** – auto-generiertes UI aus TypeScript-Typen
- **Actions** – Output-Event-Logging im Storybook-Panel

## Aufgabe

Erstelle eine wiederverwendbare `AlertComponent` und dokumentiere sie vollständig mit Storybook – inklusive verschiedener Varianten, dynamischer Controls und einem Mock-Service.

### Schritte

1. **Komponente erstellen** – Erstelle eine `AlertComponent` mit folgenden Inputs:
   - `type: 'success' | 'warning' | 'error' | 'info'` (Standard: `'info'`)
   - `message: string`
   - `dismissible: boolean` (Standard: `false`)
   - Output: `dismissed: EventEmitter<void>`

   Style die Komponente per `:host`-Binding mit verschiedenen Hintergrundfarben je nach `type`.

2. **Storybook initialisieren** – Falls noch nicht vorhanden:
   ```bash
   npx storybook@latest init --type angular
   ```
   Stelle sicher, dass `@storybook/angular` und `@storybook/addon-essentials` im Projekt sind.

3. **Meta-Objekt definieren** – Erstelle `alert.stories.ts` neben der Komponente:
   ```typescript
   import type { Meta, StoryObj } from '@storybook/angular';
   import { AlertComponent } from './alert.component';

   const meta: Meta<AlertComponent> = {
     title: 'UI/Alert',
     component: AlertComponent,
     tags: ['autodocs'],
     argTypes: {
       type: {
         control: 'select',
         options: ['success', 'warning', 'error', 'info'],
       },
       dismissed: { action: 'dismissed' },
     },
   };

   export default meta;
   type Story = StoryObj<AlertComponent>;
   ```

4. **Stories schreiben** – Erstelle mindestens 4 Stories: `Success`, `Warning`, `Error`, `Info` – und eine `Dismissible`-Story:
   ```typescript
   export const Success: Story = {
     args: { type: 'success', message: 'Aktion erfolgreich!', dismissible: false },
   };

   export const Dismissible: Story = {
     args: { type: 'warning', message: 'Bitte überprüfen!', dismissible: true },
   };
   ```

5. **Service mocken** – Füge der Komponente einen `LogService` hinzu, der beim Schließen einen Log-Eintrag erzeugt. Mocke diesen Service in Storybook mit `applicationConfig`:
   ```typescript
   import { applicationConfig } from '@storybook/angular';
   import { importProvidersFrom } from '@angular/core';

   export const WithMockedService: Story = {
     decorators: [
       applicationConfig({
         providers: [
           { provide: LogService, useValue: { log: (msg: string) => console.log('[Mock]', msg) } },
         ],
       }),
     ],
     args: { type: 'error', message: 'Kritischer Fehler!', dismissible: true },
   };
   ```

6. **Storybook starten** und alle Stories in der UI prüfen:
   ```bash
   npm run storybook
   ```

## Hints

<details>
<summary>Hint 1 – AlertComponent Grundstruktur</summary>

```typescript
@Component({
  selector: 'app-alert',
  standalone: true,
  template: `
    <div [class]="'alert alert--' + type">
      {{ message }}
      @if (dismissible) {
        <button (click)="onDismiss()">×</button>
      }
    </div>
  `,
  styles: [`
    .alert { padding: 12px 16px; border-radius: 4px; display: flex; justify-content: space-between; }
    .alert--success { background: #d4edda; color: #155724; }
    .alert--warning { background: #fff3cd; color: #856404; }
    .alert--error   { background: #f8d7da; color: #721c24; }
    .alert--info    { background: #d1ecf1; color: #0c5460; }
  `],
})
export class AlertComponent {
  type = input<'success' | 'warning' | 'error' | 'info'>('info');
  message = input.required<string>();
  dismissible = input(false);
  dismissed = output<void>();

  onDismiss() { this.dismissed.emit(); }
}
```

</details>

<details>
<summary>Hint 2 – `autodocs` und Controls automatisch generieren</summary>

Mit `tags: ['autodocs']` im Meta-Objekt generiert Storybook automatisch eine Dokumentationsseite für die Komponente. Die **Controls** werden aus den TypeScript-Typen der `@Input`-Properties abgeleitet.

Mit `argTypes` kann man die automatisch generierten Controls überschreiben oder verfeinern, z.B. einen `select`-Control für Union-Types erzwingen:

```typescript
argTypes: {
  type: { control: 'select', options: ['success', 'warning', 'error', 'info'] },
  message: { control: 'text' },
  dismissible: { control: 'boolean' },
}
```

Für Outputs (`EventEmitter` oder Signal `output()`) nutze `{ action: 'eventName' }`, um Events im Actions-Panel zu loggen.

</details>

## Beispiellösung

```typescript
// alert.stories.ts
import type { Meta, StoryObj } from '@storybook/angular';
import { applicationConfig } from '@storybook/angular';
import { AlertComponent } from './alert.component';
import { LogService } from '../services/log.service';

const meta: Meta<AlertComponent> = {
  title: 'UI/Alert',
  component: AlertComponent,
  tags: ['autodocs'],
  argTypes: {
    type: { control: 'select', options: ['success', 'warning', 'error', 'info'] },
    dismissed: { action: 'dismissed' },
  },
  args: {
    message: 'Dies ist eine Beispielmeldung.',
    dismissible: false,
    type: 'info',
  },
};

export default meta;
type Story = StoryObj<AlertComponent>;

export const Info: Story = {};

export const Success: Story = {
  args: { type: 'success', message: 'Speichern erfolgreich!' },
};

export const Warning: Story = {
  args: { type: 'warning', message: 'Bitte alle Felder ausfüllen.' },
};

export const Error: Story = {
  args: { type: 'error', message: 'Verbindung unterbrochen.' },
};

export const Dismissible: Story = {
  args: { type: 'warning', message: 'Sitzung läuft bald ab.', dismissible: true },
};

export const WithMockedLogService: Story = {
  decorators: [
    applicationConfig({
      providers: [
        {
          provide: LogService,
          useValue: { log: (msg: string) => console.log('[MockLogService]', msg) },
        },
      ],
    }),
  ],
  args: { type: 'error', message: 'Kritischer Fehler!', dismissible: true },
};
```

## Weiterführendes

- **Interaction Tests mit Storybook**: Mit `@storybook/addon-interactions` und `@storybook/testing-library` lassen sich Klick-Events und Formulareingaben in Stories simulieren und als automatisierte Tests ausführen (`play`-Funktion).
- **Visual Regression Testing**: Tools wie [Chromatic](https://www.chromatic.com/) (von den Storybook-Machern) oder [Percy](https://percy.io/) snapshotten deine Stories und schlagen bei visuellen Änderungen an – ideal für CI/CD-Pipelines.
- Offizielle Doku: [storybook.js.org/docs/angular](https://storybook.js.org/docs/angular)
