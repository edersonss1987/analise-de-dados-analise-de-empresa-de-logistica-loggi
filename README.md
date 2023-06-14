# analise-de-dados-analise-de-empresa-de-logistica-loggi
Esse é um arquivo em json, para a Analise de dados, com um conjunto de informaçoes da empresa de logística Loggi.


Esse é um projeto de analise de dados de uma grande empresa de logistica "LOGGI",  que nos forneceu seu conjunto de dados para tratamento, com o objetivo de entender quais as dificuldades enfrentados no dia-a-dia dos pontos logisticos (Hubs) da empresa, a analise é voltada para extração de algum insight, para que os hubs tenham melhor eficiência nas suas entregas, com menor tempo de espera por parte dos clientes, e maior volume de entrega de saída, por parte dos Hubs0  . No entanto estamos apenas com parte desse conjunto de dados, onde falamos das estregas de Brasilia.
   
   
   **SOBRE OS DADOS**
   
Os dados estão no formato Json e uma parte dos dados está aninhada, tendo que ser "NORMALIZADO", então usaremos o pacote Json, para tratar um bloco de código específico, também usaremos a geolocalização para enrriquecer nossos dados, extraindo o endereço completo da geolocalização e tambem vamos aproveitar para contruir insights sobre o mapa plotado.

**INSIGHTS**

A Pltagem do HeatMap, não foi o suficiente para que pudessemos extrair o maior volume de entregas realizados por distrito, por isso, tomamos outras medidas para garantir que os dados coletados nos trouxessem boas informações.

1°) Identificamos que o ditrito(df-1) percorre as menores distâncias, porém ela tem o maior volume de entregas, entre todos os outros df's, sendo assim, os consumos de combustivel se torne o menor, alcançando uma média de 4.12 litros.

2°) Já o ditrito(df-0) tem o menor volume de entregas entre todos os outros df's, mas com base nas análises realizadas, podemos indentificar que seu comsumo médio de combustivel, quando comparado com os outros dois distritos passam a ser maior, com uma média de 6.24 litros, isso leva a crer que sua frota é inferior aos outros df's

3°) Enquanto o ditrito(df-2) possuí as melhores médias, por tanto, esse modelo precisa ser tirado como referência.

**CÓDIGO**

!pip3 install geopandas -q;

import json
import time
import numpy as np
import pandas as pd
import geopandas as gdf
import seaborn as sns
import matplotlib.pyplot as plt
from geopy.geocoders import Nominatim
from geopy.extra.rate_limiter import RateLimiter
import folium
from folium.plugins import HeatMap
from haversine import haversine




!wget -q "https://raw.githubusercontent.com/edersonss1987/analise-de-dados-analise-de-empresa-de-logistica-loggi/main/deliveries.zip" -O deliveries.zip
!unzip -q deliveries.zip -d /content/

with open('/content/deliveries.json', mode='r', encoding='utf8') as file:
  data_loggi = json.load(file)

  len(data_loggi)

  # Iniciando o wrangling da estrutura, mais analise exploratória do schema;

entregas_df = pd.DataFrame(data_loggi)
entregas_df.head()

# Verificando a calda do dataframe, da mesma forma vamos olhar o rodapé do arquivo ou como conhecemos a calda do arquivo, conferir se segue o padrão do cabeçalho.
entregas_df.tail()

# Verificando a quantidade de colunas e linhas 
entregas_df.shape

# Podemos identificar que os dados iniciam na primeira linha e da um salto por vez, até a ultima linha
entregas_df.index

# Imprimindo os nomes de colunas
entregas_df.columns

# Como podemos verificar, todas as linhas estão preenchidas, não possuindo valores nulos, isso é muito bom para os proximos passos.
#verificamos também os tipos de dados, que aparentemente estão em um formato correto
entregas_df.info()

# Tipo de dado de cada coluna
entregas_df.dtypes

# Como já tinhamos visto, nunhum campo está Nulo
entregas_df.isna().any()

# Identificando se há alguma inconcistencia com o numero de linhas no arquivo e aproveitando para analisar a frequência e contagem dos dados únicos
entregas_df.select_dtypes("object").describe().transpose()

# as series "Origin" e "deliveries" vão precisar serem tratadas.
entregas_df[["origin", "deliveries"]].head()

# Tratamento da série "origen", já que ele veio em forma de dicionário..
# vamos usar o método  pd.json_normalize
origem_df = pd.json_normalize(entregas_df["origin"])
origem_df.head()

#Tratamento da série "deliveries", já que ele veio em forma de dicionário..
#vamos usar o método  .explode("deliveries")
novo_deliveries = entregas_df[["deliveries"]].explode("deliveries")
novo_deliveries.head()

#para preservar os indices, estamos usando a o metodo "pd.concat", para concatenar verticalmente os dados
# com função anonima, vamos trabalhar todas as linhas da serie 

deliveries_normalizados_df = pd.concat([
  pd.DataFrame(novo_deliveries["deliveries"].apply(lambda record: record["id"])).rename(columns={"deliveries": "id_da_entrega"}),
  pd.DataFrame(novo_deliveries["deliveries"].apply(lambda record: record["size"])).rename(columns={"deliveries": "volume_da_entrega"}),
  pd.DataFrame(novo_deliveries["deliveries"].apply(lambda record: record["point"]["lng"])).rename(columns={"deliveries": "longitude_da_entrega"}),
  pd.DataFrame(novo_deliveries["deliveries"].apply(lambda record: record["point"]["lat"])).rename(columns={"deliveries": "latitude_da_entrega"}),
], axis= 1)
deliveries_normalizados_df.head()


#Fazendo o Merge entre os dados de entregas_df e origem_df
entregas_df = pd.merge(left=entregas_df, right=origem_df, how='inner', left_index=True, right_index=True)

#Fazendo o Merge entre os dados de entregas_df e deliveries_normalizados_df
entregas_df = pd.merge(left=entregas_df, right=deliveries_normalizados_df, how='right', left_index=True, right_index=True)
entregas_df.reset_index(inplace=True, drop=True)

# Removendo as colunas que foram usadas no flatten
entregas_df = entregas_df.drop(["origin","deliveries"], axis=1) 

#Renomeando as coluna, para melhor enterpretação do usuario
entregas_df.rename(columns={"lng": "longitude_do_hub", "lat": "latitude_do_hub","name":"nome_do_hub","region":"regiao_do_hub","vehicle_capacity":"capacidade_do_veiculo"}, inplace=True)

# Verificando se existe algum dado faltante no nosso dataframe
entregas_df.isna().any()

# Olhando o cabeçalho dos dados
entregas_df.head()

# Olhando a calda dos dados
entregas_df.tail()

#Exemplo de extração de geolocalização no formato json
geolocator = Nominatim(user_agent='Ederson_Santos')
try:
    location = geolocator.reverse("-23.610186696552372, -46.928535311126396", timeout=10)
    print(json.dumps(location.raw, indent=4, ensure_ascii=False))
except AttributeError:
    print("Problema com os dados ou uma falha com o Geocode.")
except GeocoderTimedOut:
    print("Um erro de timeout ocorreu.")


    #Etapa de extração de duplicidade por região
geocoder = RateLimiter(geolocator.reverse, min_delay_seconds=1)

hub_df = entregas_df[[ "regiao_do_hub","longitude_do_hub", "latitude_do_hub"]]
hub_df = hub_df.drop_duplicates().sort_values(by="regiao_do_hub").reset_index(drop=True)
hub_df.head()

#Criando uma nova série para as coordenadas dos hubs
hub_df["coordinates"] = hub_df["latitude_do_hub"].astype(str)  + ", " + hub_df["longitude_do_hub"].astype(str) 
hub_df["geodata"] = hub_df["coordinates"].apply(geocoder)
hub_df.head()

#aplicando a função apply, para que todas onde ponssamos normalizar a coluna de geodata, assim como já fizemos anteriormente 
hub_geodata_df = pd.json_normalize(hub_df["geodata"].apply(lambda data: data.raw))
hub_geodata_df.head()

#Agora nós vamos atribuir novos valores e renomear as colunas para melhor compreensão
hub_geodata_df = hub_geodata_df[["address.town", "address.suburb", "address.city"]]
hub_geodata_df.rename(columns={"address.town": "hub_town", "address.suburb": "hub_suburb", "address.city": "hub_city"}, inplace=True)
hub_geodata_df["hub_city"] = np.where(hub_geodata_df["hub_city"].notna(), hub_geodata_df["hub_city"], hub_geodata_df["hub_town"])
hub_geodata_df["hub_suburb"] = np.where(hub_geodata_df["hub_suburb"].notna(), hub_geodata_df["hub_suburb"], hub_geodata_df["hub_city"])
hub_geodata_df = hub_geodata_df.drop("hub_town", axis=1)
hub_geodata_df.head()

#Depois dos valores de região serem extraidos, os dados vão se juntar para uma informação mais completa.
hub_df = pd.merge(left=hub_df, right=hub_geodata_df, left_index=True, right_index=True)
hub_df = hub_df[["regiao_do_hub", "hub_suburb", "hub_city"]]
hub_df.head()


#Apóes importar os dados de geolocalização do repositório, nós vamos descompactar o arquivo para leitura
!wget -q "https://raw.githubusercontent.com/edersonss1987/analise-de-dados-analise-de-empresa-de-logistica-loggi/main/deliveries-geodata.zip" -O deliveries-geodata.zip
!unzip -q deliveries-geodata.zip -d /content/
deliveries_geodata_df = pd.read_csv("deliveries-geodata.csv")
deliveries_geodata_df.head()

#agora nós realizar outro Merge, unir os dataframes de origem(df) e destino(endereço de entrega)
entregas_df = pd.merge(left=entregas_df, right=deliveries_geodata_df[["delivery_city", "delivery_suburb"]], how="inner", left_index=True, right_index=True)
entregas_df.head()

#Apenas vamos nos certificar que os dados estão ok Após Manipilação dos dados, para que possamos plotar os datas visualization e criar insights
#tipos de dados por colunas
entregas_df.info()

#valores nulos por dados gerados
entregas_df.isna().any()

#calculando o percentual de dados que estão sendo representados como nulos por coluna
100 * (entregas_df["delivery_city"].isna().sum() / len(entregas_df))

#calculando o percentual de dados que estão sendo representados por valor atribuido em delivery_city, nota-se que Brasilia tem um volume muito superior
prop_df = entregas_df[["delivery_city"]].value_counts() / len(entregas_df)
prop_df.sort_values(ascending=False).head(10)

#calculando o percentual de dados que estão sendo representados por valor atribuido em delivery_suburb, nota-se que Brasilia tem um volume muito superior
prop_df = entregas_df[["delivery_suburb"]].value_counts() / len(entregas_df)
prop_df.sort_values(ascending=False).head(10)

#Agora vamos coletar o shap de mapas, para plotagem e extrair os mesmos neste notebook
!wget -q "https://geoftp.ibge.gov.br/cartas_e_mapas/bases_cartograficas_continuas/bc100/go_df/versao2016/shapefile/bc100_go_df_shp.zip" -O distrito-federal.zip
!unzip -q distrito-federal.zip -d ./maps
!cp ./maps/LIM_Unidade_Federacao_A.shp ./distrito-federal.shp
!cp ./maps/LIM_Unidade_Federacao_A.shx ./distrito-federal.shx

#Realizando a leitura do shape
mapa = gdf.read_file("./distrito-federal.shp")
mapa = mapa.loc[[0]]
mapa.head()

#Do daframe hub, nós vamos remover dados duplicados e extrair os pontos de cada distrito em uma nova coluna
hub_df = entregas_df[["regiao_do_hub", "longitude_do_hub", "latitude_do_hub"]].drop_duplicates().reset_index(drop=True)
geo_hub_df = gdf.GeoDataFrame(hub_df, geometry=gdf.points_from_xy(hub_df["longitude_do_hub"], hub_df["latitude_do_hub"]))
geo_hub_df.head()

#A cada ponto de entrega nós vamos extrair os pontos de sua localização em uma nova coluna
geo_deliveries_df = gdf.GeoDataFrame(entregas_df, geometry=gdf.points_from_xy(entregas_df["longitude_da_entrega"], entregas_df["latitude_da_entrega"]))
geo_deliveries_df.head()

#Quantidade de entregas por hub
#Conforme analise, podemos identificar que o Hub cvrp-1-df-87	é quem mais faz entregas, na região df-1

entregas = entregas_df.groupby(by='nome_do_hub')\
.agg({'id_da_entrega': 'count', 'regiao_do_hub':'first'})\
.sort_values(by='id_da_entrega', ascending=False).reset_index()
entregas.head()

#Contando o número de hubs por região
#Podemos afirmar que a região que mais entrega é a df-1, com um volume de 304708

hub_por_regiao = entregas_df.groupby(by='regiao_do_hub')\
.agg({'nome_do_hub': 'count'})\
.sort_values(by='nome_do_hub', ascending=False)\
.reset_index()

hub_por_regiao.head()


#Calculo em percentual por região
#Identificamos que a região df-1, corresponde a 47.898841% do volume de entregas

percent = (hub_por_regiao['nome_do_hub'] /
 hub_por_regiao['nome_do_hub'].sum()) * 100
percent.head()

entregas_df.select_dtypes("object").describe().transpose()
entregas_df.drop(["nome_do_hub", "regiao_do_hub"], axis=1).select_dtypes('int64').describe().transpose()

#Utilizando o metodo haversine, nós vamos calcular a distancia percorrida dos hubs para os endereços de entrega
entregas_df["distance_hub_delivery"] = entregas_df.index
entregas_df["distance_hub_delivery"] = entregas_df["distance_hub_delivery"].apply(lambda d: haversine((entregas_df['latitude_do_hub'][d], entregas_df['longitude_do_hub'][d]),(entregas_df['latitude_da_entrega'][d], entregas_df['longitude_da_entrega'][d])))
entregas_df.head()

#Criando uma nova coluna para identificar qual distrito percorre a maior rota
entregas_df['consumo_de_combustiel'] = (5.26/10.5)*entregas_df['distance_hub_delivery']
data_mean_distance = entregas_df[['regiao_do_hub', 'distance_hub_delivery']].groupby('regiao_do_hub', as_index=False).agg('mean').sort_values(by=['distance_hub_delivery'], ascending=False)
data_mean_distance = data_mean_distance.round(2)
data_mean_distance.head()

#identificando uma maior média de consumo de combustível por DF
data_spent_gas = entregas_df[['regiao_do_hub', 'consumo_de_combustiel']].groupby('regiao_do_hub', as_index=False).agg('mean').sort_values(by=['consumo_de_combustiel'], ascending=False)
data_spent_gas = data_spent_gas.round(2)
data_spent_gas.head()

## 5\. Visualização
#Plotando proporção em percentual de entregas, por região.
data = pd.DataFrame(entregas_df[['regiao_do_hub', 'capacidade_do_veiculo']].value_counts(normalize=True)).reset_index()
data.rename(columns={0: "region_percent"}, inplace=True)

with sns.axes_style('whitegrid'):
  grafico = sns.barplot(data=data, x="regiao_do_hub", y="region_percent", ci=None, palette="pastel")
  grafico.set(title='Proporção de entregas por região', xlabel='Região', ylabel='Proporção');

  #Agora nós vamos criar os datas visualization a base de referencias de geolocalização
coordenadas= []
for lat, lng in zip(geo_deliveries_df.latitude_da_entrega, geo_deliveries_df.longitude_da_entrega):
  coordenadas.append([lat, lng])

#usando uma referencia para visualização
maps = folium.Map(location=[-15.797167522318809, -47.890771289228894], zoom_start=5, tiles='Stamen Toner')
from folium import plugins
maps.add_child(plugins.HeatMap(coordenadas))

#1° cria o plot vazio
fig, ax = plt.subplots(figsize = (50/2.54, 50/2.54))

#2° plot mapa do distrito federal
mapa.plot(ax=ax, alpha=0.4, color="lightgrey")

#3° plot das entregas
geo_deliveries_df.query("regiao_do_hub == 'df-0'").plot(ax=ax, markersize=1, color="red", label="df-0")
geo_deliveries_df.query("regiao_do_hub == 'df-1'").plot(ax=ax, markersize=1, color="blue", label="df-1")
geo_deliveries_df.query("regiao_do_hub == 'df-2'").plot(ax=ax, markersize=1, color="seagreen", label="df-2")

#4° plot dos hubs
geo_hub_df.plot(ax=ax, markersize=30, marker="x", color="black", label="hub")

#5º plot da legenda
plt.title("Entregas no Distrito Federal por Região", fontdict={"fontsize": 16})
lgnd = plt.legend(prop={"size": 15})
for handle in lgnd.legendHandles:
    handle.set_sizes([50])

    #Plorando uma média da distância entre os Hubs e os destinos
with sns.axes_style('whitegrid'):
  grafico = sns.barplot(data=data_mean_distance, x="regiao_do_hub", y="distance_hub_delivery", ci=None, palette="bright")
  grafico.set(title='Média de Km percorrido entre o Hub e o local da entrega', xlabel='Hub', ylabel='Média de Km percorrido');


  



