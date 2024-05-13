### Anleitung für die Erstellung der Pipeline
1. Erstelle einen .github Ordner im Root-Verzeichnis des Projekts. Dann darin einen workflows Ordner und darin eine Datei mit dem Namen main.yml.
2. Frontend bauen
Wir wollen uns zunächst das Frontend bauen. Da dies ein Angular-Projekt ist, bauen wir das Frontend mit dem Befehl `ng run build`. Dieser Befehl erstellt ein dist-Verzeichnis, das wir später verwenden werden. Das dist-Verzeichnis wollen wir daher als Artifakt speichern bzw. uploaden, da wir es später für das Deployment benötigen.
```
jobs:
  build-frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Install dependencies
        run: |
          cd frontend
          npm install

      - name: Build frontend
        run: |
          cd frontend
          npm run build

      - name: List output files
        run: |
          ls -la frontend/dist

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: frontend-dist
          path: frontend/dist
```
3. 