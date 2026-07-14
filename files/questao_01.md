# Questão 01 – Dockerfile para o Lift

## Entrega

---

# Prompt (Framework R-T-F)

```text
# Role

Você é um Engenheiro DevOps Sênior especializado em Docker, Kubernetes, Python e segurança de containers. Possui ampla experiência na criação de imagens Docker para ambientes de produção, seguindo as melhores práticas da OCI, Docker, Kubernetes e CIS Benchmark.

# Task

Crie um Dockerfile para uma aplicação Python/Flask considerando as seguintes informações:

## Estrutura do projeto

lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py

## Conteúdo do requirements.txt

Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9

## Informações da aplicação

- A aplicação utiliza Flask.
- O servidor de produção é executado através do Gunicorn.
- O comando utilizado é:

gunicorn --bind 0.0.0.0:8080 --workers 4 app:app

## Requisitos obrigatórios

- Utilizar uma imagem oficial do Python.
- Preferir uma imagem slim para reduzir o tamanho final.
- Otimizar o cache de build copiando inicialmente apenas o requirements.txt.
- Instalar somente as dependências necessárias.
- Utilizar pip com --no-cache-dir.
- Não incluir a pasta tests na imagem final.
- Criar um usuário não-root para execução da aplicação.
- Definir o diretório de trabalho como /app.
- Copiar apenas os arquivos necessários para execução da aplicação.
- Expor a porta 8080.
- Declarar as variáveis de ambiente DATABASE_URL e API_KEY sem definir valores, pois serão fornecidas em tempo de execução.
- Adicionar um HEALTHCHECK adequado.
- Utilizar CMD em formato JSON para iniciar o Gunicorn.

# Format

Retorne somente
1. O Dockerfile completo.
2. Uma breve explicação (máximo de 10 linhas) descrevendo as boas práticas utilizadas.
```

---

# Modelo

**Modelo utilizado:** GPT-5.5 (ChatGPT)

---

# Output

```dockerfile
# Imagem oficial e otimizada do Python
FROM python:3.12-slim

# Evita geração de arquivos .pyc e melhora o logging
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Diretório da aplicação
WORKDIR /app

# Cria usuário sem privilégios
RUN addgroup --system app && \
    adduser --system --ingroup app app

# Copia primeiro as dependências para aproveitar o cache
COPY requirements.txt .

# Instala dependências
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copia apenas os arquivos necessários
COPY app.py .
COPY lib ./lib

# Ajusta permissões
RUN chown -R app:app /app

# Executa como usuário não-root
USER app

# Variáveis de ambiente fornecidas em runtime
ENV DATABASE_URL=""
ENV API_KEY=""

# Porta utilizada pela aplicação
EXPOSE 8080

# Healthcheck simples
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
CMD python -c "import requests; requests.get('http://localhost:8080')" || exit 1

# Inicialização da aplicação
CMD ["gunicorn","--bind","0.0.0.0:8080","--workers","4","app:app"]
```

---

# Justificativa

O prompt foi estruturado utilizando o framework **R-T-F (Role, Task e Format)** para fornecer contexto suficiente ao modelo e obter uma resposta objetiva e alinhada às boas práticas.

## Role

O papel da IA é definido logo no início do prompt:

> "Você é um Engenheiro DevOps Sênior especializado em Docker, Kubernetes, Python e segurança de containers."

Essa definição orienta o modelo a responder como um especialista no assunto, aumentando a qualidade técnica da solução.

## Task

A tarefa descreve claramente o problema a ser resolvido e todas as restrições do cenário:

- Estrutura do projeto.
- Dependências da aplicação.
- Comando de execução.
- Requisitos obrigatórios.
- Boas práticas esperadas.

Isso reduz ambiguidades e garante que todos os requisitos do enunciado sejam considerados na geração do Dockerfile.

## Format

A seção **Format** determina exatamente como a resposta deve ser apresentada:

1. Dockerfile completo.
2. Breve explicação das boas práticas utilizadas.

Dessa forma, evita respostas excessivamente longas ou com informações que não fazem parte da entrega solicitada.
