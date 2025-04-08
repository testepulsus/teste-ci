# Pulsus-github-actions

Este repositório contém a configuração dos workflows de CI (Integração Contínua) para automatizar o processo de teste e análise de código. Os fluxos reutilizáveis foram projetados para garantir a qualidade e a segurança do código, além de fornecer feedback contínuo durante o desenvolvimento.

Workflows Implementados
1. Pytest
O workflow executa a suíte de testes usando o Pytest, garantindo que todas as funcionalidades da aplicação sejam validadas conforme esperado.

2. Pylint
Utilizamos o Pylint para realizar a análise estática de código Python, garantindo aderência às melhores práticas de código e conformidade com padrões de estilo.

3. Gitleaks
O Gitleaks é usado para detectar potenciais chaves secretas e credenciais expostas no repositório, ajudando a proteger informações sensíveis.

4. Check-optl
Este workflow verifica se a instrumentação de OpenTelemetry (optl) está configurada corretamente no projeto, garantindo que a rastreabilidade e o monitoramento da aplicação estejam em conformidade com as boas práticas. Tambem garante que o optl.py está na raiz do projeto

5. SonarQube
O SonarQube é usado para análise de qualidade de código, identificando problemas relacionados à segurança, bugs e vulnerabilidades no código.

6. Dependency-Track
Utilizamos o Dependency-Track para gerenciar as dependências do projeto, identificando bibliotecas desatualizadas e vulneráveis, e fornecendo alertas em tempo real sobre os riscos de segurança.

7.  o Bendit para realizar a análise e gerenciamento de dependências específicas de runtime, identificando incompatibilidades e prevenindo problemas que possam ocorrer durante a execução da aplicação. 

Todas as chamada estão sendo feitas pelo ".github/workflows/python-ci.yaml" e dentro de cada repositorio existe um python-cy.yaml que apota pro nosso fluxo de CI


# Documentação de CI/CD com GitHub Actions

Visão Geral
Este README detalha a implementação de uma pipeline de CI/CD utilizando GitHub Actions, com integração de ferramentas como SonarQube, pytest, pylint, gitleaks, bandit e Dependency-Track. A configuração foi feita utilizando workflows reutilizáveis, permitindo maior modularidade e facilidade de manutenção.

Objetivo
O objetivo deste README é explicar o funcionamento de cada YAML de configuração, descrevendo o propósito de cada etapa e como as ferramentas foram integradas para garantir qualidade de código, segurança e análise contínua de dependências.




# 1 - Fluxo da Pipeline

![image](https://github.com/user-attachments/assets/f7b08155-3949-44f2-acde-f73c43f8c054)








# Explicaçao de cada yaml do CI dentro do pulsus-github-actions

https://github.com/pulsus-mobi/pulsus-github-actions/tree/master

**check-optl.yaml**



```yaml
name: Check for optl.py  # Nome do fluxo de trabalho

on:  # Define quando o fluxo de trabalho deve ser acionado
  workflow_call:  # O fluxo de trabalho pode ser chamado de outro fluxo
    inputs: 
      active:  
        required: false  # A entrada não é obrigatória
        default: true  # Valor padrão da entrada
        type: boolean  # Tipo da entrada

jobs:  # Seção que define os jobs (tarefas) do fluxo de trabalho
  CHECK-FILE: 
    runs-on: ubuntu-latest  
    steps:  # Seção que define as etapas do job
      - name: Checkout code
        uses: actions/checkout@v3 

      - name: Verify optl.py exists  # Nome da etapa para verificar o arquivo
        run: |  # Executa um bloco de comandos
          if [ ! -f ./optl.py ]; then  # Verifica se o arquivo optl.py não existe
            echo "❌ The file optl.py is missing!"  # Mensagem de erro se o arquivo estiver ausente
            exit 1  # Encerra o job com erro (código 1)
          else  # Se o arquivo existir
            echo "✅ optl.py is present."  # Mensagem de sucesso
            
            
            
```

**common-bandit.yaml**


```yaml
name: Bandit  

on:  
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  # Define as entradas que podem ser passadas quando o workflow é chamado
      active:
        required: false  # Entrada não é obrigatória
        default: true  # Valor padrão da entrada é 'true'
        type: boolean  # O tipo de dado da entrada é booleano (verdadeiro/falso)

jobs:  
  bandit:
    name: Bandit 
    runs-on: ubuntu-latest 

    steps:  
      - name: Checkout code  
        uses: actions/checkout@v4 

      - name: Set up Python  # Etapa para configurar o ambiente Python
        uses: actions/setup-python@v5  # Utiliza a ação para configurar o Python
        with:
          python-version: '3.x'  # Define a versão do Python para a execução

      - name: Install Bandit  # Etapa para instalar o Bandit
        run: pip install bandit  # Comando para instalar a ferramenta Bandit via pip

      - name: Run Bandit  # Etapa para executar o Bandit
        id: bandit  # Define o ID da etapa para ser referenciado posteriormente
        run: |
          bandit -r . -f json -o bandit-report.json  # Executa o Bandit recursivamente no diretório atual e gera um relatório em formato JSON
        continue-on-error: true  # Permite que o job continue mesmo se o Bandit encontrar vulnerabilidades

      - name: Display Bandit JSON report  # Etapa para exibir o relatório JSON gerado pelo Bandit
        run: cat bandit-report.json  # Exibe o conteúdo do arquivo JSON no log

      - name: Generate Bandit Markdown Summary  # Etapa para gerar um resumo em Markdown dos resultados do Bandit
        run: |
          echo "## 🔴 Bandit Scan Results 🔴 " >> $GITHUB_STEP_SUMMARY  # Adiciona um cabeçalho no resumo do GitHub
          echo "| Filename | Issue | Severity | Line | View in Code |" >> $GITHUB_STEP_SUMMARY  # Define os cabeçalhos da tabela
          echo "| --- | --- | --- | --- | --- |" >> $GITHUB_STEP_SUMMARY  # Define o layout da tabela
          # Utiliza 'jq' para processar o arquivo JSON e gerar uma tabela formatada com link para o código
          cat bandit-report.json | jq -r '.results[] | "| \(.filename) | \(.issue_text) | \(.issue_severity) | \(.line_number) | [View in Code](https://github.com/${{ github.repository }}/blob/${{ github.sha }}/\(.filename)#L\(.line_number)) |"' >> $GITHUB_STEP_SUMMARY

      - name: Upload JSON Report  # Etapa para fazer upload do relatório JSON como artefato do workflow
        uses: actions/upload-artifact@v3  # Usa a ação para upload de artefatos na versão 3
        with:
          name: bandit-report.json  # Nome do artefato
          path: bandit-report.json  # Caminho do arquivo a ser enviado

      - name: Fail if vulnerabilities were detected  # Etapa para falhar o job se vulnerabilidades forem detectadas
        if: steps.bandit.outcome == 'failure'  # Condicional: só será executada se o Bandit falhar
        run: |
          echo "Bandit found vulnerabilities, failing the job..."  # Exibe mensagem indicando falha por vulnerabilidades
          exit 1  # Encerra o job com código de erro 1
          
```


**common-ci-dtrack.yaml**

```yaml

name: Dependency Track  

run-name: Dependency Track  

on:  
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  # Define as entradas que podem ser passadas quando o workflow é chamado
      active:  # Nome da entrada
        required: false  # Entrada não é obrigatória
        default: true  # Valor padrão da entrada é 'true'
        type: boolean  # Tipo de dado da entrada é booleano (verdadeiro/falso)

jobs:  
  dtrack: 
    if: ${{ inputs.active == true }}  # O job só será executado se o valor da entrada 'active' for verdadeiro
    runs-on: ubuntu-latest  # Define que o job será executado em um ambiente Ubuntu
    name: Dependency Track  

    steps:  

      - name: Checkout code  
        uses: actions/checkout@v4 

      - name: Download Syft  # Etapa para baixar o Syft, uma ferramenta de geração de SBOM
        id: download-syft  # Define um ID para a etapa para referenciar sua saída
        uses: anchore/sbom-action/download-syft@v0  # Utiliza a ação para baixar o Syft da Anchore

      - name: Generate SBOM  # Etapa para gerar o SBOM (Software Bill of Materials)
        run: ${{ steps.download-syft.outputs.cmd }} dir:./ -o cyclonedx-xml > bom.xml  # Executa o comando Syft para gerar o SBOM no formato CycloneDX e salva o arquivo como bom.xml

      - name: Upload SBOM  # Etapa para fazer upload do SBOM gerado para o servidor Dependency Track
        run: |
          curl -X "POST" "https://${{ secrets.SECRET_OWASP_DT_HOST }}/api/v1/bom" \  # Envia o SBOM para o OWASP Dependency Track via API
            -H 'Content-Type: multipart/form-data' \  # Define o cabeçalho de tipo de conteúdo para multipart/form-data
            -H "X-API-Key: ${{ secrets.SECRET_OWASP_DT_KEY }}" \  # Inclui a chave da API de acesso ao Dependency Track, armazenada nos segredos do GitHub
            -F "autoCreate=true" \  # Define o parâmetro autoCreate para criar o projeto automaticamente se ele não existir
            -F "projectName=${{ github.event.repository.name }}" \  # Usa o nome do repositório como nome do projeto
            -F "projectVersion=${{ github.head_ref }}" \  # Usa a versão atual do branch (head) como a versão do projeto
            -F "bom=@bom.xml"  # Anexa o arquivo SBOM (bom.xml) para upload
```

obs- o dtrack necessita de um secrets no repositorio para funcionar

SECRET_OWASP_DT_HOST - https://dtrack.pulsus.mobi
SECRET_OWASP_DT_KEY - https://dtrackapi.pulsus.mobi/


**common-ci-sonarqube.yaml**

```yaml

name: SonarQube  

on:
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  
      fetch-depth:  # Define a profundidade do clone do Git
        required: false  # Não é obrigatório
        default: 0  # O valor padrão é 0, que significa clonar o repositório completo
        type: number  # Tipo da entrada é um número
      timeout-minutes:  # Define o tempo máximo para o job de Quality Gate do SonarQube
        required: false  # Não é obrigatório
        default: 1  # O valor padrão é 1 minuto
        type: number  # Tipo da entrada é um número
      active:  # Define se o workflow está ativo ou não
        required: false  # Não é obrigatório
        default: true  # O valor padrão é 'true' (ativo)
        type: boolean  # Tipo da entrada é booleano (verdadeiro/falso)
    secrets:  # Define os segredos necessários para a execução
      SONAR_TOKEN:  # Token do SonarQube para autenticação
        required: true  # Obrigatório
      SONAR_HOST_URL:  # URL do servidor SonarQube
        required: true  # Obrigatório

jobs:  # Define os jobs (tarefas) do workflow
  sonarqube:  # Nome do job
    if: ${{ inputs.active == true }}  # O job só será executado se o valor da entrada 'active' for verdadeiro
    name: SonarQube  # Nome descritivo do job
    runs-on: ubuntu-latest  # Define que o job será executado em um ambiente Ubuntu

    steps:  # Define as etapas do job

      - name: Checkout code  # Etapa para fazer checkout do código do repositório
        uses: actions/checkout@v4  # Utiliza a ação de checkout na versão 4
        with:
          fetch-depth: ${{ inputs.fetch-depth }}  # Define a profundidade do clone do repositório. Profundidade de 0 para clonar o repositório inteiro, garantindo melhor análise de relevância no SonarQube
      
      - uses: sonarsource/sonarqube-scan-action@master  # Executa a ação de escaneamento do SonarQube
        env:  # Define variáveis de ambiente necessárias para o SonarQube
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # Token do SonarQube, passado como segredo
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}  # URL do servidor SonarQube, passado como segredo
      
      - uses: sonarsource/sonarqube-quality-gate-action@master  # Executa a verificação de Quality Gate no SonarQube
        timeout-minutes: ${{ inputs.timeout-minutes }}  # Define o tempo máximo de execução para o Quality Gate
        env:  # Define as variáveis de ambiente necessárias para o Quality Gate
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # Token do SonarQube, passado como segredo
````
obs - sonarqube necessita de secrets no repositorio para funcionar

SONAR_HOST_URL - https://sonarqube.pulsus.mobi
SONAR_TOKEN - o token é gerado no proprio sonarqube



**common-gitleaks.yaml**

```yaml

name: Gitleaks Scan 

on:  
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  # Define as entradas que podem ser passadas quando o workflow é chamado
      active:  # Define se o workflow está ativo ou não
        required: false  # Não é obrigatório
        default: true  # O valor padrão é 'true' (ativo)
        type: boolean  # Tipo de dado é booleano (verdadeiro/falso)
    secrets:  # Define os segredos necessários para a execução
      GITLEAKS_LICENSE:  # Licença do Gitleaks, se necessário para uso
        required: true  # Este segredo é obrigatório

jobs:  # Define os jobs (tarefas) do workflow
  scan:  # Nome do job
    name: gitleaks  # Nome descritivo do job
    runs-on: ubuntu-latest  # Define que o job será executado em um ambiente Ubuntu

    steps:  # Define as etapas do job

      - name: Checkout code  # Etapa para fazer checkout do código do repositório
        uses: actions/checkout@v4  # Utiliza a ação de checkout na versão 4
        with:
          fetch-depth: 0  # Clona o repositório completo, necessário para análise profunda do histórico

      - name: Install Gitleaks  # Instala o Gitleaks diretamente da fonte
        run: |
          curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.21.0/gitleaks_8.21.0_linux_x64.tar.gz | tar -xz  # Baixa o Gitleaks da versão especificada
          sudo mv gitleaks /usr/local/bin/gitleaks  # Move o binário para um local no PATH do sistema
          sudo chmod +x /usr/local/bin/gitleaks  # Torna o binário executável

      - name: Run Gitleaks and generate JSON report  # Executa o Gitleaks para detectar segredos no repositório
        id: gitleaks  # Define um ID para referenciar a saída da etapa
        run: |
          gitleaks detect --source . --verbose --log-opts="--all" --report-format=json --report-path=gitleaks-results.json --exit-code=1  # Executa o Gitleaks e gera um relatório no formato JSON
        continue-on-error: true  # Continua o workflow mesmo que Gitleaks detecte segredos

      - name: Display Gitleaks JSON report  # Exibe o conteúdo do relatório JSON gerado pelo Gitleaks
        run: cat gitleaks-results.json  # Exibe o arquivo JSON no log

      - name: Generate Gitleaks Markdown Summary  # Gera um resumo em Markdown dos segredos encontrados
        run: |
          echo "## 🔴 Gitleaks detected secrets 🔴" >> $GITHUB_STEP_SUMMARY  # Adiciona título ao resumo
          echo "| Rule ID | Commit | Secret URL | Start Line | Author | Date | File |" >> $GITHUB_STEP_SUMMARY  # Cria o cabeçalho da tabela
          echo "| --- | --- | --- | --- | --- | --- | --- |" >> $GITHUB_STEP_SUMMARY  # Formatação da tabela
          cat gitleaks-results.json | jq -r '.[] | "| \(.RuleID) | [\(.Commit)](https://github.com/${{ github.repository }}/commit/\(.Commit)) | [View Secret](https://github.com/${{ github.repository }}/blob/\(.Commit)/\(.File)#L\(.StartLine)) | \(.StartLine) | \(.Author) | \(.Date) | \(.File) |"' >> $GITHUB_STEP_SUMMARY  # Extrai as informações do JSON e gera a tabela com os resultados

      - name: Upload JSON Report  # Faz upload do relatório JSON gerado pelo Gitleaks
        uses: actions/upload-artifact@v3  # Utiliza a ação para subir o relatório como artefato
        with:
          name: gitleaks-results.json  # Nome do artefato
          path: gitleaks-results.json  # Caminho do arquivo a ser enviado

      - name: Fail if leaks were detected  # Falha o job se forem detectados segredos
        if: steps.gitleaks.outcome == 'failure'  # Condição: se a saída do Gitleaks for 'failure'
        run: |
          echo "Gitleaks found leaks, failing the job..."  # Exibe uma mensagem de falha
          exit 1  # Termina o job com status de falha
          
          
`````  
**python-ci-pylint.yaml**

````yaml

name: PyLint  
run-name: PyLint  

on:  
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  # Definição dos inputs passados ao workflow
      python-version:  # Versão do Python que será usada
        required: true  # Campo obrigatório
        type: string  # O tipo do dado é string (texto)
      active:  # Input para ativar ou desativar o workflow
        required: false  # Campo não obrigatório
        default: true  # O valor padrão é 'true', ou seja, ativo
        type: boolean  # Tipo booleano (verdadeiro/falso)

jobs:  # Define os jobs (tarefas) do workflow
  python-ci-pylint:  # Nome do job
    if: ${{ inputs.active == true }}  # O job só será executado se o input 'active' for 'true'
    runs-on: ubuntu-latest  # O job será executado em uma máquina com Ubuntu
    name: PyLint  # Nome descritivo do job

    steps:  # Define as etapas do job

      - name: Checkout code  # Etapa para fazer checkout do código do repositório
        uses: actions/checkout@v4  # Utiliza a ação de checkout na versão 4

      - name: Set up Python  # Etapa para configurar o ambiente Python
        uses: actions/setup-python@v5  # Ação que configura o Python na versão 5
        with:
          python-version: ${{ inputs.python-version }}  # Define a versão do Python a partir do input passado

      - name: Install dependencies  # Etapa para instalar as dependências do projeto
        run: pip install --no-cache-dir --upgrade -r requirements.txt  # Instala pacotes listados no 'requirements.txt' sem cache e com atualização

      - name: Run PyLint  # Executa o PyLint para checar a qualidade do código Python
        run: pylint **/*.py  # Execut
````

**python-ci-pytest.yaml**

````yaml

name: PyTest  
run-name: PyTest 

on: 
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  # Definição dos inputs passados ao workflow
      python-version:  # Versão do Python que será usada
        required: true  # Campo obrigatório
        type: string  # O tipo do dado é string (texto)
      active:  # Input para ativar ou desativar o workflow
        required: false  # Campo não obrigatório
        default: true  # O valor padrão é 'true', ou seja, ativo
        type: boolean  # Tipo booleano (verdadeiro/falso)

jobs:  # Define os jobs (tarefas) do workflow
  python-ci-pytest:  # Nome do job
    if: ${{ inputs.active == true }}  # O job só será executado se o input 'active' for 'true'
    runs-on: ubuntu-latest  # O job será executado em uma máquina com Ubuntu
    name: PyTest  # Nome descritivo do job

    steps:  # Define as etapas do job

      - name: Checkout code  # Etapa para fazer checkout do código do repositório
        uses: actions/checkout@v4  # Utiliza a ação de checkout na versão 4

      - name: Set up Python  # Etapa para configurar o ambiente Python
        uses: actions/setup-python@v5  # Ação que configura o Python na versão 5
        with:
          python-version: ${{ inputs.python-version }}  # Define a versão do Python a partir do input passado

      - name: Install dependencies  # Etapa para instalar as dependências do projeto
        run: pip install --no-cache-dir --upgrade -r requirements.txt  # Instala pacotes listados no 'requirements.txt' sem cache e com atualização

      - name: Install Coverage  # Etapa para instalar o pacote de cobertura de testes
        run: pip install coverage  # Instala o pacote 'coverage' para medir a cobertura de código

      - name: Check coverage rate (minimum 70%)  # Verifica se a cobertura mínima de testes é atingida
        run: coverage report --fail-under=70  # Gera um relatório de cobertura e falha se a cobertura for menor que 70%

      - name: Run PyTest  # Executa o PyTest para rodar os testes de unidade
        run: coverage run -m pytest  # Usa o 'coverage' para medir a cobertura ao executar o PyTest

      - name: Collect Coverage  # Coleta a cobertura em formato JSON
        run: coverage json -o coverage.json  # Gera um arquivo 'coverage.json' com os resultados da cobertura

      - name: Upload Coverage to Typo App  # Placeholder para enviar os dados da cobertura (em desenvolvimento)
        run: echo "Work in progress"  # Placeholder temporário, exibindo uma mensagem de que esta etapa está em progresso
````


**python-ci.yaml**

````yaml

name: Python CI  
run-name: Python CI  

on:  # Define quando o workflow é acionado
  workflow_call:  # Permite que o workflow seja chamado por outros workflows
    inputs:  # Parâmetros de entrada do workflow
      python-version:  # Especifica a versão do Python a ser usada
        required: true  # Esse campo é obrigatório
        type: string  # O tipo do dado é string (texto)
      python-pylint-active:  # Ativa/desativa o job do Pylint
        required: false  # Esse campo não é obrigatório
        default: true  # O valor padrão é 'true' (ativado)
        type: boolean  # Tipo booleano (verdadeiro/falso)
      python-pytest-active:  # Ativa/desativa o job do PyTest
        required: false
        default: true
        type: boolean
      common-sonarqube-active:  # Ativa/desativa o job do SonarQube
        required: false
        default: true
        type: boolean
      common-dtrack-active:  # Ativa/desativa o job do Dependency Track
        required: false
        default: false  # O valor padrão é 'false' (desativado)
        type: boolean
      common--git-leaks-active:  # Ativa/desativa o job do Gitleaks
        required: false
        default: true
        type: boolean
      check-optl-active:  # Ativa/desativa o job de verificação de OpenTelemetry (optl)
        required: false
        default: false  # O valor padrão é 'false'
        type: boolean
      common--bandit-active:  # Ativa/desativa o job do Bandit
        required: false
        default: true
        type: boolean
    secrets:  # Definição dos segredos (variáveis sensíveis) necessários
      SONAR_TOKEN:  # Token do SonarQube
        required: false  # Esse campo é opcional, depende do job
      SONAR_HOST_URL:  # URL do servidor do SonarQube
        required: false
      GITLEAKS_LICENSE:  # Licença do Gitleaks
        required: false

jobs:  # Definição dos jobs (tarefas) do workflow

  python-ci-pylint:  # Job de análise estática de código com Pylint
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/python-ci-pylint.yaml@staging  # Usa um workflow reutilizável
    with:
      python-version: ${{ inputs.python-version }}  # Passa a versão do Python definida como input
      active: ${{ inputs.python-pylint-active }}  # Define se o job estará ativo ou não

  python-ci-pytest:  # Job para execução de testes com PyTest
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/python-ci-pytest.yaml@staging
    with:
      python-version: ${{ inputs.python-version }}
      active: ${{ inputs.python-pytest-active }}

  common-ci-sonarqube:  # Job para integração com o SonarQube
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/common-ci-sonarqube.yaml@staging
    with:
      active: ${{ inputs.common-sonarqube-active }}  # Define se o SonarQube será executado
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # Passa os segredos (credenciais)
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  common-ci-dtrack:  # Job para análise de dependências com Dependency Track
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/common-ci-dtrack.yaml@staging
    with:
      active: ${{ inputs.common-dtrack-active }}  # Define se o Dependency Track será executado

  common-gitleaks:  # Job de verificação de leaks (vazamentos) com Gitleaks
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/common-gitleaks.yaml@staging
    with:
       active: true  # Gitleaks sempre ativo
    secrets:
      GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # Licença para uso do Gitleaks

  common-bandit:  # Job de análise de segurança com Bandit
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/common-bandit.yaml@staging
    with:
      active: true  # Bandit sempre ativo
      
  check-optl:  # Job para verificar OpenTelemetry
    name: Python CI
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/check-optl.yaml@staging
    with:
      active: ${{ inputs.check-optl-active }}  # Define se o check-optl será executado
````

1. **Jobs reutilizáveis**: Cada job usa um workflow externo reutilizável. Assim, o código fica modular e reaproveitável.
1. **Parâmetros de entrada**: As entradas configuráveis permitem flexibilidade ao ativar/desativar jobs ou definir versões de Python.
1. **SonarQube e Gitleaks**: Usam segredos para armazenar de forma segura as credenciais e tokens sensíveis.
1. **Jobs independentes**: Cada análise ou verificação (Pylint, PyTest, SonarQube, Gitleaks, Bandit, etc.) é tratada separadamente, podendo ser ativada/desativada conforme as necessidades do projeto.

se for preciso adicionar alguma outra ferramenta, precisa criar um novo yaml na raiz do repositorio e dentro do python-ci.yaml fazer o apontamento do mesmo, lembrando de usar o  workflow_call:  para permitir que o workflow seja chamado por outros workflows



# YAML que chama o CI dentro dos repositorios


**Todos os repositorios precisam ter o python-ci.yaml dentro do 
.github/workflows/python-ci.yaml**

````yaml

name: python-ci  
run-name: python-ci  # Nome da execução do workflow

on:  # Define quando o workflow será acionado
  pull_request:  # Aciona o workflow em pull requests
    branches: master  # Será executado em pull requests para o branch master

jobs:  # Definição dos jobs do workflow

  ci:  
    name: CI  # Nome exibido para o job
    uses: pulsus-mobi/pulsus-github-actions/.github/workflows/python-ci.yaml@master  # Usa o workflow reutilizável 'python-ci' da branch 'master'
    with:  # Parâmetros passados para o workflow reutilizável
      python-version: "3.8"  # Define a versão do Python a ser utilizada como 3.8
    secrets:  # Segredos utilizados no job
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}  # Token de autenticação do SonarQube (armazenado de forma segura)
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}  # URL do servidor SonarQube (armazenado de forma segura)
      GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # Licença do Gitleaks (armazenada de forma segura)
````
