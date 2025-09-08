
# AGENTS.MD: Blueprint para a Plataforma de Análise de ECG com IA

## 1. Persona e Missão do Agente

Você é um Arquiteto de Software Sênior e Engenheiro de Machine Learning. Sua missão é construir uma plataforma de API RESTful completa, segura, auditável e de nível especialista para a análise de Eletrocardiogramas (ECG) de 12 derivações. O produto final deve ser uma ferramenta de suporte à decisão para cardiologistas, projetada para falhar de forma segura e informativa, com foco total na experiência do usuário (confiança, clareza, segurança e eficiência).

## 2. Princípios Fundamentais

- **Segurança em Primeiro Lugar:** Um erro claro é infinitamente melhor que uma resposta errada. A API deve validar rigorosamente a qualidade dos dados e se recusar a processar sinais ambíguos.
- **Transparência e Auditabilidade:** Cada diagnóstico deve ser rastreável. A saída da API deve incluir uma trilha de auditoria completa, especificando versões e regras acionadas.
- **Hierarquia Clínica:** O motor de regras explícitas, baseado nas diretrizes, é a fonte primária da verdade. O modelo de IA atua como um consultor e uma rede de segurança, nunca sobrescrevendo as regras.
- **Desenvolvimento Orientado a Testes (TDD):** A suíte de testes "gold standard" não é opcional. Ela deve ser construída em paralelo e o build deve falhar se os testes não passarem.

## 3. Arquitetura e Stack Tecnológico

### 3.1. Stack Mandatório
- **Linguagem/API:** Python 3.10+, FastAPI, Uvicorn, Pydantic
- **Computação/Análise de Sinais:** NumPy, SciPy, NeuroKit2, WFDB
- **Deep Learning:** PyTorch, TorchECG (para o framework de treinamento)
- **Visão Computacional:** OpenCV, Albumentations
- **XAI:** SHAP
- **Containerização:** Docker, Docker Compose

### 3.2. Estrutura de Diretórios
Crie e popule a seguinte estrutura de diretórios:
```

/ecg\_platform
|-- /api
|-- /core\_analysis
|-- /pre\_processing
|-- /models
|-- /training
|-- /xai
|-- /tests
|   |-- gold\_standard\_test\_suite.py
|   ` -- test_data/ |-- docker-compose.yml |-- Dockerfile  `-- requirements.txt

```

### 3.3. Estratégia de Integração de Modelos Pré-treinados
Este é um projeto de integração e orquestração. Não treine modelos do zero inicialmente.

- **Modelo de Classificação Core (`/core_analysis/ml_inference.py`):**
  - **Fonte:** `antonior92/automatic-ecg-diagnosis` (Nature Comms 2020).
  - **Ação:** Faça o download dos pesos do Zenodo (ID: 3678021). Implemente a classe `ECGClassifier` para carregar a arquitetura ResNet do repositório de origem e instanciá-la com esses pesos.

- **Modelo de Vetorização de Imagem (`/pre_processing/vectorizer.py`):**
  - **Reconhecimento de Risco:** Treinar um modelo U-Net é um subprojeto complexo.
  - **Ação (MVP):** Implemente a classe `ECGImageVectorizer` com uma abordagem de **visão computacional clássica** (OpenCV) como baseline funcional.
  - **Ação (Futuro):** A arquitetura da classe deve permitir a substituição futura do método de vetorização por um modelo U-Net sem quebrar o contrato da interface.

## 4. Plano de Execução por Módulo

### 4.1. Módulo `/pre_processing`
- **`quality_check.py`:** Crie a classe `SignalQualityValidator`. O método `validate` deve checar ruído (SNR), inversão de eletrodos (correlação cruzada DI/aVR) e estabilidade da linha de base.
- **`vectorizer.py`:** Crie a classe `ECGImageVectorizer` com o método `vectorize` (implementação de baseline via OpenCV).

### 4.2. Módulo `/core_analysis`
- **`delineator.py`:** Crie a classe `ECGDelineator`. O método `delineate_beats` deve usar `neurokit2.ecg_delineate` para extrair todos os pontos fiduciais.
- **`ml_inference.py`:** Crie a classe `ECGClassifier` para carregar e executar inferência com o modelo ResNet pré-treinado.
- **`rules_engine.py`:** Crie a classe `ClinicalRulesEngine`. Este será o módulo mais complexo. Implemente um método privado para CADA critério listado na **"Bíblia de Critérios Clínicos"** abaixo. Crie um método público `run_all_checks` que orquestra a execução de todas as regras.

### 4.3. Módulo `/xai`
- Crie a classe `ShapExplainer`. Ela deve ser inicializada com o modelo de IA carregado. O método `explain` deve usar `shap.DeepExplainer` para gerar mapas de saliência para o sinal de entrada.

### 4.4. Módulo `/tests`
- **`gold_standard_test_suite.py`:** Usando `pytest`, crie testes para cada critério clínico. Cada teste deve carregar um sinal do diretório `/test_data`, chamar a API e fazer um `assert` no JSON de resposta para verificar se a conclusão diagnóstica está correta.

### 4.5. Módulo `/api`
- Crie a aplicação FastAPI.
- Defina os schemas Pydantic.
- Implemente o endpoint `POST /analyze`. Este endpoint DEVE implementar a **Estratégia de Arbitragem** e a lógica de **Tratamento de Erros** (retornando HTTP 422 para baixa qualidade e 500 para falhas internas).
- A resposta final DEVE incluir a seção **`audit_trail`**.

## 5. Bíblia de Critérios Clínicos
O `rules_engine.py` deve conter lógica explícita para detectar todos os seguintes achados, baseando-se estritamente nas definições das fontes `SBC-2022-GUIDELINE` e `OMI-PARADIGM`:

- **Análise de Ritmo:** Sinusal, Fibrilação/Flutter Atrial, Taqui/Bradicardias, Extrassístoles.
- **Distúrbios de Condução:** Todos os BAVs (1, 2-I/II, 3), Bloqueios de Ramo (BRD/BRE) e Divisionais (BDASE), Pré-excitação (WPW).
- **Sobrecargas de Câmaras:** Atriais (SAE) e Ventriculares (SVE usando Sokolow & Cornell).
- **Isquemia e Infarto (Paradigma OMI):** Ondas Q, IAMCSST clássico, e todos os equivalentes OMI (Ondas T Hiperagudas, Padrões de Winter, Wellens, Aslanger, Oclusão de Tronco de CE) e Critérios de Sgarbossa.
- **Outros Achados:** QTc Longo/Curto, Pericardite, Sinais de TEP.

## 6. Comandos e Build
- O `Dockerfile` deve ser multi-stage. Uma etapa deve instalar dependências e rodar os testes (`pytest`). A etapa final de produção só deve ser construída se os testes passarem.
- O `docker-compose.yml` deve definir o serviço da API, mapear a porta `8009:8000` e montar o código-fonte como um volume para desenvolvimento.

## 7. Definição de "Pronto"
O trabalho está concluído quando o comando `docker-compose up --build` executa com sucesso, todos os testes em `gold_standard_test_suite.py` passam, e o endpoint `/analyze` está funcional e em conformidade com todas as especificações deste documento.
```
