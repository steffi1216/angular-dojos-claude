# Angular Dojo: Custom CLI Builders
**Datum:** 2026-10-22
**Dauer:** ~25 Minuten
**Level:** Fortgeschritten

## Lernziel
Du lernst, wie du eigene Angular CLI Builder mit der `@angular-devkit/architect` API erstellst, um benutzerdefinierte Build-Targets in `angular.json` zu integrieren und wiederverwendbare Tooling-Workflows zu automatisieren.

## Hintergrund & Theorie

Angular CLI baut auf dem **Architect** Framework auf – einem task-basierten Build-System, das sogenannte **Builder** als Ausführungseinheiten nutzt. Jeder `ng`-Befehl (`ng build`, `ng test`, `ng lint`) ist intern ein Builder.

Ein Builder ist eine Funktion, die:
1. Ein **`BuilderContext`**-Objekt erhält (Zugriff auf Workspace, Logger, Signale)
2. Optionen aus `angular.json` entgegennimmt (typsicher via JSON Schema)
3. ein `BuilderOutput` Observable oder Promise zurückgibt (`{ success: boolean }`)

```typescript
// Struktur eines Builders
import { createBuilder, BuilderContext, BuilderOutput } from '@angular-devkit/architect';

export default createBuilder<MyOptions>(async (options, context) => {
  context.logger.info(`Running custom builder with: ${JSON.stringify(options)}`);
  // ... Task-Logik
  return { success: true };
});
```

Builder werden in einer `package.json` des Builder-Pakets über das `builders`-Feld registriert, das auf eine **builders.json** Datei zeigt. In `angular.json` werden sie dann als `builder: "mein-paket:mein-target"` referenziert.

Typische Anwendungsfälle: Dateien kopieren/transformieren, CI-Validierungen, Codegenerierung, Asset-Komprimierung, Deployment-Skripte.

## Aufgabe

Erstelle einen lokalen Custom Builder namens `file-size-checker`, der nach einem `ng build` die erzeugten Bundle-Größen gegen konfigurierbare Limits prüft und den Build fehlschlägt, wenn ein Limit überschritten wird.

### Schritte

1. **Builder-Paket anlegen** – Erstelle das Verzeichnis `tools/builders/` im Workspace. Lege `package.json`, `builders.json` und `src/file-size-checker/index.ts` an.

2. **JSON Schema definieren** – Erstelle `src/file-size-checker/schema.json` mit den Optionen `outputPath: string` und `limits: Record<string, number>` (Dateinamen-Glob → max. Bytes).

3. **Builder implementieren** – Lese in `index.ts` alle Dateien im `outputPath`, prüfe jede Datei gegen das passende Limit und gib `{ success: false }` zurück, wenn ein Limit überschritten wird. Nutze `context.logger.error()` für aussagekräftige Fehlermeldungen.

4. **Builder registrieren** – Trage den Builder in `builders.json` ein und verlinke das Paket über `npm link` oder einen lokalen `file:`-Pfad in `package.json` des Angular-Projekts.

5. **`angular.json` konfigurieren** – Füge ein neues Target `check-bundle-sizes` hinzu, das deinen Builder verwendet, und rufe es mit `ng run my-app:check-bundle-sizes` auf.

## Hints

<details>
<summary>Hint 1 – Dateistruktur des Builder-Pakets</summary>

```
tools/builders/
├── package.json          ← "builders": "./builders.json"
├── builders.json         ← Builder-Registry
└── src/
    └── file-size-checker/
        ├── index.ts      ← Builder-Implementierung
        └── schema.json   ← Optionen-Schema
```

`builders.json`:
```json
{
  "$schema": "@angular-devkit/architect/src/builders-schema.json",
  "builders": {
    "file-size-checker": {
      "implementation": "./src/file-size-checker/index",
      "schema": "./src/file-size-checker/schema.json",
      "description": "Checks bundle sizes against configured limits."
    }
  }
}
```

`package.json` des Builder-Pakets:
```json
{
  "name": "my-workspace-builders",
  "version": "1.0.0",
  "builders": "./builders.json"
}
```

</details>

<details>
<summary>Hint 2 – Builder-Implementierung mit Node.js fs</summary>

```typescript
import { createBuilder, BuilderContext, BuilderOutput } from '@angular-devkit/architect';
import * as fs from 'fs';
import * as path from 'path';
import { minimatch } from 'minimatch';

interface Options {
  outputPath: string;
  limits: Record<string, number>;
}

export default createBuilder<Options>(async (options, context): Promise<BuilderOutput> => {
  const root = context.workspaceRoot;
  const outDir = path.resolve(root, options.outputPath);

  if (!fs.existsSync(outDir)) {
    context.logger.error(`Output path not found: ${outDir}`);
    return { success: false };
  }

  const files = fs.readdirSync(outDir);
  let passed = true;

  for (const file of files) {
    const filePath = path.join(outDir, file);
    const stat = fs.statSync(filePath);
    if (!stat.isFile()) continue;

    for (const [pattern, maxBytes] of Object.entries(options.limits)) {
      if (minimatch(file, pattern)) {
        if (stat.size > maxBytes) {
          context.logger.error(
            `❌ ${file}: ${stat.size} bytes exceeds limit of ${maxBytes} bytes (pattern: ${pattern})`
          );
          passed = false;
        } else {
          context.logger.info(`✅ ${file}: ${stat.size} bytes (limit: ${maxBytes} bytes)`);
        }
      }
    }
  }

  return { success: passed };
});
```

</details>

## Beispiellösung

```typescript
// tools/builders/src/file-size-checker/schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema",
  "title": "File Size Checker Options",
  "type": "object",
  "properties": {
    "outputPath": {
      "type": "string",
      "description": "Path to the build output directory."
    },
    "limits": {
      "type": "object",
      "description": "Map of filename globs to max allowed bytes.",
      "additionalProperties": { "type": "number" }
    }
  },
  "required": ["outputPath", "limits"]
}

// angular.json – neues Target im "architect"-Block der App
// "check-bundle-sizes": {
//   "builder": "my-workspace-builders:file-size-checker",
//   "options": {
//     "outputPath": "dist/my-app/browser",
//     "limits": {
//       "main.*.js":   500000,
//       "polyfills.*.js": 50000,
//       "*.css":       80000
//     }
//   }
// }
```

```typescript
// tools/builders/src/file-size-checker/index.ts
import { createBuilder, BuilderContext, BuilderOutput } from '@angular-devkit/architect';
import * as fs from 'fs';
import * as path from 'path';

interface FileSizeCheckerOptions {
  outputPath: string;
  limits: Record<string, number>;
}

function matchesGlob(filename: string, pattern: string): boolean {
  const regexStr = pattern
    .replace(/\./g, '\\.')
    .replace(/\*/g, '[^/]*');
  return new RegExp(`^${regexStr}$`).test(filename);
}

export default createBuilder<FileSizeCheckerOptions>(
  async (options, context: BuilderContext): Promise<BuilderOutput> => {
    const outDir = path.resolve(context.workspaceRoot, options.outputPath);
    context.logger.info(`Checking bundle sizes in: ${outDir}`);

    if (!fs.existsSync(outDir)) {
      context.logger.error(`Output directory not found: ${outDir}`);
      return { success: false };
    }

    const files = fs.readdirSync(outDir).filter(f =>
      fs.statSync(path.join(outDir, f)).isFile()
    );

    let allPassed = true;

    for (const file of files) {
      const size = fs.statSync(path.join(outDir, file)).size;
      const kb = (size / 1024).toFixed(1);

      for (const [pattern, maxBytes] of Object.entries(options.limits)) {
        if (matchesGlob(file, pattern)) {
          if (size > maxBytes) {
            context.logger.error(
              `FAIL ${file}: ${kb} KB > ${(maxBytes / 1024).toFixed(1)} KB limit`
            );
            allPassed = false;
          } else {
            context.logger.info(
              `OK   ${file}: ${kb} KB / ${(maxBytes / 1024).toFixed(1)} KB`
            );
          }
        }
      }
    }

    if (allPassed) {
      context.logger.info('All bundle size checks passed!');
    }

    return { success: allPassed };
  }
);
```

## Weiterführendes

- **`scheduleBuilder` & `scheduleTarget`**: Aus einem Builder heraus weitere Builder oder Targets aufrufen (z. B. zuerst `build`, dann `check-bundle-sizes`). Nützlich für zusammengesetzte Pipelines.
- **Offizielle Doku**: [Angular CLI Builders Guide](https://angular.dev/tools/cli/cli-builders) – erklärt die vollständige API inklusive Progress-Reporting und Input-Validierung.
- **Nx Custom Executors**: Das Executor-Konzept in Nx baut auf demselben Architect-Fundament auf – Custom Builder-Wissen lässt sich direkt übertragen.
