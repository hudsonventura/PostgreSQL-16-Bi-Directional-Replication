# PostgreSQL-16-Bi-Directional-Replication
Exemplo de replicação bidirecional no Postgres e teste de estresse  
Na versão 16 do PostgreSQL foi implementada uma replicação bidirecional, possibilitando a criação de um cluster de banco de dados. Cada nó do cluster possui propriedades de leitura e escrita.  
<br>
Neste repositório que encontrará exemplos para uso e configuração de um *cluster* e um teste de estresse, para validar o funcionamento do *cluster*.  
<br>
Creditos a Tristen Raab  
Fonte: https://www.highgo.ca/2023/12/18/new-in-postgresql-16-bi-directional-logical-replication/  
<br>
<br>
<br>

# Teoria

## Replicação Lógica

A replicação lógica é suportada desde o PostgreSQL 10 e recebeu inúmeras atualizações e melhorias nos anos seguintes. A replicação lógica é o processo de cópia (ou seja, replicação) de objetos de dados representados como suas alterações. Dessa forma, podemos copiar apenas alterações específicas de objetos, como tabelas, em vez de bancos de dados inteiros, e transmitir essas alterações em diferentes plataformas e versões. Tudo isso contrasta com a replicação física, que usa endereços de bloco exatos e, como resultado, está limitada a copiar apenas bancos de dados inteiros e não pode transmitir entre plataformas ou versões, pois os dados devem corresponder em ambas.  

A replicação lógica também introduz dois elementos muito importantes, necessários para a compreensão de sua contraparte bidirecional. Estes são Publicadores (*Publishers*) e Assinantes (*Subscribers*) , você pode considerá-los como um nó líder (*Publisher*) e um nó seguidor (*Subscriber*). O Publicador reunirá suas alterações recentes e as enviará como uma lista ordenada de comandos ao Assinante. Uma vez recebido, o Assinante pega esta série de comandos e aplica-a aos seus dados. Se ambos os bancos de dados iniciarem com os mesmos dados, o Assinante estará atualizado com o Publicador.

<br>
<br>
<br>

## Replicação Lógica bidirecional

Agora que entendemos o que é replicação lógica, o que a replicação bidirecional está fazendo de diferente? Resumindo, a replicação lógica bidirecional ocorre quando todos os nós na replicação são Publicador e Assinante. Cada banco de dados agora pode lidar com solicitações de leitura e gravação, e todas as alterações serão transmitidas entre si. Este é o aspecto Bidirecional, pois em vez de as mudanças fluírem numa direção como antes, elas fluem em ambas as direções.  

O que o Postgres 16 adiciona é um novo parâmetro à instrução WITH que filtra a replicação de determinados nós. A replicação lógica bidirecional usa este parâmetro COM (ORIGIN = NONE), que filtra toda a replicação de conexões com origens que não são NONE. Essencialmente, isso só permite que dados recém-adicionados sejam replicados. Você provavelmente pode ver por que isso acontece. Se um banco de dados inserir novos dados e replicá-los para um segundo, esse segundo banco de dados replicará os dados também inserindo-os, acionando assim outra replicação para o banco de dados original. Rapidamente obtemos um loop infinito de replicação, por isso esta opção é necessária para manter tudo finito.

# Requisitos
Será utilizado um arquivo docker-compose.yml e cadastrar dois ou mais contêineres do Postgres.  
Para isso é necessário ter o conhecimento do funcionamento e ter instalado o `Docker` e `Docker Compose`.  
<br>

# Procedimento

1. Criar o arquivo docker-compose.yml  

``` YAML
services:

  postgres1:
    image: postgres:16
    container_name: postgres1
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
     - ./pgdata1:/var/lib/postgresql/data
    networks:
      - db

  postgres2:
    image: postgres:16
    container_name: postgres2
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
     - ./pgdata2:/var/lib/postgresql/data
    networks:
      - db


networks:
  db:
    driver: bridge
```

Note que foi criado uma rede unica entre os contêineres para que eles se *enxerguem*.  
Outro ponto é que foi alterado a porta do contêiner 2 para que não haja confluito.  
Por fim, os dados foram persistidos em volumes/diretórios diferentes.  
<br>

2. Suba dos contêineres

```bash 
docker compose up
```
<br>

3. Ajuste no `postgres.conf`  

Depois dos bancos criados, é necessário ajustar os parametros dos dois bancos no arquivo `postgres.conf`

``` conf
port = 5432
wal_level = logical
```
<br>

4. Reinicie os contêineres

```bash 
docker compose restart
```
<br>

5. Criar *Publishers* e *Subscribers* em cada banco

Para isso acesse o seu banco via PGAdmin ou outro cliente do Postgresl no banco 1 

```SQL 
CREATE PUBLICATION mypub1 FOR TABLE mytable;
CREATE SUBSCRIPTION mysub1 CONNECTION 'host=postgres2 port=5432 user=postgres dbname=postgres password=postgres' PUBLICATION mypub2 WITH(ORIGIN = NONE);
```

E depois no banco 2  

```SQL 
CREATE PUBLICATION mypub2 FOR TABLE mytable;
CREATE SUBSCRIPTION mysub2 CONNECTION 'host=postgres1 port=5432 user=postgres dbname=postgres password=postgres' PUBLICATION mypub1 WITH(ORIGIN = NONE); 
```

Veja que a publicação é feita a nivel de tabela, então será necessário executar o mesmo comando para todas as tabelas em que a replicação for necesária.  
As tabelas precisam ter o mesmo nome e mesma estrutura. O Postges não replica a criação de tabelas e demais comandos `DDL`.  