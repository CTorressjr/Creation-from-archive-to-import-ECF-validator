# 📄 ECF Retificadora — Gerador em Lote

> Geração automatizada de arquivos ECF Retificadora no layout SPED da Receita Federal, processando múltiplas SCPs (Sociedades em Conta de Participação) a partir de uma planilha CSV com os dados tributários anuais.

[![Status](https://img.shields.io/badge/status-produção-green)]()
[![Layout](https://img.shields.io/badge/layout-SPED%20ECF-blue)]()
[![Tributos](https://img.shields.io/badge/tributos-IRPJ%20%2F%20CSLL-orange)]()
[![Regime](https://img.shields.io/badge/regime-Lucro%20Presumido-yellow)]()

---

## 🎯 O Problema

Escritórios de contabilidade que administram grupos com múltiplas SCPs precisam entregar uma ECF Retificadora por empresa — cada arquivo com estrutura de blocos SPED específica, contagem exata de linhas por registro, cálculo dos 4 trimestres de IRPJ/CSLL e aplicação condicional de blocos zerados vs. com valores.

Montar isso manualmente para dezenas de SCPs por ciclo de retificação é lento, repetitivo e altamente sujeito a erro de contagem de registros 9900 — que invalida o arquivo na validação da RFB.

---

## 💡 A Solução

Script Python que lê uma planilha CSV com os dados de todas as SCPs e gera automaticamente um arquivo `.txt` por empresa, com:

- Estrutura completa de blocos SPED (0000, P001, P030, P200, P300, P400, P500, P990, Q, Y, 9900, 9999)
- Aplicação condicional: trimestre com valor → bloco completo | trimestre zerado → bloco mínimo
- Contagem automática de linhas totais (`|9999|N|`) e por tipo de registro (`|9900|PXXX|N|`)
- Geração em lote: uma execução processa todas as SCPs do CSV

---

## 🏗️ Estrutura do Layout ECF Gerado

```
Arquivo por SCP:
│
├── |0000| → Identificação da ECF (com número de recibo para retificação)
├── |0001| → Abertura do bloco 0
├── |0010| → Parâmetros do SPED
├── |0020| → Dados cadastrais
├── |0030| → Endereço
├── |0930| → Responsável e procurador
├── |0990| → Encerramento bloco 0
│
├── |P001| → Abertura bloco P (Lucro Presumido)
│   ├── |P030| × 4 → Período de apuração trimestral
│   ├── |P200| → Discriminação receita bruta
│   ├── |P300| → IRPJ — base de cálculo e imposto apurado
│   ├── |P400| → CSLL — base de cálculo
│   └── |P500| → CSLL — contribuição apurada
├── |P990| → Encerramento bloco P (com contagem de linhas)
│
├── |Q001/Q100/Q990| → Bloco Q (Operações com exterior — zerado)
│
├── |Y001| → Abertura bloco Y
│   ├── |Y600| → Composição societária (sócio PF + sócio PJ ostensiva)
│   ├── |Y672| → Identificação de fundos
│   └── |Y720| → Outras informações
├── |Y990| → Encerramento bloco Y
│
└── |9001/9900/9990/9999| → Bloco 9 (totalizadores e encerramento)
    ├── |9900| × N → Contagem por tipo de registro
    └── |9999| → Total geral de linhas do arquivo
```

---

## 📊 Lógica de Blocos Condicionais

Para cada trimestre, o script aplica automaticamente:

```python
# Trimestre com receita → bloco completo com P200, P300, P400, P500
if VARIAVEL_T1 != 0:
    BLOCO1 = template_valort1   # Receita + Base + IRPJ 15% + CSLL 9%

# Trimestre sem receita → bloco mínimo (zerado)
else:
    BLOCO1 = template_zerado1   # Apenas cabeçalhos obrigatórios
```

---

## 📁 Estrutura do CSV de Entrada

O arquivo CSV deve ter as seguintes colunas:

| Coluna | Descrição |
|--------|-----------|
| `recibo` | Número do recibo da ECF original (para retificação) |
| `cnpj` | CNPJ da SCP (com `1` na frente para preservar zeros à esquerda) |
| `cpf` | CPF do sócio ostensivo (com `1` na frente) |
| `nome` | Nome do sócio ostensivo PF |
| `TRI1` a `TRI4` | Número de identificação de cada trimestre |
| `1T` a `4T` | Receita bruta por trimestre |
| `BASE32%T1` a `BASE32%T4` | Base de cálculo (32% da receita) por trimestre |
| `IMP15%T1` a `IMP15%T4` | IRPJ à alíquota de 15% por trimestre |
| `IMP9%T1` a `IMP9%T4` | CSLL à alíquota de 9% por trimestre |

---

## 🔧 Como Usar

### Pré-requisitos

```bash
pip install pandas openpyxl
```

### Configuração

Edite as duas variáveis de configuração no início do script:

```python
# Caminho para o CSV com os dados das SCPs
tabela = pd.read_csv(r'caminho/para/seu/arquivo.csv', sep=';')

# Pasta onde os arquivos ECF serão gerados
DIRETORIO = r'caminho/para/pasta/de/saida'
```

### Execução

```bash
python ECF_retificadora.py
```

Resultado: um arquivo `{CNPJ}.txt` por linha do CSV, gerado na pasta configurada.

---

## ⚙️ Contagem Automática de Registros

O ponto crítico do layout ECF é a contagem exata de linhas e registros no bloco 9. O script calcula automaticamente:

```python
# Total de linhas do arquivo
CONT_LINHA = len(BLOCO_TEMP.splitlines()) + 38  # +38 do bloco final fixo

# Contagem por tipo de registro (para os |9900|)
CONT_P     = len([l for l in linhas if '|P' in l]) - 1
CONT_P200  = len([l for l in linhas if '|P200' in l])
CONT_P300  = len([l for l in linhas if '|P300' in l])
CONT_P400  = len([l for l in linhas if '|P400' in l])
CONT_P500  = len([l for l in linhas if '|P500' in l])
```

Sem essa contagem correta, o validador da RFB (PGE/Receitanet) rejeita o arquivo.

---

## 📁 Estrutura do Repositório

```
ecf-retificadora/
├── README.md
├── ECF_retificadora.py        ← Script principal
├── data/
│   └── modelo_entrada.csv     ← Modelo de CSV de entrada (dados fictícios)
└── output/
    └── .gitkeep               ← Pasta de saída (não versionada)
```

> ⚠️ A pasta `data/` com arquivos reais e a pasta `output/` devem estar no `.gitignore`.

---

## 🔗 Projetos Relacionados

Este repositório faz parte de um conjunto de ferramentas de automação fiscal:

| Projeto | Descrição |
|---------|-----------|
| [MIT-JSON](https://github.com/CTorressjr/MIT-JSON) | Gerador de JSON para o MIT (IRPJ, CSLL, PIS/COFINS com SCP) |
| [importacao-contabil-dominio](https://github.com/CTorressjr/importacao-contabil-dominio) | Gerador de lançamentos contábeis para Domínio Web ERP |

---

## 👤 Autor

**Carlos Torres** — AI Solutions Architect  
[LinkedIn](https://www.linkedin.com/in/carlostorressjr/) · [GitHub](https://github.com/CTorressjr)

> Desenvolvido para automação de obrigações acessórias de grupos econômicos com múltiplas SCPs em regime de Lucro Presumido.

---

## ⚠️ Aviso Legal

Os arquivos gerados devem ser validados com o PGE (Programa Gerador da ECF) antes da transmissão. O autor não se responsabiliza por transmissões incorretas. Dados reais de empresas nunca devem ser versionados no repositório.
