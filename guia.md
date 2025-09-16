# Guia do AdvogaSUS
O presente documento tem por objetivo auxiliar novos desenvolvedores do projeto AdvogaSUS a entenderem o propósito e a estrutura do sistema de software que estarão desenvolvendo.

Uma das principais mensagens que eu desejo transmitir a qualquer desenvolvedor que venha a ter contato com esse sistema é que, em sua essência, o projeto
AdvogaSUS é bastante simples e pode ser concluído rapidamente, SE for desenvolvido de maneira organizada por uma equipe bem coordenada. Não há motivos para temer a tarefa em mãos.

## Propósito:
O SUS (Sistema Único de Saude) nem sempre possui capacidade, insumos ou infraestrutura para prestar todos os procedimentos médicos demandados pela população. Nesses casos, o estado
paga para que essas operações sejam realizadas por um estabelecimento de saúde da rede privada. O cliente que solicitou o desenvolvimento do AdvogaSUS é um advogado que atende
a esses hospitais particulares que defendem a tese de que o valor pago a eles em função da realização de um conjunto de procedimentos é menor do que aquele que DEVERIA ter sido pago. Portanto, esses hospitais privados contratantes do nosso cliente entram na justiça solicitando o pagamento do que ainda falta pagar (VALOR QUE DEVERIA SER PAGO)-(VALOR EFETIVAMENTE PAGO).

O objetivo final do sistema AdvogaSUS é portanto:
Ser capaz de calcular o valor que foi pago a um hospital em uma certa janela temporal bem como o valor que DEVERIA ter sido pago nessa mesma janela de tempo.
Além disso, é preciso calcular a diferença entre os dois valores para que se obtenha valor que falta pagar e aplicar atualização monetária sobre os resultados calculados.
Por fim, o sistema deve gerar um laudo em pdf com os resultados compilados.

## O valor que foi pago:
Para definir o valor pago pelo estado ao hospital em uma dada janela de tempo, é preciso ter os registros de todos os procedimentos realizados no período. O SUS disponibiliaza essas informações ao público
por meio de um sistema chamado dataSUS. Esse repositório de dados públicos pode ser acessado por meio de uma interface web com o usuário ou com um FTP (file transfer protocol).
O link de acesso para o FTP é "ftp://ftp.datasus.gov.br/". 
- SUGESTÃO: Abrir o ftp em um gerenciador de arquivos

Dentro da estrutura de arquivos do servidor FTP, os dados relevantes para a definição dos procedimentos realizados por um hospital encontram-se
nos seguintes paths: "/dissemin/publicos/SIASUS" e "/dissemin/publicos/SIHSUS". Esses são os caminhos que levam para os registros históricos de procedimentos do SIA (sistema de informações ambulatoriais) e
SIH (sistema de informações hospitalares). Dentro dessas pastas, um enorme número de tabelas contendo registros de procedimentos realizados pelo sistema de saúde brasileiro está divido por mês e estado.

O script de processamento do AdvogaSUS tem como uma de suas atribuições fazer o download dos dados relevantes para o cálculo requisitado, converter os arquivos do seu formato nativo (dbc) para um formato mais
simples (csv), filtrar os dados referentes ao hospital alvo da pesquisa, e, por fim somar o valor pago por cada um dos procedimentos registrados. Em um tópico posterior será abordado a disposição dos dados relevantes
nas tabelas do dataSUS

## O que deveria ter sido pago:
O sistema deve ser capaz de definir qual valor deve ser pago por procedimento. Para tanto, existem três métodos disponíveis. O método específico a ser utilizdo é definido por um
parâmtro denominado "metodo" passado ao script de processamento. Os três métodos são:
 - IVR
 - TUNEP
 - BEST

### IVR
O método IVR prescreve que o valor que DEVERIA ter sido pago por um dado procedimento é exatamente 1.5 vezes o valor efetivamente pago pelo procedimento. Portando faltaria pagar
sempre 0.5x vexes valor origial, considerando que o valor original já foi pago (1.0x + 0.5x).

### TUNEP
O método TUNEP se baseia em uma tabela estática (nunca vai mudar) de mesmo nome que associa um valor a um procedimento. Contudo, existem algumas considerações importantes a respeito dessa tabela. A primeira
observação é a de que existem mais de um código de identificação de procedimento médico no sistema de saúde brasileiro. No AdvogaSUS dois códigos são de relevância
para o desenvolvedor. O primeiro é o código de oito dígitos usado na tabela TUNEP que chamarei de "código TUNEP" doravante. Esse código é diferente daquele presente nas tabelas do
SIA (sistema de informações ambulatoriais) e SIH (sistema de informações hospitalares) do dataSUS das quais retiramos os registros dos procedimentos realizados por um hospital para
o sus. A este segundo tipo de código, de 9 dígitos, darei o nome de "código SUS". O desafio de estabelecer o valor de um procedimento, portanto, se resume a estabelecer uma relação entre
esses dois tipos de código.
- **Ex (fictício):**

Tabela TUNEP prescreve em uma de suas linhas que:
```
EXAME DE SANGUE, de código 12345678 deve custar 20R$
```
Em uma das linhas de uma das tabelas do SIA há o registro:
```
Em DD/MM/AAAA, o hospital privado X realizou um procedimento de código 987654321 para o sus.
```
Se tivermos o conhecimento de que o código 12345678 da TUNEP corresponde ao código SUS 987654321, poderíamos afirmar que o valor que deveria ter sido pago pelo procedimento X é de 20R$.
A dificuldade de aplicação da tabela TUNEP reside justamente no fato de que estabelecer a relação entre esses dois códigos não é trivial pelos seguintes motivos:
- Nem sempre um código SUS possui um equivalente código TUNEP.
- Muitas vezes, um único código SUS possui mais de um correspondente na tabela TUNEP (um mesmo código SUS pode representar uma biópsia, ou amputação na tabela TUNEP)
- Também é possível que um código TUNEP pode corresponder a mais de um código SUS (a respeito desse tópico eu não absoluta certeza, mas creio que seja irrelevante para os propósitos do AdvogaSUS)
- Um mesmo código SUS pode representar procedimentos diferentes a depender proveniênia dele (SIA ou do SIH).

#### Como contornar o problema:
Seguindo as orientações do cliente o que se pode fazer para contornar o problema é:
- Caso um código SUS não possua um código TUNEP associado, estabelece-se o IVR como o valor que deveria ter sido pago pelo procedimento.
- Caso um código SUS possua mais de um código TUNEP associado, por conservadorismo, escolhe-se o código TUNEP de menor valor como o que deveria ter sido pago.

### BEST
O método BEST se parece muito com o método TUNEP. A única diferença está no fato de que, por procedimento realizado, deve-se
escolher sempre o maior valor entre aqueles obtidos aplicando os outros dois métodos anteriores (IVR e TUNEP). Assim, se, aplicando o método IVR e o TUNEP, obtem-se 20R$ e 50R$ respectivamente,
o valor a ser que deveria ter sido pago é de 50R$.

## A disposição dos dados no dataSUS: 
Dentro do FTP do dataSUS, nas pastas indicadas na seção "O valor que foi pago", os dados que registram os procedimentos realizdos no sistema de saúde brasileiro estão dispostos em alguns tipos de tabelas tabelas.
Os dois tipos de tabela relevantes para o projeto são:
- as tabelas presentes nos arquivos do SIA com prefixo "PA"
- as tabelas presentes nos arquivos do SIH com prefixo "SP".
Abaixo estão descritas os elementos relevantes da estrutura dos dois tipos de tabela. Vale notar que essa estrutura só é válida para os arquivos mais recentes (posteriores a 2008, se não me falha a memória):

### Estrutura dos arquivos PA
Os as tabelas do SIA com o prefixo PA possuem as seguintes colunas: 
PA_CODUNI,PA_GESTAO,PA_CONDIC,PA_UFMUN,PA_REGCT,PA_INCOUT,PA_INCURG,PA_TPUPS,PA_TIPPRE,PA_MN_IND,PA_CNPJCPF,PA_CNPJMNT,PA_CNPJ_CC,PA_MVM,PA_CMP,PA_PROC_ID,PA_TPFIN,PA_SUBFIN,PA_NIVCPL,PA_DOCORIG,PA_AUTORIZ,PA_CNSMED,PA_CBOCOD,PA_MOTSAI,PA_OBITO,PA_ENCERR,PA_PERMAN,PA_ALTA,PA_TRANSF,PA_CIDPRI,PA_CIDSEC,PA_CIDCAS,PA_CATEND,PA_IDADE,IDADEMIN,IDADEMAX,PA_FLIDADE,PA_SEXO,PA_RACACOR,PA_MUNPCN,PA_QTDPRO,PA_QTDAPR,PA_VALPRO,PA_VALAPR,PA_UFDIF,PA_MNDIF,PA_DIF_VAL,NU_VPA_TOT,NU_PA_TOT,PA_INDICA,PA_CODOCO,PA_FLQT,PA_FLER,PA_ETNIA,PA_VL_CF,PA_VL_CL,PA_VL_INC,PA_SRV_C,PA_INE,PA_NAT_JUR

Dessas colunas, as que as de maior relevância são:
- **PA_CODUNI**: esse é a coluna que contém o código identificador do hospital em questão, o cnes ou "código nacional de estabelacimento de saúde"
- **PA_VALAPR:**: indica o valor total aprovado e pago por aquela linha da tabela, representativa de um procedimento ou de mais de um procedimento de mesmo tipo.
- **PA_PROC_ID**: Esse é o código identificador de procedimento no SIA. Esse campo é relevante para a aplicação do método TUNEP, como foi descutido em um capítulo anterior.


### Estrutura dos arquivos SP
Os as tabelas do SIH com o prefixo SP possuem as seguintes colunas: 
SP_GESTOR,SP_UF,SP_AA,SP_MM,SP_CNES,SP_NAIH,SP_PROCREA,SP_DTINTER,SP_DTSAIDA,SP_NUM_PR,SP_TIPO,SP_CPFCGC,SP_ATOPROF,SP_TP_ATO,SP_QTD_ATO,SP_PTSP,SP_NF,SP_VALATO,SP_M_HOSP,SP_M_PAC,SP_DES_HOS,SP_DES_PAC,SP_COMPLEX,SP_FINANC,SP_CO_FAEC,SP_PF_CBO,SP_PF_DOC,SP_PJ_DOC,IN_TP_VAL,SEQUENCIA,REMESSA,SERV_CLA,SP_CIDPRI,SP_CIDSEC,SP_QT_PROC,SP_U_AIH

Dessas colunas, as que as de maior relevância são:
- ****: esse é a coluna que contém o código identificador do hospital em questão, o cnes ou "código nacional de estabelacimento de saúde"
- **SP_VALATO**: indica o valor total aprovado e pago por aquela linha da tabela, representativa de um procedimento ou de mais de um procedimento de mesmo tipo.
- **SP_ATOPROF:**: Esse é o código identificador de procedimento no SIA. Esse campo é relevante para a aplicação do método TUNEP, como foi descutido em um capítulo anterior.


## Estrutura do script de processamento 
O programa que realiza o processamento dos dados do sus com o propósito de gerar o laudo e os arquivos csv requsitados está quase inteiramente contido em um único arquivo, "main.py".
Dentro desse arquivo a maior parte das linhas de código enconta-se dividida em classes que desempenham funções específicas dentro do programa. Abaixo encontra-se uma breve explicação
de cada uma dessas classes.

### ProjPaths
Essa classe é responsável por definir, criar, e gerênciar as pastas que o programa irá utilizar para armazenar os dados relevantes à sua execução. A principal função dentro dessa classe é a função init. Esta função realiza as seguintes operações:
- Define quais são os endereços dos diretórios do programa relativos ao arquivo principal ("main.py"). 
- Cria os diretórios que ainda não foram criados
- Esvazia os diretórios, caso eles já tivessem sido criados e preenchidos em execuções anteriores do programa (evita conflitos de arquivo)

### ProjParams
Armazena os parâmetros passados ao programa de forma que eles estejam disponíveis às outras classes através de atributos estáticos com nomes descritivos como:
```
CNES: str = "0000000"
STATE: str = "AC"
SYSTEM:str = "SIA"
...
NOME_FANTASIA: str = "Nome Fantasia"
NUMERO_PROCESSO: str = "Número Processo"
```

### InterestRate:
Essa classe disponibiliza métodos para que se possa aplicar correção monetária no programa. Os índices de correção aplicados são:
- O IPCAE até dezembro de 2022 
- A SELIC a partir dessa data

Uma Observação importante a respeito da aplicação da correção monetária no programa é que esta deve ser aplicada de forma simples, em comformidade com aquilo que prescreve o manual de cálculos da Justiça Federal.

As funções internas a classe são:

#### **load_selic():**
Puxa os dados históricos da SELIC por meio da API do Banco Central. Além disso, essa função salva esses dados em um arquivo para que, caso a API do Banco Central acabe caindo (o que acontece com alguma freqência) o programa possa puxar esses dados de uma execução anterior.

#### **cumulative_selic():**
É a função responsável por aplicar a taxa SELIC dado um mês de início e de fim.

#### **rate_until_01_2022():**
É a função responsável por aplicar o IPCAE da data de início especificada até o mês dezembro de 2021.

#### **complete_rate_split():**
Essa função combina as duas funções anteriores para, dados os meses de início e de fim da aplicação da correção monetária, fornecer uma tupla contendo, respectivamente, os seguintes índices em uma tupla:

(a correção decorrente do IPCAE, a correção decorrente da SELIC, os dois índices combinados em única taxa)

#### **complete_rate():**
Essa função é exatamente igual a função "complete_rate_split()", contudo, nessa versão da função, apenas a taxa combinada (IPCAE * SELIC) é retornada.
