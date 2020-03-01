+++
title = "Melhorando o desempenho do índice na biblioteca storm"
description = "Neste post descrevo como foi procurar e corrigir o gargalo de desempenho do indice de chave primária na biblioteca storm."
date = 2020-02-29T10:14:00Z
author = "Anderson Bargas"
tags = [
    "boltdb",
    "golang",
    "storm",
    "rainstorm",
    "query",
    "database",
]
categories = [
    "golang",
    "database",
]
+++

### Introdução

Nas últimas semanas estive bastante envolvido num projeto pessoal que utiliza o [boltdb](https://github.com/etcd-io/bbolt "Link para o repositório no Github"), uma biblioteca fantástica para criação de banco de dados embutido.

O [boltdb](https://github.com/etcd-io/bbolt "Link para o repositório no Github") nos permite criar um banco de dados do tipo chave-valor, além de nos proporcionar uma organização através de buckets. Pense em buckets como se fossem diretórios em um sistema de arquivos. Podemos ter um bucket dentro do outro. A única regra a ser respeitada é que, dentro de um mesmo bucket, as chaves devem ter valores únicos.

Uma das vantagens em se utilizar o boltdb é sua API simples, no entanto, para alguns casos de uso, precisamos também de uma biblioteca para cuidar dos índices.

Vou dar um exemplo:

Imagine que temos um arquivo CSV com CEPs do Brasil, algo parecido com isso:

(coloquei em uma tabela para facilitar a leitura)

| CEP      | UF | Cidade    | Bairo  | Logradouro             |
|:-------- |:-- |:--------- |:------ |:---------------------- |
| 01015080 | SP | São Paulo | Centro | Rua Fernão Sales       |
| 01015090 | SP | São Paulo | Centro | Viaduto Diario Popular |
| 01015100 | SP | São Paulo | Centro | Avenida Exterior       |
| 01016000 | SP | São Paulo | Centro | Rua Venceslau Brás     |
| 01016010 | SP | São Paulo | Centro | Rua Irmã Simpliciana   |
| 01016020 | SP | São Paulo | Centro | Rua Santa Teresa       |

Sabemos que o CEP é um número único, por isto, podemos usá-lo como chave.\
Legal, já sabemos que a chave será o CEP e, no valor, iremos guardar o objeto completo.
Para isto precisamos serializar o objeto pois o boltdb trabalha com [slices](https://tour.golang.org/moretypes/7 "Para caso queira saber mais sobre Slices") de byte.\
Para a serialização temos vários Codecs disponíveis. Um dos mais utilizados é o [encoding/json](https://pkg.go.dev/encoding/json?tab=doc "Link para a documentação") que nos permite codificar e decodificar objetos Go para JSON.

Se pensarmos em nossa lista de CEPS, codificados em JSON e salvas no boltdb, teríamos algo assim:

| Chave    | Valor                                                                                                     |
|:-------- |:--------------------------------------------------------------------------------------------------------- |
| 01015080 | {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Rua Fernão Sales"}       |
| 01015090 | {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Viaduto Diario Popular"} |
| 01015100 | {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Avenida Exterior"}       |
| 01016000 | {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Rua Venceslau Brás"}     |
| 01016010 | {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Rua Irmã Simpliciana"}   |
| 01016020 | {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Rua Santa Teresa"}       |

Deste modo, usando o próprio CEP como índice, podemos recuperá-lo facilmente:

{{< highlight go >}}
CEPaSerRecuperado := "01015080"
db.View(func(tx *bolt.Tx) error {
	bucket := tx.Bucket([]byte("BucketDeCEPs"))
	valor := bucket.Get([]byte(CEPaSerRecuperado))
	fmt.Printf("JSON do CEP recuperado: %s\n", valor)
	// {"CEP":"01015080","UF":"SP","Cidade":"São Paulo","Bairro":"Centro","Logradouro":"Rua Fernão Sales"}
	return nil
})
{{< /highlight >}}

Funciona! Mas agora surge outra dúvida. E se precisarmos recuperar todos os CEPs de determinada cidade?
Isso iria dar uma baita trabalho! Teríamos que passar por todos os registros, decodificando cada valor (de JSON para objeto) para verificamos se a cidade coincide com a pesquisa.

Além de trabalhoso, a busca levaria um tempo enorme, considerando que a base de CEPs da qual falei no início, possui quase 1 milhão de registros.\
É aqui que entra a biblioteca [storm](https://github.com/asdine/storm "Link para o repositório no Github"), a qual possibilita a criação de índices que aceleram nossa busca em campos que não são chave primária.

### Indexando através da biblioteca "storm"

Com a biblioteca [storm](https://github.com/asdine/storm "Link para o repositório no Github"), podemos utilizar tags para sinalizar que precisamos indexar alguns campos.
Vejamos como ficaria nosso objeto CEP com estas tags:

{{< highlight go >}}
// CEP contém o Código de Endereçamento Postal + informações do endereço
type CEP struct {
	CEP        string `storm:"id"`
	UF         string
	Cidade     string `storm:"index"`
	Bairro     string
	Logradouro string
}
{{< /highlight >}}

No código acima vimos que, além do índice na cidade, tivemos que informar a chave primária (id).

A biblioteca [storm](https://github.com/asdine/storm "Link para o repositório no Github") obriga que seja informado qual campo se refere à chave primária.\
Temos dois modos para isso:

  * através da tag (como no exemplo);
  * dando o nome de ID para o campo.

Neste projeto em que estive trabalhando, a ideia inicial era apenas indexar a chave primária para fazer consultas diretamente a um CEP.

Sendo assim vamos ver um trecho de código que nos mostra esta indexação:

(para simplificar o exemplo, ao invés de carregarmos os CEPs de um arquivo, vamos simplesmente iterar de 1 a 1 milhão)

{{< highlight go >}}
package main

import (
	"fmt"

	storm "github.com/asdine/storm/v3"
)

// CEP contém o Código de Endereçamento Postal + informações do endereço
type CEP struct {
	CEP        string `storm:"id"`
	UF         string
	Cidade     string
	Bairro     string
	Logradouro string
}

func main() {
	// inicia o banco de dados
	db, err := storm.Open("banco.db")
	if err != nil {
		panic(err)
	}

	// garante o fechamento do banco de dados
	defer func() {
		if err := db.Close(); err != nil {
			panic(err)
		}
	}()

	// inicia uma transação
	tx, err := db.Begin(true)
	if err != nil {
		panic(err)
	}

	// garante o rollback da transação
	defer func() {
		if tx == nil {
			return
		}
		if err := tx.Rollback(); err != nil {
			panic(err)
		}
	}()

	// loop
	for codigoCEP := 1; codigoCEP <= 2000; codigoCEP++ {
		cep := &CEP{
			CEP:        fmt.Sprintf("%08d", codigoCEP),
			UF:         "SP",
			Cidade:     "São Paulo",
			Bairro:     "Centro",
			Logradouro: "Avenida Teste",
		}

		// salva o CEP
		if err := tx.Save(cep); err != nil {
			panic(err)
		}

		// commit a cada mil registros
		if codigoCEP%1000 == 0 {
			if err := tx.Commit(); err != nil {
				panic(err)
			}
			// ponto para indicar, visualmente, um commit
			fmt.Printf(".")
			tx, err = db.Begin(true)
			if err != nil {
				panic(err)
			}
		}
	}

	// conta quantos CEPs estão no banco de dados
	qtde, _ := db.Count(&CEP{})
	fmt.Printf("\n\nQuantidade de CEPs inseridos: %d\n", qtde)
}
{{< /highlight >}}

Código compilado e executado. Vamos ver o output:

![asciicast](/tty/primary-index-storm.gif)

Os commits estão sendo feitos a cada mil inserções.\
Cada ponto indica um commit.\
Visualmente fica fácil de perceber que quanto mais se insere, mais lento fica.

Vamos dar uma olhada no que está acontecendo.


### Analisando com a ajuda do CPU Profiler

Precisamos entender o motivo da inserção ficar cada vez mais lenta.
Para isto vamos utilizar o profiler de uso de CPU.
O Go já tem incorporado este profiler, portanto, basta adicionarmos um pequeno trecho no início do nosso código:

{{< highlight go >}}
...

func main() {
	// cria um arquivo para receber os dados de profiling
	arquivoCPUprof, err := os.Create("./cpu.prof")
	if err != nil {
		panic(err)
	}

	// inicia o profiling
	pprof.StartCPUProfile(arquivoCPUprof)

	// garante a parada do profiler
	defer pprof.StopCPUProfile()

	// inicia o banco de dados
	...
{{< /highlight >}}

Além disso, vamos diminuir a quantidade de CEPs a serem gravados para que não tenhamos de esperar muito tempo:

{{< highlight go >}}
// loop
for codigoCEP := 1; codigoCEP <= 30000; codigoCEP++ {
	...
{{< /highlight >}}
Feito isto, basta executar o programa novamente e deixá-lo rodando até que termine, para que o profiler possa salvar informações suficientes para nossa análise.

{{< highlight zsh >}}
➜  rm banco.db 
➜  go build .
➜   ./indice 
..............................

Quantidade de CEPs inseridos: 30000
➜  ls cpu.prof
cpu.prof
{{< /highlight >}}

Pronto! Agora temos o nosso arquivo gerado pelo profiler.\
Vamos ver o que ele nos diz.

O Go já vem com uma ferramenta bastante útil para analisarmos o resultado do profiler.\
Para acessá-la, vamos executar o comando, passando o arquivo gerado:

{{< highlight zsh >}}
➜  go tool pprof cpu.prof  
File: indice
Type: cpu
Time: Feb 29, 2020 at 2:55pm (-03)
Duration: 8.51s, Total samples = 8.33s (97.90%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof)
{{< /highlight >}}

Quando ativamos o profiler de CPU, o programa faz pequenas paradas (em torno de 100 vezes por segundo). Neste momento, ele registra uma amostra que consiste na verificação do que está sendo executado na pilha de execução.

Estas amostra são gravadas no arquivo de saída do profiler, o qual podemos abrir com a ajuda da ferramenta "pprof".

Esta ferramenta possui vários comandos. Um bem útil para o nosso caso é o "topN", onde N é um número.\
Vamos rodar o comando "top10" para visualizarmos as dez principais execuções:

{{< highlight zsh >}}
(pprof) top10
Showing nodes accounting for 7280ms, 87.39% of 8330ms total
Dropped 108 nodes (cum <= 41.65ms)
Showing top 10 nodes out of 32
      flat  flat%   sum%        cum   cum%
    1750ms 21.01% 21.01%     4830ms 57.98%  go.etcd.io/bbolt.(*Cursor).next
    1130ms 13.57% 34.57%     1830ms 21.97%  go.etcd.io/bbolt.(*Cursor).keyValue
    1020ms 12.24% 46.82%     6150ms 73.83%  go.etcd.io/bbolt.(*Cursor).Next
     750ms  9.00% 55.82%      750ms  9.00%  memeqbody
     620ms  7.44% 63.27%     1130ms 13.57%  go.etcd.io/bbolt.(*Cursor).first
     510ms  6.12% 69.39%      510ms  6.12%  go.etcd.io/bbolt.(*elemRef).count
     420ms  5.04% 74.43%      420ms  5.04%  go.etcd.io/bbolt.(*elemRef).isLeaf
     410ms  4.92% 79.35%     1340ms 16.09%  bytes.Equal
     370ms  4.44% 83.79%     8000ms 96.04%  github.com/asdine/storm/v3/index.(*UniqueIndex).RemoveID
     300ms  3.60% 87.39%      300ms  3.60%  go.etcd.io/bbolt._assert
(pprof)
{{< /highlight >}}

É importante notar que o profiler não contabiliza quantas vezes um determinado método foi invocado.
O que ele faz é, com base na quantidade de vezes em que determinado método foi visto nas amostras, calcular aproximadamente quanto tempo um método ficou em execução.
É este o motivo de termos os resultados em tempo ou percentualmente.

Nosso programa faz somente uma coisa, salvar registros, repetidamente.\
Olhando para os itens do top10, vemos algo um tanto estranho.
Temos um método chamado "UniqueIndex.RemoveID".
Além disso, também temos chamadas ao "Cursor.Next", o que nos sinaliza que temos algum tipo de loop percorrendo registros do banco.

Se olharmos para o nosso programa parece não haver um motivo óbvio para a execução de nenhum destes métodos pois:
  * não estamos lendo dados que faria o cursor percorrer registros;
  * não estamos removendo ID;
  * e nem trabalhando com índice único.

Numa rápida conferida ao código da biblioteca storm, descobrimos que índices de chave primária são tratados do mesmo modo que índices únicos:

{{< highlight go "linenos=table,hl_lines=4 6,linenostart=113">}}
...
for _, tag := range tags {
			switch tag {
			case "id":
				f.IsID = true
				f.Index = tagUniqueIdx
			case tagUniqueIdx, tagIdx:
				f.Index = tag
			case tagInline:
				...
{{< /highlight >}}

Agora está claro o motivo da lentidão. Toda vez que inserimos um CEP, a biblioteca faz um loop em todos os registros para verificar se já existe algum CEP com o mesmo código (como se fosse um índice único).\
Sendo assim, se estivermos inserindo o décimo CEP, este loop irá iterar 9 vezes. Mas se estivermos inserindo o milionésimo CEP, o loop irá iterar 999.999 vezes!

### Solucionando o problema

Como vimos anteriormente, o [boltdb](https://github.com/etcd-io/bbolt "Link para o repositório no Github") nos diz que, dentro de um mesmo bucket, a chave deve ser única mas, não podemos deixar que isso nos confunda com o índice único.\
Além de única, a chave primária tem a capacidade de conseguir identificar um registro específico.\
Sendo assim, podemos usar a própria chave primária como ID (identificador), excluindo a necessidade de se criar um índice apartado.

Neste sentido, criei um fork da biblioteca e fiz os ajustes necessários. O repositório está no endereço: [github.com/AndersonBargas/rainstorm](https://github.com/AndersonBargas/rainstorm "Link para o repositório no Github")

Claro que, antes de criar o repositório tentei contato com o criador da biblioteca [storm](https://github.com/asdine/storm "Link para o repositório no Github") via [issue](https://github.com/asdine/storm/issues/222#issuecomment-586647571 "Link para a issue, no Github") existente para este problema, mas infelizmente não obtive resposta. O jeito foi criar um fork.

Ajustes feitos, hora de testar!

### Teste final

Código final, novamente com 1 milhão de iterações e importando a biblioteca rainstorm:

{{< highlight go >}}
package main

import (
	"fmt"

	"github.com/AndersonBargas/rainstorm/v4"
)

// CEP contém o Código de Endereçamento Postal + informações do endereço
type CEP struct {
	CEP        string `rainstorm:"id"`
	UF         string
	Cidade     string
	Bairro     string
	Logradouro string
}

func main() {
	// cria um arquivo para receber os dados de profiling
	// arquivoCPUprof, err := os.Create("./cpu.prof")
	// if err != nil {
	// 	panic(err)
	// }

	// inicia o profiling
	// pprof.StartCPUProfile(arquivoCPUprof)

	// garante a parada do profiler
	// defer pprof.StopCPUProfile()

	// inicia o banco de dados
	db, err := rainstorm.Open("banco.db")
	if err != nil {
		panic(err)
	}

	// garante o fechamento do banco de dados
	defer func() {
		if err := db.Close(); err != nil {
			panic(err)
		}
	}()

	// inicia uma transação
	tx, err := db.Begin(true)
	if err != nil {
		panic(err)
	}

	// garante o rollback da transação
	defer func() {
		if tx == nil {
			return
		}
		if err := tx.Rollback(); err != nil {
			panic(err)
		}
	}()

	// loop
	for codigoCEP := 1; codigoCEP <= 1000000; codigoCEP++ {
		cep := &CEP{
			CEP:        fmt.Sprintf("%08d", codigoCEP),
			UF:         "SP",
			Cidade:     "São Paulo",
			Bairro:     "Centro",
			Logradouro: "Avenida Teste",
		}

		// salva o CEP
		if err := tx.Save(cep); err != nil {
			panic(err)
		}

		// commit a cada mil registros
		if codigoCEP%1000 == 0 {
			if err := tx.Commit(); err != nil {
				panic(err)
			}
			// ponto para indicar, visualmente, um commit
			fmt.Printf(".")
			tx, err = db.Begin(true)
			if err != nil {
				panic(err)
			}
		}
	}

	// conta quantos CEPs estão no banco de dados
	qtde, _ := db.Count(&CEP{})
	fmt.Printf("\n\nQuantidade de CEPs inseridos: %d\n", qtde)
}
{{< /highlight >}}

Compilando e executando:

![asciicast](/tty/primary-index-rainstorm.gif)

Como vimos no output, 1 milhão de CEPs inseridos em 7 segundos.\
Tivemos um excelente ganho de performance.

### Próximos passos

O trabalho ainda não acabou. Após analisar os outros tipos de índices, verifiquei que ainda há oportunidades de melhorias.\
Assim que tiver novidades volto a postar por aqui.

Obrigado por ter lido e espero que tenha gostado!

## Referências

[Profiling Go Programs - The Go Blog](https://blog.golang.org/profiling-go-programs "Link para a referência")