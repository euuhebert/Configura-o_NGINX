
# Guia Completo para fazer deploy de um servidor Streamlit no Linux

## 1. Instalação do Python 3.9 e Criação do Ambiente Virtual

### Para Debian/Ubuntu:
1. **Atualizar repositórios:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. **Instalar dependências para o ambiente Python:**
   ```bash
   sudo apt install -y build-essential libssl-dev libffi-dev zlib1g-dev libbz2-dev \
   libreadline-dev libsqlite3-dev wget curl llvm libncurses-dev xz-utils tk-dev \
   libxml2-dev libxmlsec1-dev liblzma-dev
   ```

3. **Instalar Python 3.9 e venv (se necessário):**
   ```bash
   sudo apt install python3.9 python3.9-venv
   ```

4. **Criar o ambiente virtual:**
   ```bash
   python3.9 -m venv iazin
   ```

---

### Para Oracle Linux/RHEL:
1. **Instalar dependências para o Python:**
   ```bash
   sudo yum install -y gcc libffi-devel zlib-devel bzip2 bzip2-devel readline-devel \
   sqlite sqlite-devel openssl-devel tk-devel libgdbm-devel xz-devel libuuid-devel
   ```

2. **Instalar Python 3.9 e venv (se necessário):**
   ```bash
   sudo yum install python39 python39-venv
   ```

3. **Criar o ambiente virtual:**
   ```bash
   python3.9 -m venv nome_da_venv
   ```

---

## 2. Instalação do Streamlit no Ambiente Virtual

1. **Ativar o ambiente virtual:**
   ```bash
   source nome_da_venv/bin/activate
   ```

2. **Atualizar o pip:**
   ```bash
   pip install --upgrade pip
   ```

3. **Instalar o Streamlit:**
   ```bash
   pip install streamlit
   ```

---

## 3. Criação do Script Simples do Streamlit

1. **Criar o arquivo `app.py` com o conteúdo abaixo:**
   ```python
   import streamlit as st

   st.title('Hello, World!')
   st.write("Este é um exemplo simples de Streamlit!")
   if st.button('Clique aqui'):
       st.write('Você clicou no botão!')
   ```

2. **Testar o Streamlit:**
   ```bash
   streamlit run app.py
   ```

---

## 4. Criação de um Script Bash para Executar o Streamlit

1. **Criar o script `start_streamlit.sh` com o conteúdo:**
   ```bash
   #!/bin/bash
   source /pasta_onde_esta_venv/nome_do_venv/bin/activate
   cd /diretorio/onde/esta/script
   streamlit run app.py
   ```

2. **Tornar o script executável:**
   ```bash
   chmod +x /diretorio/onde/esta/script/start_streamlit.sh
   ```
3. **Caso Esteja usando RHEL/ORACLE LINUX é necessário definir o contexto do arquivo de script:**
   ```bash
   chcon -t bin_t /seu_diretorio/start_streamlit.sh
   sls -z start_streamlit.sh
   ```      

---

## 5. Criação de um Serviço Systemd

1. **Criar o arquivo de serviço `/etc/systemd/system/streamlitsite.service` com o conteúdo:**
   ```ini
   [Unit]
   Description=Streamlit site

   [Service]
   ExecStart=/diretorio/onde/esta/script/script_streamlit.sh
   WorkingDirectory=/diretorio/onde/esta/o_bin/do_venv #exemplo /home/user/nome_venv/bin
   Restart=always
   User=root
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```


2. **Recarregar o systemd para aplicar as mudanças:**
   ```bash
   sudo systemctl daemon-reload
   ```

3. **Habilitar o serviço para iniciar automaticamente no boot:**
   ```bash
   sudo systemctl enable streamlitsite.service
   ```

4. **Iniciar o serviço:**
   ```bash
   sudo systemctl start streamlitsite.service
   ```

5. **Verificar o status do serviço:**
   ```bash
   sudo systemctl status streamlitsite.service
   ```

6. **Verificar os logs (se necessário):**
   ```bash
   journalctl -u streamlitsite.service
   ```

---

## 6. Redirecionar o Fluxo pela Porta 80

### Para Debian/Ubuntu:

1. **Instalar o NGINX:**
   ```bash
   sudo apt-get install nginx
   ```

2. **Configurar o arquivo do site em NGINX:**
   ```bash
   cd /etc/nginx
   cd /sites-available
   nano streamlitsite
   ```

3. **Exemplo de configuração do servidor:**
   ```nginx
   server {
       listen 80;
       server_name seu_ip_aqui;  # Coloque o IP do seu servidor ou domínio

       location / {
           proxy_pass http://127.0.0.1:8501/;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection "upgrade";
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

4. **Criar o link simbólico para habilitar o site:**
   ```bash
   ln -s /etc/nginx/sites-available/streamlitsite /etc/nginx/sites-enabled/
   ```

5. **Remover o arquivo de configuração padrão:**
   ```bash
   cd /etc/nginx/sites-enabled/
   rm default
   ```

6. **Testar a configuração do NGINX:**
   ```bash
   nginx -t
   ```

7. **Testar no navegador com o IP do servidor:**
   Abra o navegador e digite o IP do servidor, como `http://seu_ip`.


---

### Para Oracle Linux/RHEL:

1. **Instalar o NGINX:**
   ```bash
   sudo yum install -y oracle-epel-release-el7
   sudo yum install -y nginx
   ```

2. **Entrar no diretório do NGINX:**
   ```bash
   cd /etc/nginx
   ```

3. **Abrir o arquivo de configuração do NGINX:**
   ```bash
   vi /etc/nginx/nginx.conf
   ```

4. **Exemplo de configuração de proxy reverso para o Streamlit:**
   ```nginx
   user nginx;
   worker_processes auto;
   error_log /var/log/nginx/error.log;
   pid /run/nginx.pid;

   events {
       worker_connections 1024;
   }

   http {
       include       /etc/nginx/mime.types;
       default_type  application/octet-stream;

       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                           '$status $body_bytes_sent "$http_referer" '
                           '"$http_user_agent" "$http_x_forwarded_for"';

       access_log  /var/log/nginx/access.log  main;

       sendfile        on;
       tcp_nopush      on;
       tcp_nodelay     on;
       keepalive_timeout  65;
       types_hash_max_size 2048;

       server {
           listen 80;
           server_name seu_ip_aqui;  # Substitua pelo IP ou domínio do seu servidor

           location / {
               proxy_pass http://127.0.0.1:8501/;
               proxy_http_version 1.1;
               proxy_set_header Upgrade $http_upgrade;
               proxy_set_header Connection "upgrade";
               proxy_set_header Host $host;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header X-Forwarded-Proto $scheme;
               proxy_cache_bypass $http_upgrade;
           }

           error_page 404 /404.html;
           location = /40x.html {}

           error_page 500 502 503 504 /50x.html;
           location = /50x.html {}
       }
   }
   ```

5. **Liberar o serviço do NGINX para se comunicar com o Streamlit:**
   ```bash
   setsebool -P httpd_can_network_connect on
   ```

6. **Testar a configuração do NGINX:**
   ```bash
   nginx -t
   ```

7. **Testar no navegador com o IP do servidor:**
   Abra o navegador e digite o IP do servidor, como `http://seu_ip_aqui`.

---
