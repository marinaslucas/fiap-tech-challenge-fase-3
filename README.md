# Relatório Técnico: Assistente Médico Virtual com Fine-tuning e LangChain

**Autores:** Joalisson dos Santos Borges | Luis Gustavo Santini | Marina Souza Lucas | Ivna Cavalcante Barros Meireles | Diego Santos

**Projeto:** Tech Challenge IADT - Fase 3

---

## Sumário Executivo

**Problema:** Um hospital universitário precisa de um assistente virtual inteligente, treinado com dados médicos, capaz de auxiliar nas condutas clínicas, responder dúvidas de médicos e sugerir procedimentos com base em protocolos internos, integrando dados reais de prontuários eletrônicos.

**Solução:** Sistema baseado em três componentes integrados:

- **Fine-tuning de LLM**: adaptação do Qwen3.5-4B ao domínio clínico com PubMedQA e MedQuAD via LoRA em bf16.
- **Base de prontuários real**: integração com MIMIC-IV Clinical Database Demo (PhysioNet), com 100 pacientes reais desidentificados do Beth Israel Deaconess Medical Center.
- **Pipeline LangGraph + ReAct**: StateGraph com 6 nós explícitos orquestrando o fluxo de decisão clínica, com AgentExecutor ReAct operando dentro do nó `gerar_parecer` para raciocínio clínico em português e function calling, com validação automática de segurança e log de auditoria JSON por sessão.

---

## 1 – Objetivo

Este projeto visa desenvolver um assistente médico virtual inteligente para um hospital universitário. O sistema será treinado com dados médicos para auxiliar nas condutas clínicas, responder dúvidas de médicos, sugerir procedimentos baseados em protocolos internos e integrar dados reais de prontuários eletrônicos. O objetivo é criar uma solução robusta, segura e explicável que possa fornecer suporte clínico preciso e rastreável.

---

## 2 – Dataset

O projeto utiliza dois tipos principais de dados:

### 2.1 – Dados para Fine-tuning do LLM

O dataset de fine-tuning é construído a partir de duas fontes públicas:

- **PubMedQA (`pqa_artificial`):** Perguntas e respostas clínicas com contexto científico, cobrindo uma ampla gama de condições com linguagem médica formal.
- **MedQuAD (`keivalya/MedQuad-MedicalQnADataset`):** Perguntas e respostas médicas frequentes que simulam triagem clínica e dúvidas do corpo clínico. Registros curtos ou incompletos são descartados para garantir a qualidade do treinamento.

### 2.2 – Base de Prontuários e Exames Reais

Integração com o **MIMIC-IV Clinical Database Demo (PhysioNet)**, que contém dados reais e desidentificados de 100 pacientes internados no Beth Israel Deaconess Medical Center. Este dataset inclui:

- Dados demográficos dos pacientes.
- Diagnósticos CID (CID-9/10).
- Medicamentos prescritos.
- Resultados de exames laboratoriais (`labevents`).
- Informações de admissões (`admissions`).

---

## 3 – Metodologia

O sistema é construído sobre três pilares principais:

### 3.1 – Fine-tuning do Qwen3.5-4B com Unsloth + LoRA

- O modelo **Qwen3.5-4B** foi escolhido por seu desempenho em português e capacidade clínica.
- O fine-tuning é realizado usando **LoRA em bf16** para preservar a qualidade clínica, otimizado com a biblioteca **Unsloth**.
- O `max_seq_length` é definido como 1024, cobrindo a maioria dos exemplos do dataset.
- O treinamento é limitado a 100 steps, visto que a redução de loss se estabiliza de forma eficiente neste ponto.

### 3.2 – Exportação para GGUF e Configuração do Ollama

- O modelo fine-tuned é exportado para o formato **GGUF** com quantização `q4_k_m`, resultando em um modelo de aproximadamente 2.5GB para inferência local.
- O **Ollama** é utilizado como servidor local para o modelo GGUF, permitindo o uso eficiente do LLM.
- O processo de exportação remove componentes de visão para compatibilidade com `llama.cpp`.

### 3.3 – Pipeline LangGraph + ReAct com Tools

- O assistente utiliza um pipeline **LangGraph + ReAct**.
- Um **StateGraph** com 6 nós explícitos orquestra o fluxo de decisão clínica:
  - `receber_consulta`: registra a entrada e detecta ID MIMIC-IV.
  - `buscar_contexto_paciente`: aciona `buscar_prontuario` e `verificar_exames`.
  - `selecionar_protocolo`: infere e aciona `consultar_protocolo` com base nos diagnósticos.
  - `gerar_parecer`: opera o **AgentExecutor ReAct**.
  - `validar_seguranca`: verifica aviso de responsabilidade e ausência de prescrição direta.
  - `registrar_auditoria`: salva o log JSON completo da consulta.
- O **AgentExecutor ReAct** opera dentro do nó `gerar_parecer`, utilizando raciocínio clínico em português e `function calling`.

**Tools disponíveis:**

- `buscar_prontuario`: consulta dados demográficos, diagnósticos CID, admissões e medicamentos do MIMIC-IV Demo.
- `verificar_exames`: consulta exames laboratoriais (`labevents`) do MIMIC-IV Demo.
- `consultar_protocolo`: consulta protocolos clínicos institucionais simulados baseados em diretrizes reais (AHA/ACC, GOLD, GINA, ACR, Surviving Sepsis Campaign).

---

## 4 – Como executar (Recomendado: Google Colab)

Para replicar e interagir com o assistente médico virtual, siga os passos abaixo em um ambiente Google Colab:

1.  **Abertura do Notebook:** Abra o notebook `Tech_Challenge_Fase3.ipynb` no [Google Colab](https://colab.research.google.com/).
2.  **Execução Sequencial:** Execute todas as células do notebook sequencialmente, da primeira à última.
    - As instalações de dependências (`1.1 – Instalação das Dependências`) serão feitas automaticamente.
    - O ambiente (`1.2 – Configuração do Ambiente`) será configurado, o Google Drive será montado e os dados do MIMIC-IV Demo serão baixados.
    - O fine-tuning do Qwen3.5-4B (`1.4 – Fine-tuning do Qwen3.5-4B com Unsloth + LoRA`) será executado, seguido da exportação do modelo para GGUF e configuração do Ollama (`1.5 – Exportação para GGUF e Configuração do Ollama`).
    - O agente médico com LangChain e LangGraph (`1.7 – Criação do Agente com LangGraph (StateGraph)`) será inicializado.
    - Os testes do assistente (`1.8 – Testes do Assistente Médico`) e a avaliação (`1.9 – Avaliação do Sistema`) serão executados.
3.  **Terminal Interativo:** A última célula (`1.10 – Terminal Interativo (Demonstração ao Vivo)`) iniciará um loop conversacional onde você poderá interagir com o assistente, consultando prontuários e protocolos.
    - Digite `sair` a qualquer momento para encerrar a sessão interativa. Os logs de auditoria serão salvos automaticamente no Google Drive.

---

## 5 – Requisitos no Ambiente Colab

Para executar o notebook e o fine-tuning do modelo, é necessário um ambiente Google Colab com as seguintes especificações:

- **GPU:** Uma GPU NVIDIA, preferencialmente uma **L4 ou A100**.
- **Suporte a bf16:** A GPU deve suportar operações em `bf16` (bfloat16) para o fine-tuning. Caso contrário, `fp16` (float16) será utilizado, o que pode impactar ligeiramente a performance ou VRAM utilizada.

---

## 6 – Estrutura do Projeto

O projeto organiza seus arquivos e saídas da seguinte forma no Google Drive:

```
/content/drive/MyDrive/Tech_Challenge_Fase3/
├── outputs/
│   ├── qwen35_medico/             # Modelo Qwen3.5-4B fine-tuned (peft adapters)
│   └── gguf/                      # Modelos GGUF exportados (q4_k_m, bf16)
├── logs/
│   ├── langgraph_YYYYMMDD_HHMMSS.json  # Logs de auditoria de cada consulta do LangGraph
│   └── sessao_YYYYMMDD_HHMMSS.json     # Logs de sessões completas do terminal interativo
├── data/
│   ├── dataset_curado.json        # Dataset combinado e curado para fine-tuning
│   └── mimic/
│       └── hosp/                  # Dados do MIMIC-IV Clinical Database Demo
│           ├── admissions.csv
│           ├── diagnoses_icd.csv
│           ├── labevents.csv
│           ├── patients.csv
│           └── prescriptions.csv
├── fluxo_langgraph.png            # Diagrama do fluxo do LangGraph
└── README.md                      # Este arquivo README
```

---

## 7 – Saídas e Artefatos Gerados

Durante a execução do notebook, os seguintes artefatos são gerados e salvos no Google Drive:

1.  **Modelo Fine-tuned (PEFT adapters):** O modelo Qwen3.5-4B adaptado ao domínio médico é salvo em `outputs/qwen35_medico/`.
2.  **Modelos GGUF:** Versões quantizadas do modelo (`q4_k_m`) são exportadas para `outputs/gguf/` para uso com Ollama.
3.  **Logs de Auditoria:** Cada interação com o assistente via LangGraph gera um log JSON detalhado em `logs/`, registrando o fluxo, status de segurança e o parecer final.
4.  **Logs de Sessão:** As interações completas do terminal interativo são salvas em arquivos JSON em `logs/`.
5.  **Diagrama do LangGraph:** Uma imagem PNG do fluxo do StateGraph é gerada e salva como `fluxo_langgraph.png`.

---

## 8 – Notas Importantes

- **Segurança e Validação:** O sistema é projetado com validação automática de segurança, garantindo que nunca prescreva diretamente e sempre cite a fonte das informações. Cada resposta inclui um aviso legal explícito: "DIAGNÓSTICO E CONDUTA FINAL SOB RESPONSABILIDADE DO MÉDICO ASSISTENTE."
- **Rastreabilidade:** O uso do LangGraph com nós explícitos e logs de auditoria detalhados garante a rastreabilidade completa de cada decisão e informação fornecida pelo assistente.
- **Fontes:**
  1.  Johnson, A. et al. (2023). _MIMIC-IV Clinical Database Demo._ PhysioNet. https://physionet.org/content/mimic-iv-demo/2.2/
  2.  Johnson, A. et al. (2023). _MIMIC-IV, a freely accessible electronic health record dataset._ Scientific Data. DOI: 10.1038/s41597-022-01899-x
  3.  Jin, Q. et al. (2019). _PubMedQA: A Dataset for Biomedical Research Question Answering._ EMNLP.
  4.  Abacha, A. & Demner-Fushman, D. (2019). _MedQuAD: Medical Question Answering Dataset._ BMC Bioinformatics.
  5.  Qwen Team. (2025). _Qwen3.5 Technical Report._ Alibaba Group. https://huggingface.co/Qwen/Qwen3.5-4B
  6.  Han, E. et al. (2024). _Unsloth: Efficient LLM Fine-tuning._ https://github.com/unslothai/unsloth
  7.  LangChain. (2024). _LangChain Documentation._ https://python.langchain.com
  8.  LangGraph. (2024). _LangGraph Documentation._ https://langchain-ai.github.io/langgraph
  9.  Ollama. (2024). _Ollama: Run LLMs locally._ https://ollama.com
