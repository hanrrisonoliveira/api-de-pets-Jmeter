# Teste de Performance – API de Pets (Apache JMeter)

Projeto de teste de performance com **Apache JMeter** para a [API de Pets](https://github.com/tiagotaq/api-de-pets).

O arquivo `API_de_Pets.jmx` executa os endpoints `POST /pets` e `GET /pets` com dois modelos de carga, **fechado** e **aberto**, sempre fazendo o login antes.

## Estrutura do plano de teste

```text
Teste de API de Pets
├── setUp Thread Group
│   ├── HTTP Header Manager                   (Content-Type: application/json)
│   └── POST /login
│       ├── Extrair token                     (JSON Extractor → tokenExtraido)
│       └── JSR223 PostProcessor - Salvar token (salva na propriedade tokenProp)
├── Thread Group - Modelo Fechado
│   ├── User Defined Variables
│   ├── HTTP Header Manager                   (Content-Type + Authorization: Bearer ${__property(tokenProp)})
│   ├── POST /pets
│   ├── GET /pets
│   └── Listeners (Summary Report, View Results Tree, gráficos jp@gc)
└── Open Model Thread Group - Modelo Aberto
    ├── User Defined Variables
    ├── HTTP Header Manager                   (Content-Type + Authorization: Bearer ${__property(tokenProp)})
    ├── POST /pets
    ├── GET /pets
    └── Listeners (Summary Report, View Results Tree, gráficos jp@gc)
```

### Login

O **setUp Thread Group** roda antes dos demais Thread Groups e faz o `POST /login` com o usuário `tiago` / `123456`. O token retornado é extraído e salvo como **propriedade** do JMeter (`tokenProp`), porque propriedades são visíveis para todos os Thread Groups. Os dois modelos de carga enviam esse token no header `Authorization: Bearer ${__property(tokenProp)}`.

### Modelo fechado – `Thread Group - Modelo Fechado`

Quantidade **fixa** de usuários simultâneos. Cada usuário só envia a próxima requisição depois de receber a resposta da anterior.

| Configuração | Valor |
| --- | --- |
| Número de usuários (threads) | 500 |
| Ramp-up | 30 s |
| Loop Count | Infinito |
| Duração | 60 s |

### Modelo aberto – `Open Model Thread Group - Modelo Aberto`

Os usuários **chegam numa taxa definida** (usuários por segundo), sem depender do tempo de resposta da API.

```text
rate(0) random_arrivals(20 sec) rate(40/sec) random_arrivals(40 sec) rate(40/sec) random_arrivals(10 sec) rate(0)
```

| Fase | Duração | Taxa de chegada |
| --- | --- | --- |
| Subida | 20 s | de 0 até 40 usuários/s |
| Carga constante | 40 s | 40 usuários/s |
| Descida | 10 s | de 40 até 0 usuários/s |

Cada usuário que chega executa `POST /pets` e `GET /pets` uma vez.

## Pré-requisitos

1. **Node.js** (para rodar a API)
2. **Java 8+** (recomendado Java 17+)
3. **Apache JMeter 5.6.3**: [download](https://jmeter.apache.org/download_jmeter.cgi). O Open Model Thread Group existe a partir da versão 5.5.
4. **JMeter Plugins Manager** com os plugins usados nos listeners:
   - Baixe o [`jmeter-plugins-manager.jar`](https://jmeter-plugins.org/install/Install/) e coloque em `apache-jmeter-5.6.3/lib/ext/`
   - Reinicie o JMeter e abra **Options → Plugins Manager**
   - Instale:
     - **3 Basic Graphs** (`jpgc-graphs-basic`): Active Threads Over Time, Transactions per Second e Response Times Over Time
     - **Custom Thread Groups** (`jpgc-casutg`): Throughput Shaping Timer (ele está desabilitado no plano, mas o plugin precisa estar instalado para o arquivo abrir sem erro)

## 1. Subir a API de Pets

```bash
git clone https://github.com/tiagotaq/api-de-pets.git
cd api-de-pets
npm install
npm start
```

A API sobe em `http://localhost:3000`. Para conferir se ela está no ar, acesse `http://localhost:3000/status` ou a documentação em `http://localhost:3000/api-docs`.

> A API guarda os dados em memória. Os pets criados durante o teste ficam acumulados e a resposta do `GET /pets` cresce. Reinicie a API (`Ctrl + C` e `npm start`) antes de cada nova execução para comparar resultados.

## 2. Executar o teste

### Opção A – Pela interface gráfica (para conferir e depurar)

1. Abra o JMeter (`bin/jmeter.bat` no Windows, `bin/jmeter` no Linux/macOS).
2. **File → Open** e selecione `API_de_Pets.jmx`.
3. Clique em **Start** (▶).
4. Acompanhe os resultados em **Summary Report**, **View Results Tree** e nos gráficos **jp@gc** de cada Thread Group.

> Com 500 usuários, a interface gráfica consome bastante memória e pode afetar os resultados. Use a GUI para validar o script, por exemplo reduzindo os usuários temporariamente. Para a carga de verdade, use a linha de comando.

### Opção B – Pela linha de comando (recomendado para a carga)

Na pasta onde está o `API_de_Pets.jmx`:

```bash
jmeter -n -t API_de_Pets.jmx -l resultados/resultado.jtl -e -o resultados/relatorio
```

| Parâmetro | Significado |
| --- | --- |
| `-n` | Modo sem interface (non-GUI) |
| `-t` | Arquivo do plano de teste |
| `-l` | Arquivo onde os resultados brutos são gravados |
| `-e -o` | Gera o relatório HTML na pasta indicada (a pasta precisa estar vazia ou não existir) |

Ao terminar, abra `resultados/relatorio/index.html` no navegador.

> Se o comando `jmeter` não for reconhecido, use o caminho completo, por exemplo `C:\apache-jmeter-5.6.3\bin\jmeter.bat -n -t ...`.

## Executar um modelo de cada vez

Por padrão, o **Modelo Fechado** e o **Modelo Aberto** rodam **ao mesmo tempo**, logo depois do setUp. Para medir cada modelo separadamente, escolha uma destas opções:

- **Rodar um após o outro:** clique em **Teste de API de Pets** (Test Plan) e marque **Run Thread Groups consecutively (i.e. one at a time)**.
- **Rodar só um dos modelos:** clique com o botão direito no Thread Group que não quer executar e escolha **Disable**. Não desabilite o setUp Thread Group, porque é ele que faz o login.

## Alterar endereço, porta ou carga

- **Host, porta e protocolo:** as variáveis `protocol`, `baseUrl` e `port` ficam em **User Defined Variables**, que aparece no Test Plan e dentro de cada Thread Group. Altere em todos os lugares.
- **Usuário do login:** edite o corpo do `POST /login`, dentro do setUp Thread Group.
- **Modelo fechado:** altere o número de threads, o ramp-up e a duração no próprio Thread Group.
- **Modelo aberto:** altere o campo **Schedule** do Open Model Thread Group. Exemplos de sintaxe:
  - `rate(10/sec) random_arrivals(1 min)`: 10 usuários/s durante 1 minuto
  - `rate(0) random_arrivals(30 sec) rate(20/sec)`: subida de 0 a 20 usuários/s em 30 s

## Problemas comuns

| Sintoma | Causa provável |
| --- | --- |
| Erro ao abrir o `.jmx` falando em `kg.apc...` | Plugins jp@gc não instalados (veja os pré-requisitos) |
| `Connection refused` | API não está rodando em `localhost:3000` |
| `401 Token required` ou `403 Invalid token` | O login do setUp falhou ou o setUp Thread Group foi desabilitado |
| `409` no `POST /pets` | Nome de pet repetido para o mesmo usuário (o plano usa `${__UUID()}` para evitar isso) |
| `400` no `POST /pets` | Corpo sem `name` ou `type`, ou header `Content-Type` ausente |
