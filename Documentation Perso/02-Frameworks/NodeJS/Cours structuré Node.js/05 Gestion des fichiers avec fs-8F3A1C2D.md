# Formation Node.js — Gestion des fichiers avec `fs`

> **Public** : développeurs Node.js (débutant → intermédiaire)  
> **Pré-requis** : JavaScript ES6, notions de callbacks/Promises, terminal, npm  
> **Durée suggérée** : 3h à 4h (avec ateliers)  
> **Objectif** : savoir lire/écrire/manipuler des fichiers et dossiers de façon fiable, performante et sécurisée avec le module `fs`.

---

## Plan de la formation

1. [Introduction : pourquoi `fs` ?](#1-introduction--pourquoi-fs-)
2. [Installer le contexte : `fs`, `fs/promises`, `path`](#2-installer-le-contexte--fs-fspromises-path)
3. [Lire un fichier](#3-lire-un-fichier)
4. [Écrire dans un fichier](#4-écrire-dans-un-fichier)
5. [Manipuler chemins et encodages](#5-manipuler-chemins-et-encodages)
6. [Travailler avec les dossiers](#6-travailler-avec-les-dossiers)
7. [Lister, statuer, supprimer, renommer](#7-lister-statuer-supprimer-renommer)
8. [Copier et déplacer des fichiers](#8-copier-et-déplacer-des-fichiers)
9. [Streams : gros fichiers et performance](#9-streams--gros-fichiers-et-performance)
10. [Surveiller des changements : `fs.watch`](#10-surveiller-des-changements--fswatch)
11. [Droits, erreurs et robustesse](#11-droits-erreurs-et-robustesse)
12. [Cas pratiques (ateliers guidés)](#12-cas-pratiques-ateliers-guidés)
13. [Checklist & bonnes pratiques](#13-checklist--bonnes-pratiques)

---

## 1) Introduction : pourquoi `fs` ?

Le module **core** Node.js `fs` (File System) permet de :

- **Lire** des fichiers (`readFile`, `createReadStream`)
- **Écrire** des fichiers (`writeFile`, `appendFile`, `createWriteStream`)
- **Créer/supprimer** des fichiers et dossiers (`mkdir`, `rm`, `unlink`)
- **Lister** des répertoires (`readdir`)
- **Obtenir des infos** (`stat`, `lstat`)
- **Modifier** des permissions (`chmod`) et timestamps (`utimes`)
- **Surveiller** les changements (`watch`)

> **Point clé** : Node est très utilisé côté back-end. Gérer des fichiers sert pour l’upload, génération de rapports PDF/CSV, logs, caches, traitements de médias, scripts d’automatisation, etc.

---

## 2) Installer le contexte : `fs`, `fs/promises`, `path`

### 2.1 Importer `fs`

Deux familles d’API coexistent :

- API **callback** : `fs.readFile(path, cb)`
- API **Promises** : `fs/promises` (recommandé) : `await fs.readFile(path)`

```js
// API callback
import fs from "node:fs";

// API Promises
import fsp from "node:fs/promises";

// Manipulation de chemins (essentiel)
import path from "node:path";
```

### 2.2 Le piège des chemins relatifs

- Un chemin relatif (`./data/file.txt`) est relatif au **répertoire courant** (`process.cwd()`), pas au fichier JS.
- Pour construire un chemin **fiable**, utilisez `import.meta.url` + `path`.

```js
import path from "node:path";
import { fileURLToPath } from "node:url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const filePath = path.join(__dirname, "data", "notes.txt");
```

---

## 3) Lire un fichier

### 3.1 Lire un fichier complet : `readFile`

#### Avec `fs/promises` (recommandé)

```js
import fsp from "node:fs/promises";

const content = await fsp.readFile("./data/hello.txt", "utf8");
console.log(content);
```

- Le 2ᵉ paramètre (`"utf8"`) indique que vous voulez une **string**.
- Sans encodage, vous obtenez un **Buffer**.

#### Lire en Buffer

```js
const buf = await fsp.readFile("./data/image.png");
console.log(buf.length);
```

### 3.2 Gérer les erreurs

```js
try {
  const txt = await fsp.readFile("./data/missing.txt", "utf8");
  console.log(txt);
} catch (err) {
  if (err.code === "ENOENT") {
    console.error("Fichier introuvable");
  } else {
    console.error("Erreur lecture:", err);
  }
}
```

**Codes utiles** :
- `ENOENT` : fichier/dossier inexistant
- `EACCES` : permission refusée
- `EISDIR` : on essaie de lire un dossier comme un fichier

### 3.3 Lire partiellement : `open`, `read`

Pour des cas avancés (lecture à un offset précis), vous pouvez ouvrir un descripteur :

```js
import fsp from "node:fs/promises";

const file = await fsp.open("./data/big.bin", "r");
try {
  const buffer = Buffer.alloc(1024);
  const { bytesRead } = await file.read(buffer, 0, 1024, 0);
  console.log("Lu", bytesRead, "octets");
} finally {
  await file.close();
}
```

---

## 4) Écrire dans un fichier

### 4.1 Écrire : `writeFile`

```js
import fsp from "node:fs/promises";

await fsp.writeFile("./out/report.txt", "Rapport\n", "utf8");
```

> `writeFile` **remplace** le fichier s’il existe.

### 4.2 Ajouter à la fin : `appendFile`

```js
await fsp.appendFile("./out/log.txt", "Nouvelle ligne\n", "utf8");
```

### 4.3 Contrôler le mode d’ouverture (flags)

`writeFile` peut recevoir des options :

```js
await fsp.writeFile("./out/data.json", JSON.stringify({ ok: true }), {
  encoding: "utf8",
  flag: "wx", // échoue si le fichier existe déjà
});
```

Flags fréquents :
- `"w"` : write (crée/remplace)
- `"a"` : append
- `"wx"` : write + fail if exists (évite d’écraser)
- `"ax"` : append + fail if exists

### 4.4 Écriture atomique (approche)

Pour éviter un fichier partiellement écrit (crash, interruption), pratique courante :

1) écrire dans un fichier temporaire
2) renommer (opération souvent atomique sur un même disque)

```js
import fsp from "node:fs/promises";
import path from "node:path";

async function writeAtomic(filePath, content) {
  const tmpPath = filePath + ".tmp";
  await fsp.writeFile(tmpPath, content, "utf8");
  await fsp.rename(tmpPath, filePath);
}

await writeAtomic("./out/config.json", JSON.stringify({ v: 1 }, null, 2));
```

---

## 5) Manipuler chemins et encodages

### 5.1 `path.join`, `path.resolve`

- `path.join` assemble proprement (`/` vs `\\`)
- `path.resolve` produit un chemin absolu

```js
import path from "node:path";

const p1 = path.join("data", "users", "1.json");
const p2 = path.resolve("data", "users", "1.json");
```

### 5.2 Encodages

- Texte : `utf8` (standard)
- Binaire : Buffer

```js
const txt = await fsp.readFile("./data/a.txt", "utf8");
const bin = await fsp.readFile("./data/a.bin");
```

### 5.3 Attention aux chemins utilisateurs (sécurité)

Quand un utilisateur fournit un nom de fichier (ex: téléchargement), attention à la **path traversal** : `../../etc/passwd`.

Mesures :
- n’autoriser que des noms de fichiers attendus (whitelist)
- normaliser et vérifier que le chemin final reste dans un dossier racine

```js
import path from "node:path";

function safeJoin(baseDir, userPath) {
  const target = path.resolve(baseDir, userPath);
  const base = path.resolve(baseDir);
  if (!target.startsWith(base + path.sep)) {
    throw new Error("Path traversal detectée");
  }
  return target;
}
```

---

## 6) Travailler avec les dossiers

### 6.1 Créer un dossier : `mkdir`

```js
import fsp from "node:fs/promises";

await fsp.mkdir("./out/reports", { recursive: true });
```

- `recursive: true` crée aussi les parents si nécessaire.

### 6.2 Vérifier si un chemin existe

Node ne fournit pas une fonction `exists` en Promises directe recommandée (il existe `fs.existsSync` mais pas idéal).

Approche fiable : `access`.

```js
import fsp from "node:fs/promises";
import fs from "node:fs";

async function exists(p) {
  try {
    await fsp.access(p, fs.constants.F_OK);
    return true;
  } catch {
    return false;
  }
}

console.log(await exists("./out"));
```

---

## 7) Lister, statuer, supprimer, renommer

### 7.1 Lister un dossier : `readdir`

```js
import fsp from "node:fs/promises";

const entries = await fsp.readdir("./data");
console.log(entries);
```

Option utile : récupérer directement le type d’entrée :

```js
const entries = await fsp.readdir("./data", { withFileTypes: true });
for (const e of entries) {
  console.log(e.name, e.isFile() ? "file" : e.isDirectory() ? "dir" : "other");
}
```

### 7.2 Obtenir des infos : `stat` / `lstat`

```js
const st = await fsp.stat("./data/hello.txt");
console.log(st.size, st.mtime);
```

- `stat` suit les symlinks
- `lstat` donne l’info du symlink lui-même

### 7.3 Renommer / déplacer : `rename`

```js
await fsp.rename("./out/a.txt", "./out/b.txt");
```

> Sur un même disque, `rename` est souvent très rapide. Entre disques/volumes, cela peut échouer : dans ce cas, copier puis supprimer.

### 7.4 Supprimer

- Supprimer un fichier : `unlink`
- Supprimer dossier (et contenu) : `rm` avec `recursive`

```js
await fsp.unlink("./out/old.txt");

await fsp.rm("./out/tmp", { recursive: true, force: true });
```

---

## 8) Copier et déplacer des fichiers

### 8.1 Copier : `copyFile`

```js
import fsp from "node:fs/promises";

await fsp.copyFile("./data/source.txt", "./out/backup.txt");
```

### 8.2 Déplacer “robuste” (fallback)

```js
import fsp from "node:fs/promises";

async function moveFile(src, dest) {
  try {
    await fsp.rename(src, dest);
  } catch (err) {
    if (err.code === "EXDEV") {
      // cross-device: copier puis supprimer
      await fsp.copyFile(src, dest);
      await fsp.unlink(src);
    } else {
      throw err;
    }
  }
}
```

---

## 9) Streams : gros fichiers et performance

Lire/écrire tout un fichier avec `readFile/writeFile` charge tout en mémoire. Pour des gros fichiers, utilisez les **streams**.

### 9.1 Lire en stream : `createReadStream`

```js
import fs from "node:fs";

const rs = fs.createReadStream("./data/big.log", { encoding: "utf8" });
rs.on("data", chunk => {
  console.log("chunk:", chunk.length);
});
rs.on("end", () => console.log("Terminé"));
rs.on("error", err => console.error(err));
```

### 9.2 Copier via `pipe`

```js
import fs from "node:fs";

const rs = fs.createReadStream("./data/video.mp4");
const ws = fs.createWriteStream("./out/video-copy.mp4");

rs.pipe(ws);
```

### 9.3 Copier via `stream/promises.pipeline` (recommandé)

`pipeline` gère mieux la propagation d’erreurs.

```js
import fs from "node:fs";
import { pipeline } from "node:stream/promises";

await pipeline(
  fs.createReadStream("./data/big.csv"),
  fs.createWriteStream("./out/big-copy.csv")
);
```

### 9.4 Transformer un flux (ex: uppercase)

```js
import fs from "node:fs";
import { Transform } from "node:stream";
import { pipeline } from "node:stream/promises";

const upper = new Transform({
  transform(chunk, _enc, cb) {
    cb(null, chunk.toString("utf8").toUpperCase());
  },
});

await pipeline(
  fs.createReadStream("./data/input.txt"),
  upper,
  fs.createWriteStream("./out/output.txt")
);
```

---

## 10) Surveiller des changements : `fs.watch`

Utile pour relancer un traitement quand un fichier change (devtools, import automatique, etc.).

```js
import fs from "node:fs";

const watcher = fs.watch("./data", (eventType, filename) => {
  console.log("Event:", eventType, "File:", filename);
});

// plus tard
// watcher.close();
```

**Attention** : `fs.watch` dépend de l’OS et peut être “bruyant” (plusieurs événements). Pour des besoins avancés, on utilise souvent des bibliothèques comme `chokidar`.

---

## 11) Droits, erreurs et robustesse

### 11.1 Permissions et umask

- `chmod` modifie les permissions (Unix)
- `mode` lors de la création (rarement nécessaire)

```js
await fsp.chmod("./out/script.sh", 0o755);
```

### 11.2 Concurrence : éviter d’écraser

- `flag: "wx"` pour éviter les écrasements
- locks (plus complexe) si plusieurs processus écrivent

### 11.3 Valider les entrées

- Ne jamais écrire directement un chemin fourni par un client sans validation.
- Éviter de stocker des fichiers **exécutables** provenant d’utilisateurs.

### 11.4 Lire/écrire du JSON proprement

```js
import fsp from "node:fs/promises";

async function readJson(filePath) {
  const raw = await fsp.readFile(filePath, "utf8");
  return JSON.parse(raw);
}

async function writeJson(filePath, data) {
  await fsp.writeFile(filePath, JSON.stringify(data, null, 2), "utf8");
}
```

> Astuce : toujours gérer `JSON.parse` avec try/catch si l’entrée peut être invalide.

---

## 12) Cas pratiques (ateliers guidés)

### Atelier 1 — Mini utilitaire “cat” (lecture)

**Objectif** : reproduire partiellement `cat fichier.txt`.

1. Créez `cat.mjs`
2. Passez le chemin du fichier en argument CLI
3. Affichez le contenu

```js
// cat.mjs
import fsp from "node:fs/promises";

const file = process.argv[2];
if (!file) {
  console.error("Usage: node cat.mjs <file>");
  process.exit(1);
}

try {
  const content = await fsp.readFile(file, "utf8");
  process.stdout.write(content);
} catch (err) {
  console.error("Erreur:", err.message);
  process.exit(2);
}
```

### Atelier 2 — Logger simple (append)

**Objectif** : écrire une ligne horodatée dans un fichier de log.

```js
import fsp from "node:fs/promises";

async function logLine(message) {
  const line = `[${new Date().toISOString()}] ${message}\n`;
  await fsp.appendFile("./out/app.log", line, "utf8");
}

await logLine("Démarrage");
await logLine("Action utilisateur");
```

### Atelier 3 — Copier un dossier (niveau intermédiaire)

**Objectif** : recopier récursivement un dossier `srcDir` vers `destDir`.

```js
import fsp from "node:fs/promises";
import path from "node:path";

async function copyDir(srcDir, destDir) {
  await fsp.mkdir(destDir, { recursive: true });
  const entries = await fsp.readdir(srcDir, { withFileTypes: true });

  for (const e of entries) {
    const src = path.join(srcDir, e.name);
    const dest = path.join(destDir, e.name);

    if (e.isDirectory()) {
      await copyDir(src, dest);
    } else if (e.isFile()) {
      await fsp.copyFile(src, dest);
    }
  }
}

await copyDir("./data", "./out/data-copy");
```

### Atelier 4 — Traitement de gros fichier avec streams

**Objectif** : transformer un texte en uppercase sans charger tout en mémoire.

- Input : `./data/big.txt`
- Output : `./out/big.upper.txt`

Reprenez l’exemple `pipeline` + `Transform` (section Streams).

---

## 13) Checklist & bonnes pratiques

- Utiliser **`fs/promises`** + `async/await` pour la plupart des opérations.
- Utiliser **Streams** (`pipeline`) pour les gros fichiers.
- Toujours gérer les erreurs (`try/catch`) et inspecter `err.code`.
- Construire les chemins avec **`path`** (pas de concaténation naïve).
- Sécuriser les chemins utilisateurs (anti path traversal).
- Préférer les écritures **atomiques** pour les fichiers critiques.
- Éviter `existsSync` en code serveur chaud; préférer `access` + gestion d’erreur.

---

## Annexes — Mémo des fonctions courantes

| Besoin | Fonction |
|---|---|
| Lire un fichier | `fsp.readFile` |
| Écrire un fichier | `fsp.writeFile` |
| Ajouter à un fichier | `fsp.appendFile` |
| Créer un dossier | `fsp.mkdir({ recursive: true })` |
| Lister un dossier | `fsp.readdir({ withFileTypes: true })` |
| Infos fichier | `fsp.stat` / `fsp.lstat` |
| Renommer/déplacer | `fsp.rename` |
| Copier | `fsp.copyFile` |
| Supprimer fichier | `fsp.unlink` |
| Supprimer dossier | `fsp.rm({ recursive: true, force: true })` |
| Stream lecture | `fs.createReadStream` |
| Stream écriture | `fs.createWriteStream` |
| Pipeline robuste | `stream/promises.pipeline` |
| Surveiller changements | `fs.watch` |
