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
Als nächstes wollen wir das Frontend deployen. Dafür deployen wir das Frontend auf ein S3 Bucket. Dafür müssen wir die AWS Credentials in den Secrets hinterlegen. Dafür gehen wir in die Settings des Repositories und klicken auf Secrets. Dort fügen wir dann die AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY und die AWS_SESSION_TOKEN hinzu. Außerdem muss noch der Bucket-Name angegeben werden. Achtet hier bitte drauf, dass ihr den richtigen Namen des Artifakts verwendet. In unserem Fall ist das frontend-dist.
```
  deploy-frontend:
    runs-on: ubuntu-latest
    needs: build-frontend
    steps:
      - name: Download artifact
        uses: actions/download-artifact@v4
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
4. Backend bauen
Als nächstes wollen wir das Backend bauen. Dazu verwenden wir ein Dockerfile, um später das Backend auf der EC2-Instanz zu starten. Das Dockerfile liegt bereits im Backend-Ordner. Unser Workflow sieht wie folgt aus:
```
  build_backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Log in to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_TOKEN }}
      - name: Build and push
        run: |
          docker build -t ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }} ./backend
          docker push ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
```
Achtung: Hier müssen wir natürlich den DOCKER_USERNAME, DOCKER_TOKEN als Secrets setzen und das IMAGE_NAME und IMAGE_TAG als Environment Variables setzen.
5. Backend deployen
Als Letztes wollen wir das Backend deployen. Dafür verwenden wir die EC2-Instanz, die wir bereits erstellt haben. Dafür müssen wir die SSH-Keys in den Secrets hinterlegen. Dafür gehen wir in die Settings des Repositories und klicken auf Secrets. Dort fügen wir dann die SSH_PRIVATE_KEY.
```
  deploy-backend:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Ansible environment
        uses: dawidd6/action-ansible-playbook@v2
        with:
          key: ~/.ssh/id_rsa
          playbook: deploy_backend.yml
          inventory: hosts.ini
```

Achtung: Wir müssen den SSH-Key erst speichern in dem Container des Jobs und die EC2-Instanz zu den known_hosts hinzufügen. Das machen wir mit folgendem Befehl:
```
      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keygen -y -f ~/.ssh/id_rsa

      - name: Start SSH agent
        run: |
          eval "$(ssh-agent -s)"
          ssh-add ~/.ssh/id_rsa

      - name: Add known_hosts
        run: |
          ssh-keyscan -H 54.93.240.125 >> ~/.ssh/known_hosts
```

Wir müssen dazu natürlich noch das deploy_backend.yml und das hosts.ini erstellen. Das deploy_backend.yml sieht wie folgt aus:
```
- hosts: all
  become: yes
  tasks:
    - name: Install Docker
      yum:
        name: docker
        state: present

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes

    - name: Run Docker container
      shell: >
        docker run -d -p 3000:3000 helenhaveloh/k8s-application
      args:
        executable: /bin/bash
```
Das hosts.ini sieht wie folgt aus:
```
[backend-servers]
<public-ip der EC2> ansible_user=ec2-user 
```
###### ALTERNATIV: Mit ssh Connection auf EC2-Instanz und dann mit Docker starten
```
  deploy-backend:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PRIVATE_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keygen -y -f ~/.ssh/id_rsa

      - name: Start SSH agent
        run: |
          eval "$(ssh-agent -s)"
          ssh-add ~/.ssh/id_rsa

      - name: Add known_hosts
        run: |
          ssh-keyscan -H 54.93.240.125 >> ~/.ssh/known_hosts

      

      - name: Deploy Backend with Docker
        run: |
          ssh -o StrictHostKeyChecking=no ec2-user@54.93.240.125 "sudo docker run -dp 3000:3000 ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}"
```
ACHTUNG: Hier müssen wir auf der EC2-Instanz aber docker installieren und den Docker-Daemon starten. Das machen wir mit folgenden Befehlen:
```
sudo yum install docker -y
sudo service docker start
```
Außerdem muss der ec2-user dann noch die Berechtigungen haben, docker zu verwenden. 
```
sudo usermod -a -G docker ec2-user
```
Dann funktioniert es. 