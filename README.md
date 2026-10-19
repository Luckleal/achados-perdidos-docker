
# Achados & Perdidos

Projeto desenvolvido para a disciplina de Computação em Nuvem.

O objetivo é executar uma aplicação de achados e perdidos utilizando Docker e Docker Compose, com banco PostgreSQL e armazenamento persistente para os dados e as fotos dos objetos cadastrados.

## 1. Tecnologias utilizadas

- Python 3.12
- Flask 3.0.3
- Gunicorn 23.0.0
- PostgreSQL 16 Alpine
- Nginx Alpine
- Adminer (opcional)
- Docker
- Docker Compose

## 2. Arquitetura

O ambiente possui três serviços principais:

- nginx: recebe as requisições HTTP e funciona como proxy reverso.
- web: executa a aplicação Flask utilizando Gunicorn.
- db: executa o PostgreSQL e armazena os registros.

Existe também o serviço adminer, utilizado para administrar o
PostgreSQL e ativado somente com o profile debug.

Os serviços utilizam a rede Docker rede_interna, com driver bridge.

O Nginx publica a porta 8080 do computador, encaminhando as
requisições para o serviço web na porta interna 5000.

O PostgreSQL utiliza a porta 5432 internamente, sem publicação
direta no host.

Foram configurados dois volumes nomeados:

- dados_banco: armazena os dados em /var/lib/postgresql/data.
- fotos: armazena as imagens enviadas em /app/uploads.

O arquivo db/init.sql é montado como somente leitura e executado
na primeira inicialização do banco com o diretório de dados vazio.

O diagrama completo está disponível em docs/arquitetura.md.

## 3. Pré-requisitos

É necessário ter instalado:

- Docker com o mecanismo em execução.
- Docker Compose.

No Windows com WSL 2, a integração com a distribuição Linux deve estar habilitada no Docker Desktop.

Não é necessário instalar Python ou PostgreSQL diretamente no computador.

## 4. Configuração do ambiente

Na pasta principal do projeto, copie o arquivo de exemplo:

```bash
cp .env.example .env
```

Abra o arquivo:

```bash
nano .env
```

Configure as variáveis:

- POSTGRES_DB: nome do banco.
- POSTGRES_USER: usuário do banco.
- POSTGRES_PASSWORD: senha do banco.

O arquivo .env não deve ser enviado ao GitHub, pois contém credenciais.

## 5. Iniciar a aplicação

Na pasta principal do projeto, execute:

```bash
docker compose up -d --build
```

O Docker Compose constrói a imagem web e inicia os três serviços
principais: db, web e nginx.

Para verificar os contêineres:

```bash
docker compose ps
```

O PostgreSQL deve aparecer como healthy, e os serviços web e
nginx devem estar em execução.

A aplicação fica disponível em:

http://localhost:8080

O fluxo das requisições é:

Navegador -> Nginx -> Flask -> PostgreSQL

A porta 8080 pertence ao Nginx. O Flask utiliza apenas a porta
5000 na rede Docker, sem publicação direta no computador.

## 6. Rotas da API

| Método | Rota | Funcionalidade |
|---|---|---|
| GET | /health | Verifica a aplicação e a conexão com o banco |
| GET | /itens | Lista os objetos cadastrados |
| GET | /itens?local=texto | Filtra os objetos por local |
| POST | /itens | Cadastra um objeto com foto opcional |
| GET | /fotos/<nome> | Retorna uma foto armazenada |
| GET | /estatisticas | Mostra o total de itens e fotos |

## 7. Testes da aplicação

Verificar a conexão com o banco:

```bash
curl http://localhost:8080/health
```

Listar todos os itens:

```bash
curl http://localhost:8080/itens
```

Filtrar pela Biblioteca:

```bash
curl "http://localhost:8080/itens?local=biblio"
```

Cadastrar um objeto com foto:

```bash
curl -F "descricao=Oculos de grau" \
     -F "local=Bloco B" \
     -F "foto=@oculos.jpg" \
     http://localhost:8080/itens
```

A imagem oculos.jpg deve estar na pasta em que o comando for executado.

Consultar estatísticas:

```bash
curl http://localhost:8080/estatisticas
```

Consultar os logs:

```bash
docker compose logs web --tail 20
```

Para visualizar uma foto, utilize o endereço retornado pelo campo foto na listagem de itens, acrescentando http://localhost:8080 antes do caminho.

## 8. Persistência de dados

Os dados do banco e as fotos são armazenados em volumes nomeados.

Para parar e remover os contêineres, preservando os volumes:

```bash
docker compose down
```

Para iniciar novamente:

```bash
docker compose up -d
```

Os dados permanecem disponíveis porque os volumes são mantidos.

Para remover também os volumes:

```bash
docker compose down -v
```

ATENÇÃO: esse último comando apaga os dados armazenados nos volumes do projeto.

Quando o ambiente é iniciado novamente com volumes vazios, o PostgreSQL executa o init.sql e cadastra os três objetos de exemplo.

## 9. Segurança

O ambiente utiliza algumas medidas básicas de segurança:

- Credenciais mantidas no arquivo .env.
- Arquivo .env ignorado pelo Git e excluído da entrega.
- Aplicação Flask executada com usuário não privilegiado.
- PostgreSQL sem porta publicada no host.
- Flask acessível pela rede Docker, sem porta publicada no host.
- Nginx funcionando como ponto de entrada HTTP.
- Comunicação dos serviços pela rede Docker rede_interna.
- Inicialização do serviço web após o banco ficar saudável.
- Adminer iniciado somente quando o profile debug é ativado.
- Porta do Adminer restrita ao endereço local 127.0.0.1.

Essas medidas não substituem autenticação de usuários,
HTTPS, gestão de segredos e outros controles necessários
para uma implantação em produção.

## 10. Evidências e documentação

Os documentos do projeto estão na pasta docs:

- RELATORIO.md: explicação técnica da configuração.
- REFLEXAO.md: questões de reflexão e equivalência com serviços de nuvem.
- arquitetura.md: diagrama da arquitetura.
- evidencias/: resultados dos testes realizados.

## 11. Bônus 1 — Nginx como proxy reverso

Foi adicionado o serviço nginx, utilizando a imagem nginx:alpine.

O arquivo nginx/default.conf configura o encaminhamento das
requisições para o serviço web, no endereço http://web:5000.

O Nginx publica a porta 8080 e a aplicação Flask permanece
acessível somente pela rede Docker.

O funcionamento foi confirmado por uma requisição à rota
/health, com resposta HTTP 200 e banco conectado.

## 12. Bônus 2 — Backup e restauração

O volume fotos foi copiado para um arquivo compactado utilizando
um contêiner temporário Alpine e o comando tar.

Para gerar um backup:

```bash
docker run --rm   -v achados-perdidos_fotos:/origem:ro   -v "$PWD/docs/evidencias:/backup"   alpine:3.20   sh -c 'tar -czf /backup/fotos_backup.tar.gz -C /origem .'
```

O arquivo é salvo em:

docs/evidencias/fotos_backup.tar.gz

Para restaurar em um volume separado:

```bash
docker volume create achados-perdidos_fotos_restauradas

docker run --rm   -v achados-perdidos_fotos_restauradas:/destino   -v "$PWD/docs/evidencias:/backup:ro"   alpine:3.20   sh -c 'tar -xzf /backup/fotos_backup.tar.gz -C /destino'
```

A restauração foi testada em um volume separado, sem modificar
os dados originais. Os arquivos restaurados foram comparados
com os arquivos de origem, confirmando que eram idênticos.

## 13. Bônus 3 — Adminer com profile debug

O Adminer permite consultar os dados do PostgreSQL por uma
interface web e utiliza o profile debug.

Para iniciar o ambiente com o Adminer:

```bash
docker compose --profile debug up -d
```

O painel fica disponível em:

http://localhost:8081

Informações para conexão:

- Sistema: PostgreSQL
- Servidor: db
- Usuário: definido em POSTGRES_USER no arquivo .env
- Senha: definida em POSTGRES_PASSWORD no arquivo .env
- Banco: definido em POSTGRES_DB no arquivo .env

Para desativar apenas o Adminer:

```bash
docker compose --profile debug stop adminer
docker compose --profile debug rm -f adminer
```

Os outros serviços continuam funcionando normalmente.

O painel foi testado com acesso à tabela itens e aos quatro
registros existentes no momento da verificação.

## 14. Encerramento

O projeto demonstra a criação de um ambiente conteinerizado
com Docker Compose, separando aplicação, banco de dados
e proxy reverso.

Também foram implementadas persistência de dados, backup
e restauração do volume de fotos e um painel administrativo
opcional para o PostgreSQL.

Os resultados dos testes e as evidências estão na pasta docs.
