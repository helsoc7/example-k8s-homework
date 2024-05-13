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
Wenn wir das Ganze dann mal in den Actions anschauen, sehen wir schon, dass dort ein frontend-dist-Artifakt erstellt wurde. Das ist schon mal ein guter Anfang.
3. Frontend deployen
Als nächstes wollen wir das Frontend deployen. Dafür deployen wir das Frontend auf ein S3 Bucket. Dafür müssen wir die AWS Credentials in den Secrets hinterlegen. Dafür gehen wir in die Settings des Repositories und klicken auf Secrets. Dort fügen wir dann die AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY und die AWS_SESSION_TOKEN hinzu.
```
  deploy-frontend:
    runs-on: ubuntu-latest
    needs: build-frontend
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v3
        with:
          name: frontend-dist
          path: frontend/dist

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-session-token: ${{ secrets.AWS_SESSION_TOKEN }}
          aws-region: eu-central-1

      - name: Deploy to S3
        run: |
          cd frontend/dist/frontend/browser
          aws s3 cp . s3://${{ secrets.AWS_S3_BUCKET }}/ --recursive
```