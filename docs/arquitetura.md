
# Diagrama de Arquitetura — Achados & Perdidos

```mermaid
flowchart TB
    cliente["Navegador / curl"]
    administrador["Administrador"]

    subgraph host["HOST — Computador com Docker"]
        porta["Porta publicada: 8080"]
        portaAdminer["Porta local: 127.0.0.1:8081"]

        subgraph rede["rede_interna — bridge"]
            nginx["nginx<br/>Proxy reverso<br/>Porta interna 80"]

            web["web<br/>Flask + Gunicorn<br/>Porta interna 5000<br/>Usuário appuser"]

            db["db<br/>PostgreSQL 16 Alpine<br/>Porta interna 5432<br/>Sem porta publicada"]

            adminer["adminer<br/>Administração do banco<br/>Profile debug — opcional"]
        end

        subgraph armazenamento["Armazenamento persistente"]
            fotos[("Volume: fotos<br/>/app/uploads")]
            dados[("Volume: dados_banco<br/>/var/lib/postgresql/data")]
        end

        sql["Arquivo local<br/>db/init.sql"]
        backup["Backup das fotos<br/>fotos_backup.tar.gz"]
    end

    cliente -->|"HTTP localhost:8080"| porta
    porta -->|"8080:80"| nginx
    nginx -->|"HTTP web:5000<br/>DNS interno Docker"| web

    web -->|"SQL db:5432<br/>DNS interno Docker"| db

    web -->|"Volume nomeado"| fotos
    db -->|"Volume nomeado"| dados

    sql -->|"Bind mount somente leitura<br/>/docker-entrypoint-initdb.d/init.sql"| db

    fotos -.->|"Backup e restauração<br/>via contêiner temporário"| backup

    administrador -.->|"HTTP localhost:8081<br/>Somente com profile debug"| portaAdminer
    portaAdminer -.->|"8081:8080"| adminer
    adminer -.->|"Conexão SQL db:5432"| db
```

## Descrição

A aplicação Achados & Perdidos utiliza Docker Compose para
gerenciar os serviços nginx, web e db. Existe também o serviço
opcional adminer, ativado somente com o profile debug.

Todos os serviços estão conectados à rede bridge
personalizada rede_interna.

### Nginx — Proxy reverso

O Nginx é o ponto de entrada da aplicação.

Ele publica a porta 8080 do computador e encaminha
as requisições HTTP para o serviço web na porta 5000.

O serviço web não publica uma porta diretamente no host.

### Web — Flask e Gunicorn

O serviço web executa a aplicação Flask utilizando
o servidor Gunicorn.

A aplicação é executada com o usuário não privilegiado
appuser, definido no Dockerfile.

O serviço utiliza o nome db para localizar o PostgreSQL
por meio do DNS interno do Docker.

### Banco de dados — PostgreSQL

O serviço db utiliza a imagem postgres:16-alpine.

A porta 5432 é utilizada internamente, sem publicação
direta no computador.

O healthcheck verifica se o PostgreSQL está pronto para
receber conexões. O serviço web depende dessa condição
para ser iniciado pelo Docker Compose.

### Persistência dos dados

Dois volumes nomeados são utilizados:

- fotos: armazena as imagens enviadas pelos usuários
  no diretório /app/uploads.
- dados_banco: armazena os arquivos do PostgreSQL
  no diretório /var/lib/postgresql/data.

Esses volumes permitem preservar os dados mesmo após
a remoção e recriação dos contêineres, desde que os
volumes não sejam excluídos.

### Inicialização do banco

O arquivo db/init.sql é disponibilizado ao contêiner
do PostgreSQL por um bind mount somente leitura.

O script é executado automaticamente na primeira
inicialização de um diretório de dados vazio.

### Backup e restauração

Foi realizado um backup do volume fotos utilizando
um contêiner temporário com Alpine Linux e o comando tar.

O arquivo gerado foi salvo em:

docs/evidencias/fotos_backup.tar.gz

A restauração foi testada em um volume separado.
Os arquivos restaurados foram comparados com os
originais para verificar sua integridade.

### Adminer — Profile debug

O serviço adminer permite consultar e administrar
o banco PostgreSQL por uma interface web.

Ele utiliza o profile debug e não é iniciado durante
a execução normal do projeto.

Quando ativado, o painel fica disponível em:

http://localhost:8081

A publicação da porta é restrita ao endereço 127.0.0.1
do host, evitando sua exposição direta em interfaces
externas.

O Adminer se conecta ao PostgreSQL utilizando o
endereço db na rede Docker.

### Credenciais

As credenciais de acesso ao PostgreSQL são configuradas
por variáveis de ambiente.

O arquivo .env contém os valores locais e não deve
ser incluído no repositório nem no ZIP de entrega.

O arquivo .env.example contém apenas valores de exemplo.

## Serviços por modo de execução

Execução normal:

docker compose up -d --build

Serviços iniciados:
- db
- web
- nginx

Execução com ferramentas de administração:

docker compose --profile debug up -d

Serviços iniciados:
- db
- web
- nginx
- adminer
```
