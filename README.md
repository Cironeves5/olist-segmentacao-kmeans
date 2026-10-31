# Segmentação de Lojistas Parceiros da Olist

**Clusterização com k-means a partir da lógica RFM, da operação logística e da satisfação dos compradores**

<!-- Imagem de capa opcional -->

# 1. Problema de Negócio

A Olist conecta pequenos e médios lojistas aos grandes marketplaces brasileiros. O lojista anuncia pela Olist Store, mantém o próprio estoque e despacha o produto diretamente ao comprador por meio dos parceiros logísticos da plataforma. Para quem compra, o desempenho do lojista se confunde com a experiência na Olist, e um atraso ou um produto fora do esperado pesa sobre a marca da plataforma.

A base de lojistas é bastante heterogênea. Alguns concentram centenas de pedidos, a maioria vendeu poucas vezes, e uma parte parou de vender meses antes do fim do período. Tratar todos da mesma forma, seja em ações de retenção, de cobrança por prazo ou de incentivo, desperdiça esforço com quem não precisa e deixa de atuar onde há risco.

O projeto segmenta os lojistas em grupos de comportamento interpretáveis, para que cada grupo possa orientar uma ação específica. A análise busca responder a três perguntas.

- **P1.** Quais perfis de lojistas aparecem quando recência, frequência e valor são combinados a indicadores de operação e de satisfação?
- **P2.** Quais grupos concentram o volume e a receita da plataforma, e quais estão perdendo atividade?
- **P3.** Os problemas de operação, como atraso na postagem e nota baixa, se concentram em grupos específicos ou estão espalhados pela base?

# 2. Ferramentas Utilizadas

**Linguagem e ambiente.** Python no Google Colab.

**Bibliotecas.**
- **pandas e numpy** para a integração das nove tabelas e a construção da tabela de lojistas.
- **matplotlib e seaborn** para a análise exploratória e a descrição dos grupos.
- **scikit-learn** para a padronização (`StandardScaler`), o k-means, a silhueta e o índice de Rand ajustado (ARI).
- **phik** para medir a associação entre variáveis numéricas e categóricas numa mesma matriz.
- **kmodes** para o k-prototypes, usado como modelo de comparação.

**Métodos.** Segmentação RFM, suavização de taxas para baixo volume, transformação logarítmica, poda de variáveis por correlação de Pearson e de Spearman, e avaliação de agrupamentos por silhueta, cotovelo, tamanho do menor grupo e estabilidade entre sementes.

# 3. Descrição dos Dados

**Fonte.** [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), com cerca de 100 mil pedidos feitos entre 2016 e 2018, distribuídos em nove tabelas relacionais (pedidos, itens, pagamentos, avaliações, produtos, categorias, clientes, vendedores e geolocalização).

**Por que lojistas e não compradores.** A segmentação RFM normalmente parte dos clientes, mas na Olist a grande maioria dos compradores aparece com uma única compra, o que esvazia a recência e a frequência como medidas de relacionamento. Os lojistas vendem de forma recorrente ao longo do período, e é essa recorrência que sustenta o RFM.

**Base de modelagem.** 2.965 lojistas, com uma linha por lojista, construída a partir de 96.470 pedidos entregues.

**Variáveis do modelo final**

| Variável | Bloco | Descrição |
|---|---|---|
| `recencia_dias` | RFM (R) | Dias entre a última entrega do lojista e a data de referência (18/10/2018) |
| `n_pedidos` | RFM (F) | Quantidade de pedidos entregues |
| `receita_total` | RFM (M) | Soma do valor dos itens vendidos, sem frete |
| `ticket_medio` | Valor | Receita total dividida pelo número de pedidos |
| `dias_de_casa` | Tempo | Dias entre a primeira venda e a data de referência |
| `nota_media` | Satisfação | Média das notas recebidas nos pedidos avaliados |
| `pct_pesado` | Catálogo | Proporção dos itens vendidos na faixa pesada (acima de 1.300 g) |
| `taxa_atraso_postagem_suav` | Operação | Proporção suavizada de pedidos postados após o limite |
| `taxa_atraso_entrega_suav` | Operação | Proporção suavizada de pedidos entregues após a data estimada |

**Variáveis descritivas, fora do cálculo da distância.** `taxa_cancelamento`, `frete_total`, `frete_pct`, `pct_interestadual`, `rotulo_valor`, `rotulo_peso` e `vendedor_uf`.

# 4. Desenvolvimento da Solução

## 4.1 Recorte da base e unidade de análise

A base de modelagem foi restrita aos pedidos entregues, já que pedidos cancelados, em trânsito ou travados em etapas intermediárias não têm data de entrega nem avaliação, e o k-means não aceita valores ausentes. A taxa de cancelamento, única informação perdida com o recorte, foi calculada antes e incorporada depois como variável descritiva.

Os dados foram agregados da granularidade de item para a de pedido e lojista, e depois para uma linha por lojista. Os lojistas sem nenhuma entrega concluída e os 5 sem nenhuma avaliação ficaram fora do agrupamento e são tratados como segmentos à parte.

## 4.2 Engenharia de variáveis

**Prazos.** O atraso de postagem compara o despacho com o limite acordado e é o único trecho do ciclo inteiramente sob controle do lojista, sem interferência de distância, transportadora ou peso. O atraso de entrega compara a chegada com a data estimada, que já nasce ajustada à rota e ao produto. Ambos entram como proporção de pedidos atrasados, porque a média de dias seria dominada pelas antecipações.

**Pseudocategoria de catálogo.** A categoria do produto está ausente para 56 lojistas, e por isso não entrou no modelo. No lugar dela, cada produto recebeu uma faixa de preço (cortes em R$ 49,90 e R$ 119,99) e uma faixa de peso (cortes em 400 g e 1.300 g), definidas pelos tercis. Os rótulos de cada lojista consideram apenas os itens entre o seu próprio p10 e p90, o que evita que um item atípico redefina o perfil do catálogo.

## 4.3 Tratamento do baixo volume

Cerca de um terço dos lojistas tem até 3 pedidos, e com um único pedido a taxa de atraso só pode valer 0% ou 100%. Para evitar que o acaso defina o grupo, as duas taxas de atraso foram suavizadas em direção à média da plataforma.

```
taxa_suav = (taxa × n_pedidos + m × taxa_base) / (n_pedidos + m),  com m = 7 (mediana de pedidos por lojista)
```

Com isso, um lojista com um único pedido atrasado passa de 100% para cerca de 19%, e um lojista com centenas de pedidos praticamente não muda. As taxas suavizadas entram apenas no algoritmo, e as originais descrevem os grupos.

## 4.4 Transformação e seleção de variáveis

- `n_pedidos`, `receita_total`, `ticket_medio` e `recencia_dias` passaram por `log1p` para reduzir a assimetria (a receita, por exemplo, foi de 8,88 para 0,10 de assimetria).
- Todas as variáveis foram padronizadas com `StandardScaler`.
- Pares com correlação acima de 0,7 em Pearson ou em Spearman foram podados. `frete_total` e `frete_pct` saíram por redundância com o volume e com o ticket.
- A matriz PhiK foi usada para verificar a associação entre os rótulos categóricos e as variáveis numéricas.

## 4.5 Modelagem iterativa

| Modelo | Mudança | Resultado |
|---|---|---|
| 1 | k-means com taxa de cancelamento e sem receita | 7 grupos, um deles formado apenas por cancelamentos em lojistas de baixo volume, e volume e valor com pouco peso na separação |
| 2 | Inclusão de `receita_total` e retirada da taxa de cancelamento | 6 grupos, com as três dimensões do RFM representadas |
| 3 (final) | Retirada de `pct_interestadual` e `rotulo_valor`, de baixa contribuição | 6 grupos mais compactos, com ARI de 0,88 em relação ao modelo 2 |
| k-prototypes | Inclusão de `rotulo_valor`, `rotulo_peso` e UF como variáveis categóricas | Melhor silhueta em k = 4, e com k = 6 o ARI em relação ao modelo final foi de 0,72 |

## 4.6 Escolha do número de grupos

| k | Silhueta | Variância explicada | Menor grupo |
|---|---|---|---|
| 4 | 0,20 | 38,1% | 10,7% |
| 5 | 0,20 | 44,4% | 8,0% |
| **6** | **0,21** | **48,6%** | **5,5%** |
| 7 | 0,20 | 51,6% | 5,3% |

O k = 6 reúne a maior silhueta da série, um ganho de variância explicada que ainda compensa e um menor grupo com tamanho suficiente para interpretação.

![Cotovelo e silhueta](<img width="867" height="701" alt="image" src="https://github.com/user-attachments/assets/5f433b9a-b450-4fc2-bbbe-8b6214fa4d40" />
)

## 4.7 Estabilidade e escolha do modelo

O modelo final foi reajustado com 10 sementes diferentes, e o ARI em relação à solução original ficou entre 0,98 e 1,00. O k-prototypes também se mostrou estável, mas as variáveis categóricas não acrescentaram estrutura aos grupos, e na sua melhor divisão o grupo de produto caro e pesado se diluiu entre os demais. O k-means com seis grupos foi mantido por ser mais simples e por diferenciar o valor por transação e os tipos de problema de operação, que são os recortes mais úteis para a ação.

# 5. Resultados

| Cluster | Perfil | Lojistas | % lojistas | % receita | Pedidos (mediana) | Ticket (mediana) | Recência (mediana) | Nota média |
|---|---|---|---|---|---|---|---|---|
| 3 | Grandes e ativos | 736 | 24,8% | 77,1% | 54 | R$ 117 | 55 dias | 4,19 |
| 1 | Produto pesado e caro | 551 | 18,6% | 11,1% | 4 | R$ 251 | 111 dias | 4,37 |
| 2 | Atraso na postagem | 163 | 5,5% | 6,0% | 12 | R$ 133 | 146 dias | 3,65 |
| 0 | Novos e ativos | 787 | 26,5% | 3,3% | 4 | R$ 73 | 63 dias | 4,47 |
| 4 | Inativos | 543 | 18,3% | 1,9% | 3 | R$ 69 | 380 dias | 4,40 |
| 5 | Nota baixa | 185 | 6,2% | 0,7% | 2 | R$ 100 | 183 dias | 2,13 |

![Perfil dos grupos](imagens/perfil_clusters.png)

![Distribuição das variáveis por grupo](imagens/boxplots_clusters.png)

# 6. Conclusões

**P1. Perfis.** Os seis grupos se separam por quatro eixos, que são o volume, o tempo de atividade, o tipo de catálogo e a qualidade da operação. O cluster 1 é definido pelo catálogo, com 87% dos itens na faixa pesada e o maior ticket da base, enquanto os clusters 2 e 5 são definidos pela satisfação e pela operação.

**P2. Concentração e atividade.** A receita é fortemente concentrada, com 24,8% dos lojistas (cluster 3) respondendo por 77,1% do faturamento. O cluster 0 é o maior grupo em número de lojistas e responde por apenas 3,3% da receita, enquanto o cluster 4 reúne 18,3% da base sem vendas há mais de um ano.

**P3. Problemas de operação.** Os problemas se concentram em dois grupos pequenos, com causas distintas. No cluster 2, formado por lojistas antigos, a nota baixa acompanha a maior taxa de atraso na postagem. No cluster 5, a nota cai para 2,13 com atraso bem menor, o que aponta para produto, descrição ou atendimento, e não para logística.

**Ações sugeridas por grupo**

| Cluster | Ação |
|---|---|
| 3 · Grandes e ativos | Retenção e monitoramento de recência como alerta de esfriamento |
| 1 · Produto pesado e caro | Negociação de frete e prazos específicos para carga pesada |
| 2 · Atraso na postagem | Cobrança de SLA de postagem e acompanhamento operacional |
| 0 · Novos e ativos | Programa de desenvolvimento para aumentar volume e ticket |
| 4 · Inativos | Campanha de reativação, priorizando quem tinha maior histórico |
| 5 · Nota baixa | Auditoria de catálogo, de anúncios e de atendimento |

# 7. Limitações e Próximos Passos

**Limitações.**
- O baixo volume de muitos lojistas deixa instáveis a nota média e o ticket médio. A suavização foi aplicada apenas às taxas de atraso.
- A silhueta de 0,21 indica sobreposição entre os grupos, esperada em dados de comportamento, em que os lojistas variam de forma contínua.
- A nota de um pedido atendido por mais de um lojista é atribuída igualmente a todos, porque não é possível saber a quem o comprador se referia.

**Próximos passos.**
- Cruzar os grupos com a UF e a categoria de produto dos lojistas.
- Modelar a nota dos compradores por regressão, para identificar o que mais pesa nas avaliações baixas.

# 8. Como Reproduzir

1. Abra `Clusterizacao_Olist_Lojistas.ipynb` no Google Colab.
2. Execute as células na ordem. A primeira baixa a base diretamente do Kaggle via `kagglehub`.

# 9. Licença

- **Dados.** CC BY-NC-SA 4.0, conforme a publicação original da Olist no Kaggle. Os dados não são versionados neste repositório.
- **Código.** MIT.

# Autor

**Ciro Neves** · [LinkedIn](https://www.linkedin.com/in/SEU-PERFIL) · [GitHub](https://github.com/Cironeves5)
