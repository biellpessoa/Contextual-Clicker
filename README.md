# Contextual Clicker — Checkpoint 5 (MVP Jogável & API Desacoplada)

> **Projeto interdisciplinar de Jogos / NLP (Agentes Virtuais) e Front-End — FIAP**  
> *Continuidade direta da CP4 — Idle/Clicker 1-bit procedural alimentado por Llama 3.2 via Ollama e FastAPI.*  
> **Integrantes:** Paulo Gabriel Pessoa da Silva (RM 566446) e William Stahl Sanches Furquim Garcia (RM 562800)

---

## 📌 Sumário Executivo

O **Contextual Clicker** é um jogo do gênero *idle/clicker* e *sandbox narrativo* com estética monocromática retrô (1-bit / CRT). O diferencial do jogo é que todo o seu universo — lore da picareta, consumíveis temporários (buffs), artefatos arqueológicos e epílogo de vitória — é moldado proceduralmente em tempo de execução a partir de uma **única palavra-chave** inserida pelo jogador no início da partida.

Neste Checkpoint 5 (CP5), o conceito e mockup da CP4 foram materializados em um **MVP 100% jogável**, com:
1. **Desacoplamento de Arquitetura (Front-End):** O jogo no navegador **nunca** acessa serviços de IA diretamente. Todas as requisições passam por uma API própria em **FastAPI**, estruturada no padrão **Router ➔ Provider**, protegida com autenticação via header `X-API-Key`.
2. **Integração Real de IA Generativa (NLP):** O modelo **Llama 3.2 (3B)** roda localmente via **Ollama**, gerando JSON estruturado com regras estritas de prompt para evitar alucinações.
3. **Mecanismo de Resiliência (Fallback Gracioso):** Caso o Ollama esteja desligado ou indisponível, a API e o front-end ativam um gerador determinístico sem interromper a partida, etiquetando a lore na interface como `FALLBACK`.
4. **Testes Automatizados:** Suíte de testes em Python (`pytest`, 16 testes) cobrindo autenticação, CORS, Swagger e Provider, e suíte em JavaScript (`node --test`, 7 testes) cobrindo a lógica matemática do jogo.

### 🧩 Divisão das Modalidades de IA Generativa
* **Texto em Tempo Real via API (Front-End & NLP):** O Llama 3.2 (3B) gera as picaretas, buffs, 8 artefatos e epílogo dinamicamente de acordo com o tema. Essa modalidade é consumida exclusivamente através da API própria em **FastAPI** (`POST /v1/ia-generativa/texto`), atendendo aos requisitos de rotas, providers e autenticação da disciplina de Front-End.
* **Imagens Pré-Geradas na Pré-Produção (NLP):** As 16 imagens de pixel art (`picareta-1` a `7`, `buff`, `artefato-1` a `8`) foram geradas no Bing Image Creator (DALL-E 3) na pré-produção offline e integradas como assets estáticos em `frontend/assets/img/`, sendo tratadas em tempo de execução via CSS (`invert(1)`, `contrast(5)`, `mix-blend-mode: screen`) para eliminar fundo branco e forçar a estética 1-bit.

---

## 🏗️ Arquitetura do Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                   NAVEGADOR DO JOGADOR                      │
│   Front-End Vanilla (HTML5 / CSS3 / ES6 / Web Audio API)    │
│   • Tela 1: Menu Principal (Seed / Palavra-Chave)           │
│   • Tela 2: Mina Ativa (Planeta, Picareta, Buffs, HUD)      │
│   • Tela 3: Museu de Artefatos (8 relíquias colecionáveis)  │
│   • Tela 4: Epílogo e Vitória                               │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP POST /v1/ia-generativa/texto
                               │ Header: X-API-Key
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    BACK-END (FastAPI)                       │
│                                                             │
│   [Security]  ➔ Validação de API Key (header X-API-Key)    │
│   [Router]    ➔ routers/ia_generativa.py                   │
│   [Provider]  ➔ providers/ia_provider.py (IATextoProvider)  │
│                   ├── Validação estrita de schema Pydantic  │
│                   ├── Normalização de chaves e acentuação   │
│                   └── Fallback determinístico (se offline)  │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP POST /api/chat (JSON mode)
                               │ Localhost:11434
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                   MOTOR DE IA LOCAL                         │
│   Ollama ➔ Llama 3.2 (3B) [Execução local sem custo de API] │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 Estrutura do Repositório

```
Contextual Clicker/
├── backend/
│   ├── .env.example              # Modelo de variáveis de ambiente
│   ├── .env                      # Configuração ativa local (ignorado no git)
│   ├── requirements.txt          # Dependências Python (FastAPI, Uvicorn, httpx, etc.)
│   ├── painel_streamlit.py       # Painel alternativo em Streamlit
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py               # Ponto de entrada FastAPI, CORS, rotas estáticas
│   │   ├── schemas.py            # Modelos Pydantic (validação de entrada/saída)
│   │   ├── prompts.py            # Prompts estritos baseados na CP4
│   │   ├── core/
│   │   │   ├── config.py         # Configurações lidas do .env
│   │   │   └── security.py       # Verificação de segurança (header X-API-Key)
│   │   ├── providers/
│   │   │   ├── ia_provider.py    # Chamada isolada ao Ollama (Llama 3.2)
│   │   │   └── fallback_provider.py # Gerador de contingência
│   │   └── routers/
│   │       └── ia_generativa.py  # Router protegido /v1/ia-generativa
│   └── tests/
│       └── test_api.py           # 16 testes automatizados (pytest)
│
├── frontend/
│   ├── index.html                # Aplicação do jogo (HTML5 semântico)
│   ├── painel.html               # Painel web interativo para testes de API
│   ├── css/
│   │   └── style.css             # Estilo monocromático 1-bit CRT scanlines
│   ├── js/
│   │   ├── config.js             # URL da API e chaves locais
│   │   ├── game-logic.js         # Lógica pura (mineração, upgrades, buffs, save)
│   │   ├── api.js                # Cliente HTTP para a API própria
│   │   ├── audio.js              # Sintetizador de áudio chiptune 8-bit
│   │   ├── icons.js              # Renderizador de pixel art SVG e sprites
│   │   └── main.js               # Controlador do DOM e ciclo de vida do jogo
│   └── assets/
│       └── img/
│           └── manifest.json     # Manifesto para carregamento de sprites
│
├── tests/
│   └── test_game_logic.js        # 7 testes unitários em Node.js (mecânicas do jogo)
│
├── monochrome_miner.html         # Mockup inicial entregue na CP4 (preservado)
└── README.md                     # Guia de instalação e documentação
```

---

## 🚀 Como Executar o Projeto

### Pré-requisitos
- **Python 3.10+** (recomendado Python 3.12)
- **Node.js 18+** (opcional, para rodar os testes unitários do front-end)
- **Ollama** com o modelo `llama3.2` instalado:
  ```bash
  ollama run llama3.2
  ```

---

### Passo 1: Configurar e Iniciar o Back-End

1. Abra um terminal na pasta `backend`:
   ```bash
   cd backend
   ```

2. Crie e ative o ambiente virtual:
   * **Windows (PowerShell):**
     ```powershell
     python -m venv .venv
     .\.venv\Scripts\Activate.ps1
     ```
   * **Linux / macOS:**
     ```bash
     python3 -m venv .venv
     source .venv/bin/activate
     ```

3. Instale as dependências:
   ```bash
   pip install -r requirements.txt
   ```

4. Configure o arquivo `.env`:
   ```powershell
   # No Windows:
   Copy-Item .env.example .env

   # No Linux/macOS:
   cp .env.example .env
   ```

5. Inicie o servidor FastAPI:
   ```bash
   uvicorn app.main:app --reload --port 8000
   ```

---

### Passo 2: Acessar o Jogo e a Documentação

Com o servidor rodando em `http://localhost:8000`:
* 🎮 **Jogo Completo:** [http://localhost:8000/jogo/](http://localhost:8000/jogo/) (ou abra `frontend/index.html` em um Live Server)
* 📖 **Documentação Swagger:** [http://localhost:8000/docs](http://localhost:8000/docs)
* 🧪 **Painel Web de Testes da API:** [http://localhost:8000/painel](http://localhost:8000/painel)

> 💡 **Dica para Gravação do Vídeo:** Para acelerar a demonstração e desbloquear os 8 artefatos rapidamente, acesse o jogo com o parâmetro de teste:  
> `http://localhost:8000/jogo/?demo=1` (multiplica o ganho de pedras por 4x).

---

## 🧪 Como Rodar os Testes Automatizados

### 1. Testes do Back-End (Python / pytest)
Na pasta `backend`, com o ambiente virtual ativado:
```bash
pytest -v
```
*Cobertura: 16 testes cobrindo autenticação com `X-API-Key`, erro 401, Swagger `/docs`, CORS, rotas de geração, validação Pydantic de payload e fallback do provider.*

### 2. Testes das Mecânicas do Jogo (JavaScript / Node.js)
Na raiz do projeto:
```bash
node --test tests/test_game_logic.js
```
*Cobertura: 7 testes cobrindo mineração por clique, progressão de picaretas, multiplicação de velocidade de buffs, produção ociosa (tick idle), desbloqueio do museu e persistência via LocalStorage.*

---

## 🔐 Autenticação da API

Todas as rotas sob `/v1/ia-generativa` exigem a presença do cabeçalho HTTP:
```http
X-API-Key: cc-dev-key-troque-me
```
Se a chave for omitida ou estiver incorreta, o endpoint responde imediatamente com **`401 Unauthorized`**. No Swagger (`/docs`), clique no botão verde **Authorize** no canto superior direito e insira a chave para testar.

---