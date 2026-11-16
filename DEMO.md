# Roteiro da demonstração

Cada operação de CRUD é feita **na tela** e, logo depois, é mostrada **no banco**, para provar que cada dado foi parar no banco certo. Tempo estimado: 10 a 12 minutos.

Os comandos de terminal são para o **PowerShell**, na pasta do projeto.

---

## 0. Antes da aula (uns 10 minutos antes)

1. Abra o **Docker Desktop uma vez só** e espere ficar verde ("Engine running"). Não clique de novo se demorar.
2. Suba o projeto:
   ```powershell
   docker compose up -d --build
   ```
3. Espere os bancos ficarem prontos. O Cassandra leva 1 a 2 minutos; todos devem aparecer `Up`, e os bancos `healthy`:
   ```powershell
   docker compose ps
   ```
4. Abra **http://localhost:5173**. As **3 bolinhas no topo** (PostgreSQL, MongoDB, Cassandra) devem estar verdes.
5. Deixe **um carro estacionado há 3h10** na Garagem Paulista, para mostrar uma cobrança de verdade na saída:
   ```powershell
   $e = (Get-Date).ToUniversalTime().AddMinutes(-190).ToString("o"); Invoke-RestMethod -Method Post -Uri http://localhost:3003/sessoes/checkin -ContentType 'application/json' -Body (@{ garagem_id = '6720a0000000000000000001'; placa = 'TIC3K10'; entrada = $e } | ConvertTo-Json)
   ```
6. Deixe um **terminal aberto ao lado do navegador**, para os comandos de "mostrar no banco".

> Para começar com os bancos **zerados**, só com os dados de exemplo, rode `docker compose down -v` e depois `docker compose up -d --build`. Isso apaga tudo o que foi cadastrado.

---

## 1. Garagem: CRUD no MongoDB

| Passo | Na tela (aba **Garagens**) |
|---|---|
| **Read** | Mostre os cartões. Cada garagem tem **"Campos específicos"** diferentes: carregadores elétricos na Paulista, lava-rápido e horário em Pinheiros. É o schema flexível. |
| **Create** | **Cadastrar garagem**: nome `Garagem Santos`, cidade `Santos`, UF `SP`, carro com 30 vagas, tolerância 15, 1ª hora 12, hora adicional 6. Em **Campos extras**, escreva `{"vista_mar": true}`. Clique em **Salvar garagem**. |
| **Update** | No cartão novo, clique em **Editar**, mude as vagas para `40` e salve. |

**Mostre no banco** (o documento com o campo livre `vista_mar`):
```powershell
docker compose exec mongo mongosh garagem_garagens --quiet --eval "db.garagens.findOne({ nome: 'Garagem Santos' })"
```

> Se o professor perguntar "então o MongoDB aceita qualquer coisa?": tente inserir uma garagem **direto no banco**, sem passar pela aplicação. O próprio MongoDB recusa (`Document failed validation`), porque o validador `$jsonSchema` exige nome, endereço, vagas e tarifas:
> ```powershell
> docker compose exec mongo mongosh garagem_garagens --quiet --eval "db.garagens.insertOne({ nome: 'Sem vagas' })"
> ```

> Use nomes **sem acento** nos comandos de terminal: o PowerShell estraga caracteres como "ç" e "ã" ao repassá-los para o Docker.

O **Delete** da garagem fica para o fim (passo 5), porque ela ainda vai ser usada no plano.

---

## 2. Cliente e veículo: CRUD no PostgreSQL

| Passo | Na tela (aba **Clientes**) |
|---|---|
| **Read** | Mostre a lista e a busca, e o tipo derivado: **Mensalista** ou **Avulso**. |
| **Create** | **Cadastrar cliente**: nome `Ana Souza`, CPF `123.456.789-09`, e o veículo `ANA1B23`, modelo `Kwid`. Clique em **Salvar cliente**. *Cliente e veículo são gravados na mesma transação.* |
| **Update (cliente)** | **Editar dados**: coloque o telefone `(11) 95555-0000` e clique em **Salvar alterações**. |
| **Create / Update / Delete (veículo)** | **Adicionar veículo** `ANA2C34`, depois **Editar** (mude a cor) e **Remover**. |

**Mostre no banco:**
```powershell
docker compose exec postgres psql -U garagem -d garagem_clientes -c "SELECT c.id, c.nome, c.cpf, c.telefone, v.placa, v.modelo FROM clientes c LEFT JOIN veiculos v ON v.cliente_id = c.id WHERE c.nome ILIKE '%ana%';"
```

> Para mostrar a transação: tente cadastrar outro cliente com a placa `ANA1B23`, que já existe. O PostgreSQL recusa, e **nem o cliente é criado** (confira na busca).

---

## 3. Plano de mensalista: PostgreSQL validando no MongoDB

| Passo | Na tela (cliente **Ana Souza**) |
|---|---|
| **Create** | Em **Planos de mensalista**, escolha `Garagem Santos`, valor `300` e clique em **Criar plano**. O badge da Ana muda para **Mensalista**. *O serviço de clientes perguntou ao serviço de garagens se a garagem existe: é a integridade entre bancos diferentes.* |
| **Update** | **Alterar valor** para `320`. |
| Regra no banco | Tente criar **outro** plano na mesma garagem: a tela recusa ("Cliente já possui um plano ativo nesta garagem"). Quem impede é o índice único parcial do PostgreSQL. |

**Mostre no banco:**
```powershell
docker compose exec postgres psql -U garagem -d garagem_clientes -c "SELECT p.id, c.nome, p.garagem_id, p.valor_mensal, p.ativo, p.data_inicio FROM planos p JOIN clientes c ON c.id = p.cliente_id WHERE c.nome ILIKE '%ana%';"
```

---

## 4. Sessão: CRUD no Cassandra (os 3 serviços juntos)

Aba **Operação**, garagem **Garagem Santos**:

| Passo | Na tela |
|---|---|
| **Create (check-in)** | Em **Registrar entrada**, digite `ANA1B23`. Ela entra como **mensalista**, e um carro azul aparece na **planta da garagem**. |
| **Create** de novo | Registre a entrada de `XYZ9A99`, uma placa não cadastrada: entra como **avulso**, carro cinza na planta. |
| Concorrência | Tente registrar `ANA1B23` de novo: **recusado**, porque o veículo já tem uma sessão aberta (lightweight transaction do Cassandra). |
| **Delete** | Na tabela abaixo da planta, clique em **Cancelar** na linha da `XYZ9A99`. É o caso de "check-in registrado por engano": a vaga é liberada. |
| **Update (check-out)** | Clique em **Registrar saída** da `ANA1B23` e depois em **Confirmar saída**: **R$ 0,00**, porque é mensalista. |

Agora troque a garagem para **Garagem Paulista** e faça o **check-out da `TIC3K10`**, o carro "estacionado há 3h10" do passo 0: **R$ 42,00** (R$ 18 da 1ª hora + 3 × R$ 8). O recibo mostra a regra e a tarifa usada.

**Mostre no banco** (a mesma sessão gravada em tabelas diferentes, uma por consulta):
```powershell
docker compose exec cassandra cqlsh -e "SELECT placa, garagem_nome, entrada, saida, status, valor FROM garagem_sessoes.sessoes_por_placa WHERE placa = 'ANA1B23';"
```
```powershell
docker compose exec cassandra cqlsh -e "SELECT placa, entrada, status, valor FROM garagem_sessoes.sessoes_por_garagem WHERE garagem_id = '6720a0000000000000000001' AND dia = '$((Get-Date).ToUniversalTime().ToString('yyyy-MM-dd'))' LIMIT 5;"
```
> O Cassandra mostra os horários em **UTC** (3 horas à frente de Brasília). O segundo comando lê exatamente **uma partição**: a garagem no dia de hoje, em UTC. Depois das 21h de Brasília o dia em UTC já virou, então o carro do passo 0 pode estar na partição do dia anterior.

**Read:** aba **Relatórios**, escolha **30 dias**. Mostre a linha *"31 partição(ões) (garagem_id, dia) lidas em paralelo em X ms"*, o faturamento e o gráfico. Em **Histórico de um veículo**, busque `ANA1B23`. O **Excluir** de cada linha também é um Delete de sessão.

---

## 5. Os Deletes que faltam

| Entidade | Na tela |
|---|---|
| Plano | Cliente Ana: **Encerrar plano** (vira "Encerrado") e depois **Excluir**. |
| Cliente | **Excluir cliente**: o diálogo avisa que veículos e planos também saem (ON DELETE CASCADE). |
| Garagem | Aba Garagens, cartão **Garagem Santos**: **Excluir**. |

Para provar que saíram, rode de novo os comandos do PostgreSQL (passo 2) e do MongoDB (passo 1): vêm vazios.

---

## 6. Extra: resiliência (se der tempo)

```powershell
docker compose stop clientes
```
A bolinha do **PostgreSQL** fica vermelha em até 10 s. Registre uma entrada qualquer na Operação: **funciona mesmo assim**, com o aviso de que o cliente entrou como avulso e será reavaliado na saída. Depois religue:
```powershell
docker compose start clientes
```

---

## Se algo der errado na hora

| Sintoma | O que fazer |
|---|---|
| Bolinha vermelha | `docker compose ps` mostra qual container caiu; `docker compose restart <nome>` (ex.: `sessoes`). |
| Cassandra demorando | Normal na primeira subida. Espere ele aparecer como `healthy` em `docker compose ps`. |
| Docker Desktop não abre | Não clique várias vezes. Feche tudo, espere e abra **uma vez**. |
| Tela desatualizada | Recarregue com **F5**. |
