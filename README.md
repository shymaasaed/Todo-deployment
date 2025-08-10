# 🚀 CI/CD Enabled Todo List App using Node.js, Docker, GitHub Actions, Ansible & Watchtower

## 📌 Overview

This project is a simple Todo List web application built with Node.js and Express, styled using EJS templates. What makes it unique is the full CI/CD pipeline configured to automatically build, ship, and deploy Docker containers on a remote EC2 instance with zero downtime using Watchtower.

## 🧱 Stack

- **Database**: MongoDB
- **CI/CD**: GitHub Actions + Docker + Ansible + Watchtower
- **Infrastructure**: AWS EC2 + Docker Compose

---

## 🔄 CI/CD Pipeline Description

1. **GitHub Actions** automatically triggers on every `push` to the repository.
2. It builds a new Docker image of the Node.js Todo App.
3. The image is tagged and pushed to **Docker Hub Registry**:
```

shymaasaeed404/todo-list-nodejs\:latest

````
4. An **Ansible Playbook** is used to:
- Connect to a remote EC2 instance.
- Transfer the `docker-compose.yml` file.
- Start the container via `docker-compose up`.
5. **Watchtower** runs as a container in the EC2 instance. It checks for image updates in Docker Hub every 5 minutes and automatically replaces the running container if a new image is available.

---

## ⚙️ Test Todo Locally

1. Clone the repository:
```bash
git clone https://github.com/shymaasaed/Todo-List-nodejs.git
cd Todo-List-nodejs
````

2. Install dependencies:

   ```bash
   npm install
   ```

3. Add a `.env` file:

   ```env
   MONGO_URI=your_mongodb_uri
   PORT=8000
   ```

4. Run the app:

   ```bash
   npm start
   ```

Then open `http://localhost:8000` to view the app.

---

## 📦 Docker Build & Run (Local Testing)

Build the image locally:

```bash
docker build -t "container-name" .
```

Run the container:

```bash
docker run -d -p 8000:8000 "container name"
```

---

## 🤖 GitHub Actions CI Configuration

<details>
<summary>🔽 Click to expand: GitHub Actions workflow file (.github/workflows/master.yml)</summary>

```yaml
name: CI Docker Build and Push

on:
  push:
    branches:
      - master

env:
  REGISTRY: docker.io
  IMAGE_NAME: "docker account user-name / repo-name : tag"

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v3
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            latest
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

</details>

---

## 🐳 Docker File

<details>
<summary>🔽 Click to expand: docker-compose.yml</summary>

```yaml
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8000
CMD ["npm", "start"]

```

</details>

---

## 📡 Ansible Playbook Configuration
configure-server.yml
An Ansible playbook that automates the setup of a remote server, including Docker installation and the deployment of the application using Docker Compose. It ensures a consistent environment for the application.

<details>
<summary>🔽 Click to expand: ansible-playbook.yml</summary>

```yaml
---


  hosts: {machine ip}
  become: true
  vars:
    app_dir: /home/ec2-user/app
  tasks:
    - name: Update all packages (for Amazon Linux/CentOS)
      yum:
    name: '*'
        state: latest
      when: ansible_distribution in ["Amazon", "CentOS"]
    - name: Install Docker
      package:
    name: docker
        state: present
    - name: Start and enable Docker service
      service:
    name: docker
        state: started
        enabled: true
    - name: Add user to the docker group
      user:
    name: "{{ ansible_user }}"
        groups: docker
        append: true
    - name: Docker Login to Docker Hub
      community.docker.docker_login:
        username: "XXXXXXXXX"
        password: "XXXXXXXXX"
      no_log: true
    - name: Install Docker Compose V2 as Docker CLI plugin
      ansible.builtin.get_url:
        url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-linux-x86_64"
        dest: /usr/local/lib/docker/cli-plugins/docker-compose
        mode: '0755'
        force: true
      vars:
    docker_compose_version: "v2.27.0"
    - name: Create application directory
      file:
    path: "{{ app_dir }}"
        state: directory
        owner: "{{ ansible_user }}"
        group: "{{ ansible_user }}"
        mode: '0755'
    - name: Copy docker-compose.yml to remote server
      copy:
    src: ./docker-compose.yml
        dest: "{{ app_dir }}/docker-compose.yml"
    - name: Create .env file with MONGO_URI secret
      copy:
    content: "MONGO_URI={{ lookup('env', 'MONGO_URI') }}"
        dest: "{{ app_dir }}/.env"
    - name: Run docker-compose up (using docker_compose_v2 module)
      community.docker.docker_compose_v2:
        project_src: "{{ app_dir }}"
        state: present

```
</details>

----
## 📡 Docker-compose file
docker-compose.yml
This file defines the multi-container Docker application, orchestrating the todo-app and watchtower services. It sets up how these services run, communicate, and are exposed.

<details>
<summary>🔽 Click to expand: ansible-playbook.yml</summary>

```yaml
---
services:
  todo-app:
    image: "docker account user-name / repo-name : tag"
    container_name: todo-app
    ports:
      - "8000:8000"
    environment:
      - MONGO_URI={your-DB URL}
      - NODE_ENV=production
    restart: always
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000"]
      interval: 30s
      timeout: 10s
      retries: 5

  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ${HOME}/.docker/config.json:/config.json:ro
    restart: always
    environment:
      - WATCHTOWER_SCHEDULE=0 */5 * * * *
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_INCLUDE_STOPPED=true
      - WATCHTOWER_DEBUG=true
    command:
      - todo-app
```
</details>
---

## 🔁 Watchtower Overview

Watchtower is included in the `docker-compose.yml` file as a service. It is configured to poll Docker Hub every 5 minutes for changes in the image `shymaasaeed404/todo-list-nodejs`. If an update is found, it gracefully stops the running container and starts the updated one — ensuring zero-downtime deployment.

No additional script is used. The container handles everything internally.

## .gitignore
This file specifies intentionally untracked files that Git should ignore, such as local development dependencies (node_modules/) and sensitive environment variables (.env). It keeps your repository clean and secure.

## .env
This file stores environment-specific variables, like the MONGO_URI, that are crucial for the application's configuration. It keeps sensitive data out of version control and allows for easy environment switching.
# ⚠️ Important: Database URL Name Must Match
 The name in .env must be the same as in config/mongoose.js.
Example:
# .env
MONGO_URI=mongodb://localhost:27017/todo
# config/mongoose.js
mongoose.connect(process.env.MONGO_URI);

    If you change the name in .env, change it in the code too.
    If they don’t match, MongoDB won’t connect.
---

## 🚀 Deployment Summary:
## The CI/CD pipeline has been fully tested, with successful deployment to EC2. The Docker image builds and runs without issues, and Watchtower is actively handling automatic updates. 

---
