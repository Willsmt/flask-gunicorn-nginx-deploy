# firstproject — Flask + Gunicorn + Nginx + Systemd

Stack de deploy de aplicação Python em ambiente Linux (Ubuntu 26.04), cobrindo desde o ambiente virtual isolado até o serviço gerenciado pelo systemd com Nginx como reverse proxy.

---

## Stack

| Camada | Tecnologia | Função |
|---|---|---|
| Aplicação | Python / Flask | Lógica da aplicação web |
| Entry point WSGI | wsgi.py | Interface entre Gunicorn e Flask |
| Servidor WSGI | Gunicorn | Servidor de produção multi-worker |
| Gerenciador de processo | Systemd | Mantém a app rodando em background |
| Reverse proxy | Nginx | Único ponto de entrada HTTP |
| Isolamento | Python venv | Dependências isoladas por projeto |

---

## Arquitetura

```
Browser / Cliente HTTP
        |
        | porta 80
        ↓
    [ Nginx ]
        | proxy_pass via Unix socket
        ↓
[ firstproject.sock ]
        |
        ↓
   [ Gunicorn ]
    3 workers
        |
        ↓
  [ Flask App ]
```

O Gunicorn não é exposto diretamente na rede. A comunicação entre Nginx e Gunicorn acontece via **Unix socket** — um arquivo local invisível para a rede, eliminando a necessidade de abrir portas TCP adicionais.

---
## Documentação

Para um guia de estudo completo com a explicação detalhada de cada
camada, decisões de arquitetura, troubleshooting e glossário, consulte
a [apostila completa](docs/Apostila-Deploy-Flask-Gunicorn-Nginx.pdf).
---

## Estrutura do Projeto

```
firstproject/
├── firstproject.py         # Aplicação Flask
├── wsgi.py                 # Entry point WSGI para o Gunicorn
├── firstproject.sock       # Unix socket (gerado em runtime pelo Gunicorn)
├── firstprojectenv/        # Ambiente virtual Python (não versionar)
│   ├── bin/
│   │   ├── python3
│   │   ├── pip
│   │   └── gunicorn
│   └── lib/
│       └── python3.x/
│           └── site-packages/
└── README.md
```

---

## Pré-requisitos

- Ubuntu 22.04+ (testado no 26.04 LTS)
- Python 3.x
- Nginx instalado (`sudo apt install nginx`)
- `python3-venv` instalado (`sudo apt install python3-venv`)

---

## Setup

### 1. Clonar e entrar no projeto

```bash
git clone <repo-url> ~/projects/firstproject
cd ~/projects/firstproject
```

### 2. Criar e ativar o ambiente virtual

```bash
python3 -m venv firstprojectenv
source firstprojectenv/bin/activate
```

O prefixo `(firstprojectenv)` no terminal confirma que o venv está ativo.

### 3. Instalar dependências

```bash
pip install gunicorn flask
```

> Sempre instale o Gunicorn **dentro do venv**. O systemd aponta para o executável do venv — se o Gunicorn estiver instalado globalmente, o serviço não encontrará as dependências da aplicação.

### 4. Verificar a aplicação

```bash
python firstproject.py
# Acesse http://localhost:5000
```

Confirme que a aplicação responde, depois encerre com `Ctrl+C`.

### 5. Verificar o entry point WSGI

```bash
gunicorn --bind 0.0.0.0:8000 wsgi
# Acesse http://localhost:8000
```

Confirme que o Gunicorn serve a aplicação, depois encerre com `Ctrl+C`.

---

## Configuração do Systemd

Cria o arquivo de serviço em `/etc/systemd/system/firstproject.service`:

```ini
[Unit]
Description=Gunicorn instance to serve firstproject
After=network.target

[Service]
User=<seu-usuario>
Group=<seu-usuario>
WorkingDirectory=/home/<seu-usuario>/projects/firstproject
Environment="PATH=/home/<seu-usuario>/projects/firstproject/firstprojectenv/bin"
ExecStart=/home/<seu-usuario>/projects/firstproject/firstprojectenv/bin/gunicorn \
          --workers 3 \
          --bind unix:firstproject.sock \
          -m 007 \
          wsgi

[Install]
WantedBy=multi-user.target
```

> Substitua `<seu-usuario>` pelo seu usuário Linux.

**Parâmetros relevantes:**

| Parâmetro | Valor | Motivo |
|---|---|---|
| `--workers 3` | 3 processos | Atende requisições em paralelo |
| `--bind unix:...sock` | Unix socket | Comunicação local segura com Nginx |
| `-m 007` | Máscara de permissão | Somente dono e grupo acessam o socket |

### Ativar e iniciar o serviço

```bash
sudo systemctl daemon-reload
sudo systemctl start firstproject
sudo systemctl enable firstproject
```

### Verificar status

```bash
sudo systemctl status firstproject
```

Confirme que o socket foi criado:

```bash
ls -la ~/projects/firstproject/firstproject.sock
# srwxrwx--- 1 <usuario> <usuario> ...
```

O `s` no início indica que é um socket file, não um arquivo comum.

---

## Configuração do Nginx

Cria o arquivo em `/etc/nginx/sites-available/firstproject`:

```nginx
server {
    listen 80;
    server_name firstproject www.firstproject;

    access_log /var/log/nginx/firstproject.access.log;
    error_log /var/log/nginx/firstproject.error.log;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/<seu-usuario>/projects/firstproject/firstproject.sock;
    }
}
```

### Ativar o site

```bash
sudo ln -s /etc/nginx/sites-available/firstproject /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Permissão do socket para o Nginx

O Nginx roda como `www-data`. Para acessar o socket é necessário adicioná-lo ao grupo do seu usuário:

```bash
sudo usermod -aG <seu-usuario> www-data
sudo systemctl restart firstproject
sudo systemctl restart nginx
```

---

## DNS Local (desenvolvimento)

Para resolver o domínio `firstproject` localmente, adicione ao `/etc/hosts` da máquina cliente:

```
192.168.x.x   firstproject www.firstproject
```

> Em produção com domínio real, esse passo não é necessário — o DNS global resolve o domínio para o IP do servidor. Apenas o `server_name` no Nginx precisa ser atualizado.

---

## Firewall (UFW)

Apenas a porta 80 precisa estar aberta. Não exponha as portas do Gunicorn diretamente:

```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw status
```

---

## Por que Unix socket e não porta TCP?

Com porta TCP o Gunicorn ficaria acessível diretamente na rede, permitindo que um atacante bypasse o Nginx e todas as suas proteções (rate limiting, headers de segurança, logs). Com Unix socket:

- Não aparece em scans de porta (`nmap`)
- Só processos locais com permissão no arquivo conseguem se comunicar
- Menor superfície de ataque

---

## Comandos úteis

```bash
# Ver logs da aplicação
sudo journalctl -u firstproject -f

# Reiniciar após mudanças no código
sudo systemctl restart firstproject

# Ver logs de acesso do Nginx
sudo tail -f /var/log/nginx/firstproject.access.log

# Ver logs de erro do Nginx
sudo tail -f /var/log/nginx/firstproject.error.log

# Sair do ambiente virtual
deactivate
```

---

## Variáveis de Ambiente

Nunca exponha credenciais no código. Para projetos com banco de dados, chaves de API etc., use um arquivo `.env`:

```bash
# .env (nunca versionar este arquivo)
SECRET_KEY=sua-chave-secreta
DATABASE_URL=postgresql://user:password@localhost/dbname
```

Adicione ao `.gitignore`:

```
firstprojectenv/
.env
*.sock
__pycache__/
*.pyc
```
