# 'minikube assignment.txt

----------------------------------------------------------------------------------
Install Minikube
-----------------------------------------------------------------------------------
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
 chmod +x minikube
 sudo mv minikube /usr/local/bin/
 Install Helm
 Nginx Ingress Setup with Helm
 curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
 
---------------------------------------------------------
Add the Nginx Ingress Controller Helm Repository:
-----------------------------------------------------------------------------------
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

---------------------------------------------------------
Install Nginx Ingress Controller:
---------------------------------------------------------
helm install my-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true


---------------------------------------------------------
Install Docker
---------------------------------------------------------
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
 sudo usermod -aG docker $(whoami)
newgrp docker
docker --version

---------------------------------------------------------
Sample Application Deployment
---------------------------------------------------------

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: nginxdemos/hello
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: hello-world

---------------------------------------------------------
Create an Ingress Resource:
---------------------------------------------------------
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
spec:
  rules:
  - host: cdacmumbai-gitlab2.cdacmumbai.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80

---------------------------------------------------------
Ansible Automation
---------------------------------------------------------
---
- hosts: localhost
  become: yes
  vars:
    nginx_ingress_chart: "ingress-nginx/ingress-nginx"
    release_name: "my-ingress"
    app_name: "hello-world"
    app_image: "nginxdemos/hello"
    app_port: 80
    ingress_host: "cdacmumbai-gitlab2.cdacmumbai.in"

  tasks:
    - name: Ensure Helm is installed
      command: curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
      args:
        creates: /usr/local/bin/helm

    - name: Add the Ingress-Nginx Helm repository
      command: helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
      args:
        creates: ~/.cache/helm/repository/ingress-nginx-index.yaml

    - name: Update the Helm repositories
      command: helm repo update

    - name: Install Nginx Ingress Controller
      command: helm install {{ release_name }} {{ nginx_ingress_chart }} --set controller.publishService.enabled=true
      register: helm_install_result
      ignore_errors: true

    - name: Upgrade Nginx Ingress Controller if already installed
      command: helm upgrade {{ release_name }} {{ nginx_ingress_chart }} --set controller.publishService.enabled=true
      when: helm_install_result.rc != 0

    - name: Create Hello World Deployment YAML
      copy:
        content: |
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: {{ app_name }}
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: {{ app_name }}
            template:
              metadata:
                labels:
                  app: {{ app_name }}
              spec:
                containers:
                - name: {{ app_name }}
                  image: {{ app_image }}
                  ports:
                  - containerPort: {{ app_port }}
          ---
          apiVersion: v1
          kind: Service
          metadata:
            name: {{ app_name }}
          spec:
            ports:
            - port: {{ app_port }}
              targetPort: {{ app_port }}
            selector:
              app: {{ app_name }}
        dest: /tmp/{{ app_name }}-deployment.yaml

    - name: Apply Hello World Deployment to Kubernetes
      kubernetes.core.k8s:
        state: present
        definition: "{{ lookup('file', '/tmp/{{ app_name }}-deployment.yaml') }}"

    - name: Create Hello World Ingress YAML
      copy:
        content: |
          apiVersion: networking.k8s.io/v1
          kind: Ingress
          metadata:
            name: {{ app_name }}-ingress
          spec:
            rules:
            - host: {{ ingress_host }}
              http:
                paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service:
                      name: {{ app_name }}
                      port:
                        number: {{ app_port }}
        dest: /tmp/{{ app_name }}-ingress.yaml

    - name: Apply Hello World Ingress to Kubernetes
      kubernetes.core.k8s:
        state: present
        definition: "{{ lookup('file', '/tmp/{{ app_name }}-ingress.yaml') }}"


---------------------------------------------------------
Security with TLS/SSL
---------------------------------------------------------
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=cdacmumbai-gitlab2.cdacmumbai.in"
kubectl create secret tls hello-world-secret --key tls.key --cert tls.crt


Update Ingress Resource with TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
spec:
  tls:
  - hosts:
    - cdacmumbai-gitlab2.cdacmumbai.in
    secretName: hello-world-secret
  rules:
  - host: cdacmumbai-gitlab2.cdacmumbai.in
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80


