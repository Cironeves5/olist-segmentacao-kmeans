# Segmentação de vendedores da Olist

Este projeto divide os vendedores da Olist em grupos de perfil parecido, usando o algoritmo **k-means**. A ideia é simples. Quando a empresa trata todos os lojistas do mesmo jeito, ela gasta esforço onde não precisa e deixa de agir onde deveria. Com os grupos separados, cada perfil pode receber uma ação própria, seja de retenção, de cobrança por prazo ou de incentivo.

## A Olist e os dados

A Olist conecta pequenos e médios lojistas aos grandes marketplaces do Brasil. O lojista vende pela plataforma e envia o produto direto para o comprador.

A base usada é pública e reúne cerca de **100 mil pedidos feitos entre 2016 e 2018**, divididos em nove tabelas (pedidos, itens, pagamentos, avaliações, produtos, categorias, clientes, vendedores e localização).

## Por que vendedores e não compradores

A segmentação RFM costuma ser feita com clientes. Na Olist, porém, quase todos os compradores aparecem com uma única compra, e assim perguntas como "há quanto tempo comprou?" e "quantas vezes comprou?" dizem pouco sobre eles. Os vendedores vendem várias vezes ao longo do período, e é essa repetição que dá sentido ao RFM.

## O que é RFM

O RFM é uma forma clássica de separar clientes a partir de três medidas. Aqui ela foi aplicada aos vendedores.

| Letra | Significa | Pergunta que responde |
|---|---|---|
| **R** | Recência | Há quantos dias o vendedor fez a última venda? |
| **F** | Frequência | Quantos pedidos ele vendeu? |
| **M** | Valor monetário | Quanto ele faturou? |

Além do RFM, o modelo também olha para o prazo de envio, a nota dos compradores e o tipo de produto vendido.

## As perguntas do projeto

1. Que perfis de vendedores aparecem quando juntamos volume, valor, atividade e satisfação?
2. Quais grupos trazem a maior parte das vendas e quais estão deixando de vender?
3. Os problemas, como atraso no envio e nota baixa, ficam concentrados em alguns grupos ou espalhados pela base?

## Como foi feito

1. **Recorte da base.** Ficaram só os pedidos entregues, porque os outros não têm data de entrega nem avaliação.
2. **Uma linha por vendedor.** As informações de pedidos e itens foram resumidas numa tabela com um vendedor por linha, totalizando **2.965 vendedores**.
3. **Criação dos indicadores.** Para cada vendedor foram calculados o volume, o faturamento, o tempo de casa, a nota média, a taxa de atraso e a proporção de produtos pesados.
4. **Cuidado com quem vendeu pouco.** Um vendedor com um único pedido atrasado ficaria com 100% de atraso. Para evitar esse exagero, as taxas de quem tem poucos pedidos foram aproximadas da média da plataforma.
5. **Mesma escala para todos.** Variáveis com valores muito altos, como receita e número de pedidos, passaram por logaritmo, e depois todas foram padronizadas.
6. **Escolha do número de grupos.** Foram testados de 2 a 10 grupos. O número final, **6**, saiu da silhueta, do cotovelo e do tamanho do menor grupo.
7. **Teste de estabilidade.** O modelo foi rodado com sementes diferentes e os grupos se repetiram quase iguais. Também foi comparado com o k-prototypes, que aceita variáveis categóricas, e o k-means continuou como a melhor opção.

## As variáveis do modelo final

| Variável | O que mede |
|---|---|
| `recencia_dias` | Dias desde a última venda entregue |
| `n_pedidos` | Quantidade de pedidos entregues |
| `receita_total` | Quanto o vendedor faturou, sem o frete |
| `ticket_medio` | Quanto ele fatura, em média, por pedido |
| `dias_de_casa` | Há quantos dias ele vende na plataforma |
| `nota_media` | Média das notas dadas pelos compradores |
| `pct_pesado` | Parte dos itens vendidos que são pesados (acima de 1,3 kg) |
| `taxa_atraso_postagem` | Parte dos pedidos enviados depois do prazo |
| `taxa_atraso_entrega` | Parte dos pedidos que chegaram depois da data prometida |

## Os 6 grupos encontrados

| Grupo | Quem são | % dos vendedores | % da receita |
|---|---|---|---|
| 3 | **Grandes e ativos.** Vendem muito (54 pedidos na mediana) e seguem ativos | 24,8% | 77,1% |
| 1 | **Produto pesado e caro.** Ticket de R$ 251 na mediana e 87% dos itens pesados | 18,6% | 11,1% |
| 2 | **Atraso no envio.** Vendedores antigos, com a maior taxa de atraso na postagem e nota 3,65 | 5,5% | 6,0% |
| 0 | **Novos e ativos.** Pouco tempo de casa, poucos pedidos e boa nota | 26,5% | 3,3% |
| 4 | **Inativos.** Estão há mais de um ano sem vender | 18,3% | 1,9% |
| 5 | **Nota baixa.** Poucos pedidos e nota média de 2,13 | 6,2% | 0,7% |

## O que os grupos mostram

- **A receita está muito concentrada.** Um em cada quatro vendedores responde por mais de três quartos do faturamento.
- **O maior grupo é o que menos fatura por vendedor.** Os novos somam mais de um quarto da base e trazem só 3% da receita, o que faz deles o principal alvo de ações de crescimento.
- **Os problemas se concentram em dois grupos pequenos, com causas diferentes.** No grupo 2, a nota cai junto com o atraso no envio. No grupo 5, o atraso é bem menor, o que sugere que a insatisfação vem do produto ou do atendimento.

## Limitações

- Muitos vendedores têm poucos pedidos, e isso deixa instáveis medidas como a nota média e o ticket médio.
- Os grupos se sobrepõem em parte (silhueta de 0,21). Isso é comum em dados de comportamento, em que as pessoas variam aos poucos e sem fronteiras bem marcadas.
- Quando um pedido tem mais de um vendedor, a mesma nota é atribuída a todos eles, porque não há como saber a quem o comprador se referia.

## Próximos passos

- Cruzar os grupos com o estado e a categoria de produto dos vendedores.
- Usar regressão para entender o que mais pesa nas notas baixas.

## Como rodar

1. Abra o notebook `Clusterizacao_Olist_Vendedores.ipynb` no [Google Colab](https://colab.research.google.com/).
2. Rode as células na ordem. A primeira delas baixa os dados direto do Kaggle pelo `kagglehub`, então não é preciso baixar nada à mão.

**Bibliotecas usadas.** pandas, numpy, matplotlib, seaborn, scikit-learn, phik e kmodes.

## Dados e licença

- **Dados.** [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), sob licença CC BY-NC-SA 4.0. Os dados não ficam salvos neste repositório.
- **Código.** Licença MIT.

## Autor

**Ciro Neves**
[LinkedIn](https://www.linkedin.com/in/SEU-PERFIL) · [GitHub](https://github.com/Cironeves5)
