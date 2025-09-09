---
layout: post
title: 'O Básico bem feito: Organizando base de dados para uma empresa de SST'
date: 2025-09-09 14:17 -0300
---

## Introdução e o problema:

Dados são um dos ativos mais valiosos hoje em dia. Prospecção de clientes, levantamento de informações sobre funcionários, gestão e relacionamento com clientes, dados fiscais, informações e previsões financeiras e por aí vai. Tudo isso é uma mina de ouro, um ativo para a empresa - se estruturados.

Uma empresa de Saúde e Segurança do tabalho (SST) procurou-me para encontrar soluções que ajudassem a estruturar uma série de dados, solucionando alguns problemas que eles tinham e melhorando produtos deles já existentes. Embora a empresa não seja enorme, atende um número de clientes grande o suficiente pra ocupar operacionalmente todos seus funcionários, mas não grande o suficiente pra ter um setor apenas voltado à T.I. e processos. Tudo ficava meio no manual, no feeling e dependendo da urgência.

Inclusive, a relação com clientes.

Como a empresa oferece treinamentos e certificados das Normas Regulamentadoras (NR), esses certificados tem um período de valiade (estabelecido conforme a NR). Após o vencimento, a empresa de SST poderia oferecer um curso de reciclagem para esses clientes que já tiveram serviços prestasdos por ela. O problema? Esses documentos não eram estruturados. Informações como nome do aluno, CPF, data de emissão, tipo de curso, ficando soltas em arquivos de diferentes formatos (PDFs, PPTX, docx e imagens).

Manualmente, consultar ou validar um certificado era um processo lento e sujeito a erros. A empresa precisava de uma forma de transformar esse volume de documentos em uma base de dados organizada e pesquisável.

## A Solução: Construindo um Pipeline de Extração Automatizado
Para resolver isso, desenvolvi um pipeline de automação que processa os certificados em lote e extrai as informações de forma estruturada. A ideia era criar um fluxo robusto que pudesse ser executado repetidamente sem reprocessar arquivos já lidos.

O processo funciona em quatro etapas principais:
1. OCR para transformar arquivos em texto;
2. LLM para extrair campos específicos e geração de linhas do CSV;
3. Normalização e padronização dos dados do CSV;
4. Apresentação dos resultados para o cliente.

#### 1. Leitura e OCR (Reconhecimento Óptico de Caracteres):

Ferramenta: Docling.

Criei um script em Python que varre as pastas de certificados. Como essa é uma etapa custosa computacionalmente, o script verifica se um arquivo já foi processado e, caso tenha sido, o ignora. Para cada novo certificado, ele realiza o OCR, convertendo a imagem do documento em um arquivo de texto puro (.md).

Com o script rodando sem maiores complicações, pode-se deixar de um dia pro outro.

#### 2. Extração Inteligente com LLM (Large Language Model):

Ferramenta: API do Gemini pro e [smollm](https://huggingface.co/blog/smollm3)

O texto extraído pelo OCR, embora legível, ainda é um bloco de texto sem estrutura. Se fosse uma estrutura fixa, usaria regex + fuzzy matching pra economizar processamento. Como não é o caso, para cada arquivo, fiz uma chamada para um modelo de linguagem, concatenando-o a um prompt e instruindo o modelo a identificar e retornar as seguintes informações:

* Nome completo e CPF do participante.
* Data de emissão do certificado.
* Nome do curso/treinamento e a NR correspondente.
* Prazo de validade do curso, conforme as tabelas da empresa.

Testei dois modelos de linguagem diferentes pra tarefa.

Gemini
: O primeiro, sendo  o "Gemini 2.5 Flash-Lite"[^gemini-models]. Embora tenha um limite de 15 requisições por minuto (que controlei usando uma "sliding window" nas requisições), o plano gratuíto oferece 1000 requisições por dia, que deve ser o suficiente.

SmoLLM3
: Desde o anúncio do smollm na versão 3, fiquei curiosíssimo pra testar ele nesse tipo de tarefa. Rodar ele numa VM ou no meu PC economizaria prompts do gemini pro. Infelizmente, mostrou inconsistente. Algumas vezes gerava as linhas com boa qualidade, as vezes, retornava com um texto além das linhas csv, mesmo sendo instruído explicitamente para não o fazer.

#### 3. Normalização e Estruturação:

Por mais refinado que um prompt seja, usar LLMs significa estar sujeito a alucinações. Mas agora são erros perceptíveis, estruturados e em menor número: Linhas faltando, sobrando ou repetidas, formato de datas ou strings incorretos. Isso é fácil ajustar com expressões regulares ou no olho.
Como as linhas são geradas a partir do .md, também consigo usá-lo como informação de origem desses dados, me dando a possibilidade de verificar o arquivo original e ver se há algum problema a ser descoberto.

#### 4. Saída e Geração de Logs:

Resultado Final: Cada certificado processado gera uma nova linha em um arquivo CSV. Esse arquivo se torna, na prática, o banco de dados pesquisável que a empresa precisava.

Controle de Qualidade: Para garantir a confiabilidade, também implementei um sistema de logs. Ele registra qual arquivo de origem corresponde a cada linha do CSV, facilitando a conferência e a validação dos dados extraídos.

### Concluíndo:

Como todas essas etapas envolvem lotes de arquivos e processamentos, é importante ter um tratamento de erros relativamente seguro. Isso envolve, além de pensar em erros mais comuns (problemas com pastas e nomes de arquivos), até em alguns menos previsíveis (um curso no meio de certificados :P). Ter um fallback e logar esses erros foi fundamental pra, ao longo de 3 ou 4 iterações, garantir a qualidade da entrega dos dados.


[^gemini-rate-limits]: [Planos e restrições do gemini](https://ai.google.dev/gemini-api/docs/rate-limits#free-tier)
[^gemini-models]: [Modelos disponíveis do gemini](https://ai.google.dev/gemini-api/docs/models)

