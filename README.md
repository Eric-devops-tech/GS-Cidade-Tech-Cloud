# Cidade Tech — Cloud

Infraestrutura em nuvem para a solução Cidade Tech, da Global Solution 2026/2.

O Cidade Tech organiza consultas comunitárias por tema e bairro. Gestores
publicam propostas, moradores autenticados registram um voto e a apuração fica
disponível após o encerramento. As consultas são consultivas e não substituem
votações oficiais.

## Arquitetura

Uma VM Linux no Azure hospeda dois containers separados: a API Cidade Tech, em
Java 17/Spring Boot, e PostgreSQL 17. Docker Compose conecta os serviços em uma
rede privada e mantém os dados do banco em um volume nomeado. Somente a API
publica a porta 8080; a NSG limita o acesso ao IP autorizado. O Oracle/PLSQL
permanece na entrega separada da disciplina Database.

O diagrama de infraestrutura Azure e as decisões estão em
[docs/arquitetura.md](docs/arquitetura.md). O roteiro de provisionamento está em
[docs/azure-cli.md](docs/azure-cli.md) e em
[azure-cli-comandos.txt](azure-cli-comandos.txt).

## O que já está integrado

- [x] API Java e Dockerfile em [api/](api/), sem histórico Git nem arquivos
  gerados do ZIP recebido.
- [x] Perfil persistente PostgreSQL, migration Flyway e modelo com quatro
  tabelas relacionadas: `temas`, `consultas`, `usuarios` e `votos`.
- [x] Testes da API: `mvn clean verify` passou com nove testes de integração.
- [x] Compose Cloud para API e PostgreSQL; o banco não publica porta no host.
- [x] Diagrama Azure e comandos de provisionamento documentados.
- [ ] Executar o Compose integrado e demonstrar CRUD e SQL após iniciar o Docker
  Engine.
- [ ] Provisionar a VM Azure, validar o acesso e gravar o vídeo da entrega.

O relatório entregue junto da API também registra a migration e os fluxos de
persistência PostgreSQL verificados pelo grupo Java. Neste ambiente, ainda não
foi possível subir os containers porque o Docker Engine não está ativo.

## Executar localmente

1. Copie `.env.example` para `.env` e preencha valores fortes e exclusivos:
   - `DB_PASSWORD`: senha do usuário da aplicação no PostgreSQL.
   - `JWT_SECRET`: segredo aleatório com pelo menos 32 bytes.
   - `GESTOR_INICIAL_EMAIL` e `GESTOR_INICIAL_SENHA`: credenciais do primeiro
     gestor (senha entre 10 e 24 caracteres).
2. Proteja o arquivo `.env`; ele é ignorado pelo Git e não deve ser compartilhado
   ou exibido em capturas/vídeos.
3. Na raiz do repositório, inicie os serviços:

   ```powershell
   docker compose up --build -d --wait
   docker compose ps
   docker compose logs -f api
   ```

4. A API fica disponível em `http://localhost:8080`; a documentação Swagger
   está em `http://localhost:8080/swagger-ui.html`.
5. Após provisionar o gestor inicial, remova `GESTOR_INICIAL_EMAIL` e
   `GESTOR_INICIAL_SENHA` do `.env`. A aplicação mantém a conta no banco.

A migration Flyway cria o esquema PostgreSQL; Hibernate valida o esquema, mas
não o altera. O serviço de banco chama-se `database` na rede Compose e usa
`jdbc:postgresql://database:5432/empauta`. O volume `postgres-data` preserva os
dados quando os containers são recriados.

## Testes

Com JDK 17 e Maven 3.9:

```powershell
mvn -f api/pom.xml clean verify
```

Os nove testes de integração cobrem autenticação, autorização, CRUD, validações,
voto único, apuração, filtros, paginação, cache, HATEOAS e endpoints de saúde e
documentação.

## Demonstração da API e consulta ao banco

Consulte [api/README.md](api/README.md) para o contrato completo, papéis e
exemplos das rotas. A API oferece operações de temas e consultas, autenticação
JWT, voto e apuração.

Depois de executar operações pelo Swagger, consulte diretamente os registros
persistidos no PostgreSQL:

```powershell
docker compose exec database psql -U empauta -d empauta -c "SELECT id, titulo, bairro, estado FROM consultas ORDER BY id;"
docker compose exec database psql -U empauta -d empauta -c "SELECT consulta_id, usuario_id, escolha FROM votos ORDER BY id;"
```

No Azure, execute essas consultas com `docker compose exec` dentro da VM, por
Azure Run Command. Os nomes de tabelas e colunas correspondem à migration
versionada em `api/src/main/resources/db/migration/`.

## Segurança e limites

- A porta PostgreSQL 5432 não é publicada; somente a API conecta-se ao banco
  pela rede Docker.
- A NSG do Azure deve permitir TCP/8080 somente do IP autorizado do avaliador.
- A VM não precisa de SSH público; use Azure Run Command para administração.
- Use somente dados sintéticos. O sistema não verifica identidade civil nem
  residência e não deve ser apresentado como votação oficial.
- Uma VM é suficiente para a demonstração acadêmica, mas é ponto único de falha
  e não oferece alta disponibilidade. O volume reside no disco da VM; apagar os
  recursos pode apagar os dados.

## Entrega Cloud

O vídeo deverá demonstrar a arquitetura e os recursos Azure, os containers e
seus logs, o CRUD pela API e uma consulta SQL diretamente no PostgreSQL. Use o
arquivo [azure-cli-comandos.txt](azure-cli-comandos.txt) como roteiro dos
comandos; substitua o IP reservado de exemplo pelo IP autorizado antes de criar
as regras de rede. O roteiro não provisiona recursos até ser executado.
