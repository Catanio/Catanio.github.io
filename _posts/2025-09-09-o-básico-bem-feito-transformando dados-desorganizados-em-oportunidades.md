---
layout: post
title: 'O Básico bem feito: Transformando dados desorganizados em oportunidades'
date: 2025-09-09 14:17 -0300
categories: [Projetos, Ciência de Dados]
tags: [Automação de Processos, Ciência de Dados, Inteligência Artificial, LLM, OCR, Python, Gestão de Dados, Transformação Digital, SST (Saúde e Segurança do Trabalho)]
---

## Introdução e o problema:

Dados são hoje um dos ativos mais valiosos das empresas. Eles aparecem em diferentes frentes: prospecção de clientes, gestão de funcionários, relacionamento comercial, notas fiscais, previsões financeiras, entre outros. Tudo isso é mina de ouro e ativo crucial para a empresa - se devidamente estruturado.

Uma empresa de Saúde e Segurança do trabalho (SST) procurou-me para encontrar soluções que ajudassem a organizar uma série de dados, solucionando alguns problemas que eles tinham e melhorando outros produtos já existentes. Embora não seja uma empresa grande, o volume de clientes é suficiente para ocupar toda a equipe. Ainda assim, não tem porte para manter um setor dedicado de T.I. e processos. Tudo ficava em processos manuais, dependia da intuição da equipe e variava conforme a urgência.

Inclusive, a relação com clientes.

Como a empresa oferece treinamentos e certificados das Normas Regulamentadoras (NR), esses certificados tem um período de validade (estabelecido conforme a NR). Após o vencimento, a empresa de SST poderia oferecer um curso de reciclagem para esses clientes que já tiveram serviços prestados por ela. O problema? Esses documentos não eram estruturados. Informações como nome do aluno, CPF, data de emissão, tipo de curso, ficando soltas em arquivos de diferentes formatos (PDFs, PPTX, docx e imagens).

Manualmente, consultar um único certificado podia levar minutos, e validar a vigência de centenas deles seria uma tarefa inviável. **Além do risco de erros, a empresa perderia a oportunidade de contatar proativamente os clientes para renovações.**

## A Solução: Construindo um Pipeline de Extração Automatizado
Para resolver isso, desenvolvi um pipeline de automação que processa os certificados em lote e extrai as informações de forma estruturada. A ideia era criar um fluxo robusto que pudesse ser executado repetidamente sem reprocessar arquivos já lidos.

O processo funciona em quatro etapas principais:
1. OCR para transformar arquivos em texto;
2. LLM para extrair campos específicos e geração de linhas do CSV;
3. Normalização e padronização dos dados do CSV;
4. Apresentação dos resultados para o cliente.

![Desktop View](/assets/img/ocr-llm-processing-diagram.png)

#### 1. Leitura e OCR (Reconhecimento Óptico de Caracteres):

Ferramenta: Docling[^docling].

Criei um script em Python que varre as pastas de certificados. Como essa é uma etapa custosa computacionalmente, o script verifica se um arquivo já foi processado e, caso tenha sido, ignora-o. Para cada novo certificado, ele realiza o OCR, convertendo a imagem do documento em um arquivo de texto puro (.md).

Com o script rodando sem maiores complicações, é possível deixá-lo processando durante a noite.

#### 2. Extração Inteligente com LLM (Large Language Model):

Ferramenta: API do Gemini pro e SmolLM3[^smollm]

O texto extraído pelo OCR, embora legível, ainda é um bloco de texto sem estrutura. Se fosse uma estrutura fixa, usaria _regex + fuzzy matching_ para economizar processamento. Como não é o caso, para cada arquivo, fiz uma chamada para uma LLM, concatenando o texto a um prompt e instruindo o modelo a identificar e retornar as seguintes informações:

* Nome completo e CPF do participante.
* Data de emissão do certificado.
* Nome do curso/treinamento e a NR correspondente.
* Prazo de validade do curso, conforme as tabelas da empresa.


Testei dois modelos de linguagem diferentes pra tarefa:

Gemini
: O primeiro, sendo  o "Gemini 2.5 Flash-Lite"[^gemini-models]. Embora tenha um limite de 15 requisições por minuto[^gemini-rate-limits] (que controlei usando uma "sliding window" nas requisições), o plano gratuito oferece 1000 requisições por dia, que deve ser o suficiente.

SmolLM3[^smollm]
: Com o artigo da versão 3 e resultados promissores, fiquei curioso pra testá-lo nesse tipo de tarefa. Rodar ele numa VM ou no meu PC economizaria chamadas à API do gemini Pro. Infelizmente, os resultados foram inconsistentes. Algumas vezes os dados eram recuperados com boa qualidade, às vezes, a LLM retornava-os um texto além das linhas em csv, mesmo sendo instruído explicitamente para não o fazer.

![Desktop View](/assets/img/smollm3-comparison.png){: .w-50 }
_desempenho do SmolLM3 comparado a outros LLM_



#### 3. Normalização e Estruturação:

Por mais refinado que um prompt seja, usar LLMs significa estar sujeito a alucinações. Mas agora são erros perceptíveis, estruturados e em menor número: Linhas faltando, sobrando ou repetidas, formato de datas ou strings incorretos. Isso é fácil ajustar com expressões regulares ou manualmente.
Como as linhas são geradas a partir do .md, também consigo usá-lo como informação de origem desses dados, dando-me a possibilidade de verificar o arquivo original e se há algum problema a ser descoberto.

#### 4. Base csv:

Cada arquivo de certificados processado gerou uma ou mais linha em um arquivo CSV, com arquivo origem (para conferência), dados do trabalhador e do cecrtificado. Esse arquivo se torna, na prática, conjunto de dados estruturado e pesquisável que a empresa precisava. Pode ser importado em diferentes arquivos de planílias, banco de dados ou mesmo algum sistema de ERP ou CRM.

### Considerações:

Como as etapas envolvem lotes de arquivos e longos períodos de processamento, é importante ter um tratamento de erros relativamente seguro. Isso envolve pensar preemptivamente nos erros possíveis, além dos mais comuns (problemas com pastas e nomes de arquivos), até em alguns menos previsíveis (um pdf do curso em meio de certificados!). Ter um fallback e logar esses erros foi fundamental pra, ao longo de 3 ou 4 iterações, garantir a qualidade da entrega dos dados.


## Conclusão:

O que começou como uma massa de arquivos desorganizados em múltiplos formatos foi transformado em uma base de dados centralizada, pesquisável e acionável. A solução eliminou horas de trabalho manual e criou uma nova frente de negócios para a empresa.

Agora, o time comercial pode facilmente filtrar todos os certificados que irão expirar no próximo mês e contatar os clientes para oferecer cursos de reciclagem, transformando um processo operacional custoso em um motor de receita recorrente.

Este projeto é um exemplo breve mas potente de como a aplicação de ferramentas de automação e IA, com planejamento cuidadoso, tratamento de erros e foco na qualidade dos dados, pode gerar um impacto significativo nos resultados de uma empresa.

[^gemini-rate-limits]: [Planos e restrições do gemini](https://ai.google.dev/gemini-api/docs/rate-limits#free-tier)
[^gemini-models]: [Modelos disponíveis do gemini](https://ai.google.dev/gemini-api/docs/models)
[^docling]: [github do docling](https://github.com/docling-project/docling)
[^smollm]: [HF do smollm](https://huggingface.co/blog/smollm3)
[^lost-in-the-middle]: ["lost in the middle": how LLMs use long contexts](https://arxiv.org/pdf/2307.03172)
