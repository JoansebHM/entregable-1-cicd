# entregable-1-cicd
Primer entregable del curso de CI/CD

JOAN SEBASTIAN HERRERA MONTES - JoansebHM

### ¿Qué eventos configuraste para disparar tu workflow y por qué?
Utilicé el evento **PUSH** y el de **WORKFLOW_DISPATCH**, el de **PUSH** para que se ejecute el trigger cada que alguien sube cambios al repo, especificamente a la rama main. Por otro lado el **WORKFLOW_DISPATCH** lo puse para poder correr el workflow manualmente sin necesidad de hacer push a la rama sino que directamente de la UI de github se le da en run workflow y se ejecuta lo que tenga configurado.

---


### ¿Qué sistema operativo utiliza el runner ubuntu-latest y por qué puede ejecutar tu workflow aunque tu computadora sea Windows o macOS? ¿Qué relación tiene el runner con el shell de los pasos run?

El OS que usa el runner es Linux y el workflow se puede ejecutar desde cualquier máquina porque internamente github actions levanta contenedores o instancias en sus propios servidores que montan la imagen del OS que le hayamos definido para las pruebas, asi que en realidad nada corre en nuestra maquina.

---

### Menciona al menos 3 acciones pre-construidas del Marketplace de GitHub Actions no mencionadas en este material. Excluye actions/checkout, actions/setup-python, actions/upload-artifact y aws-actions/configure-aws-credentials, aunque alguna solo aparezca como ejemplo conceptual. Describe qué hacen y para qué sirven.

#### **the-pr-agent/pr-agent@main**
Agente de inteligencia artificial usado para resumenes, code reviews y sugerencias de mejora de código.

```yaml
# .github/workflows/pr-agent.yml
name: PR Agent
on:
  pull_request:
    types: [opened, synchronize]
jobs:
  pr_agent_job:
    runs-on: ubuntu-latest
    steps:
    - name: PR Agent action step
      uses: the-pr-agent/pr-agent@main
      env:
        OPENAI_KEY: ${{ secrets.OPENAI_KEY }}
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### **docker**
Acciones de docker para construir y pushear imagenes de docker al subir cambios al repositorio.

```yaml
name: ci

on:
  push:

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      -
        name: Login to Docker Hub
        uses: docker/login-action@v4
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v4
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4
      -
        name: Build and push
        uses: docker/build-push-action@v7
        with:
          push: true
          tags: user/app:latest
```

#### **Elastic Email Send Email Action**
Acción de envío de correos cuando un build falla, se testea, esta en despliegue o se despliega de manera exitosa o se cancela.

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run tests
        run: npm test

      - name: Email the team if the build fails
        if: failure()
        uses: ElasticEmail/elasticemail-send-email-action@v1
        with:
          api_key: ${{ secrets.ELASTIC_EMAIL_API_KEY }}
          from_email: ci@yourdomain.com
          to: dev-team@yourdomain.com
          subject: "❌ Build failed: ${{ github.repository }}"
          status: failure
```


---

### ¿Qué es un secreto en GitHub Actions y para qué se usa? ¿Cómo lo usarías en tu workflow?

Un secreto es un dato sensible que no puede ser expuesta pero es necesaria para que los jobs del workflow puedan correr. Por ejemplo en el caso de que se necesite hacer un despliegue en una instancia de aws, el cliente, que en este caso son los runners de github actions en el que estamos corriendo el job, necesita las credenciales de AWS para poder hacer el despliegue, entonces en esos casos el workflow accede mediante la expresión de ${{ secrets.SECRETO }}

```yaml
name: Deploy to EC2

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout del código
        uses: actions/checkout@v4

      - name: Configurar la llave SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/ec2.pem
          chmod 600 ~/.ssh/ec2.pem
          ssh-keyscan -H "${{ secrets.EC2_HOST }}" >> ~/.ssh/known_hosts

      - name: Copiar archivos a la instancia
        run: |
          rsync -avz --delete \
            --exclude '.git' --exclude 'node_modules' \
            -e "ssh -i ~/.ssh/ec2.pem" \
            ./ ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }}:/home/${{ secrets.EC2_USER }}/app

      - name: Desplegar en la instancia
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          ssh -i ~/.ssh/ec2.pem ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} \
            "DB_PASSWORD='$DB_PASSWORD' bash -s" << 'EOF'
            set -e
            cd ~/app
            echo "DB_PASSWORD=$DB_PASSWORD" > .env
            chmod 600 .env
            npm ci --omit=dev
            pm2 restart app || pm2 start server.js --name app
            echo "Deploy terminado: $(date)"
          EOF

      - name: Limpiar la llave
        if: always()
        run: rm -f ~/.ssh/ec2.pem
```

En mi caso me parece importante por ejemplo porque en mi trabajo tenemos un servidor on premise y como es de una universidad entonces tenemos datos sensibles que manejamos dentro de .envs y sería buena idea para no estarle pidiendo a alguien del equipo que porfavor despliegue el (porque es el unico con esas claves), simplemente subir esos datos al github secrets y automatizar los deploys con un workflow que haga la tarea manual de él.

---

### Reflexiona sobre el uso de GitHub Actions en tu flujo de trabajo diario. ¿Qué tareas automatizarías? ¿Por qué?

En mi caso los despliegues al servidor on premise para no estarle pidiendo al encargado que lo haga. También que una vez finalizada esta tarea del deploy se le asignen milestones a los pr que se mergearon para saber si ya estan desplegados o no para saber que cambios deberian estar en desarrollo y no estar preguntando si ya subio tal cambio.
