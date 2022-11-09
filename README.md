# Big Data Cluster
1. A estrutura de big data usada nesse trabalho está totalmente descrita [aqui](https://github.com/Antonio-Borges-Rufino/Hadoop_Ecosystem).
2. Assume-se que para repodução das tarefas, usa-se um cluster igual ou similar a esse apresentado.

# Detalhes do projeto
1. O monitoramento de recursos ambientais se tornaram uma importante ferramente nos ultimos anos, muito disso se deu por causa do crescente interesse das autoridades de monitorar seus ecossistemas. Sabendo da importancia que esse tipo de pesquisa tem, uma ONG chamada rios vivos (Nome teórico) decidiu monitorar a temperatura da superficie de pontos específicos do rio amazonas. A temperatura da água é um importante indicador de processos oceânicos, e vital para o ecossistema marinho.
2. Para realizar esse trabalho, pensou-se na sequinte estrutura.  
  -> 1. Colocar sensores de temperatura em lugares estratégicos.   
  -> 2. Coletar dados dos sensores em um espaço de tempo de 1 hora.  
  -> 3. Guardar esse dados em uma base de dados de fácil acesso.  
  -> 4. Apresentar os dados de forma interativa através de uma interface gráfica.
3. Levando em consideração as especificações do projeto, decidiu-se usar um sistema big data de stream.

# Arquitetura do projeto
![](https://github.com/Antonio-Borges-Rufino/IoT_Data_Enginer_Streamin/blob/main/Sensor%201.png)
1. A arquitetura de funcionamento está representada na imagem acima.
2. Primeiro, os sensores se comunicarão diretamente com o Kafka através do protocolo de ioT MQTT.
3. O kafka é um sistema de mensageria baseado em key:value, para o nosso projeto, a key será estruturada como ano-mes-dia-hora-sensor1/sensor2, enquanto o value vai ser o valor da temperatura registrada em kelvins.
4. Depois que o kafka receber os dados dos sensores, as mensagens vão ser lidas pelo spark strean em tempo real, esse serviço deve ficar rodando no back-end.
5. O spark não vai necessariamente precisar fazer algum tipo de análise nos dados, apesar de que isso pode ser muito bem vindo, pode-se futuramente trabalhar em identificações de anomalias.
6. Por fim, o spark deve salvar o registro no banco de dados redis, essa base de dados foi escolhida pois é extremamente rápida e simples, podendo facilmente se utilizada na persistencia de dados em stream.
7. Para facilitar a visualização dos dados, foi construída uma interface gráfica que deve mostrar graficamente e de qualquer máquina com acesso ao redis as informações em tempo real.

# Obtendo os dados e o problema do MQTT
![](https://github.com/Antonio-Borges-Rufino/IoT_Data_Enginer_Streamin/blob/main/SENSOR%201.PNG)
1. Apesar do projeto ser baseado em uma conversa intermediada entre os sensores e o kafka pelo MQTT, na prática isso é dificil de fazer se voce não tem os sensores nem um serviço MQTT online.
2. Para contornar esse problema, simulei a parte do sensor+MQTT-Broker através de um código python.
3. Para não sair muito da idéia, extrai informações reais de temperatura das localizações mostrada na imagem acima, portanto, admiti-se que esses são os sensores e que os dados retirados de temperatura são os dados fornecidos.
4. As imagens de satélite são de estrutura temporais diárias, portanto, teoricamente não teríamos como extrair em horas, como o sensor deve fornecer. Para contornar esse problema, foi baixada imagens diárias entre os anos de 2003 e 2010. Para extrair os pontos do mapa, usou-se apenas as 2 primeiras casas decimais após a vírgula, para poder ficar em conformância com a resolução do satélite e cada imagem representa uma hora do dia, de forma linear.
5. A função que faz o download da imagem e a extração dos pontos é descrita abaixo.
```
def get_data(nome):
  link="https://archive.podaac.earthdata.nasa.gov/podaac-ops-cumulus-protected/MUR-JPL-L4-GLOB-v4.1/"+nome+"090000-JPL-L4_GHRSST-SSTfnd-MUR-GLOB-v02.0-fv04.1.nc"
  subprocess.run(["wget",'-c','-O'+nome,'--user=XXXXXXX','--password=XXXXX',link]) 
  temp = xr.open_dataset("/content/"+nome)
  sensor1 = temp.sel(lat=-2.90,lon=-60.56).analysed_sst.data[0]
  sensor2 = temp.sel(lat=-2.50,lon=-55.00).analysed_sst.data[0]
  return [[-2.90,-60.56,sensor1],[-2.50,-55.00,sensor2]]
```
  -> 1. É passado como parâmetro o nome do arquivo, a construção do nome vai ser explicado mais adiante.  
  -> 2. Depois, usa-se o nome do arquivo dentro do link de download, optei por colocar em uma variável separada para poder ficar mais organizado.  
  -> 3. A função subprocess.rum é quem realiza comandos do terminal no python, ela quem vai baixar a imagem rastel do satélite.  
  -> 4. xr.open_dataset é uma função de uma biblioteca responsável por fazer varreduras e extrações em imagens rastel, nela, pode-se trabalhar a maioria dos formatos,         como NetCDF, NetCDF4 dentre outras.  
  -> 5. As linhas sesor1 e sensor2 são responsaveis por extrair as informações dos metadados das imagens, nesse caso,usa-se a função sel da biblioteca xarray para extrair a temperatura das coordenadas passadas nos parametros lat e lon.    
6. O código abaixo representa como foi feito o download das imagens e a suas respectivas extrações. Apesar de poder deixar um looping de ano, possuia limitações de tempo, então para deixar menos dificultoso meu trabalho, coloquei o ano como uma variavel setavel. 
```
ano_i = 2010
for mes in range(1,13):
  data = list()
  for dia in range(1,32):
    if dia < 10:
      dia_ = '0'+str(dia)
    else:
      dia_ = str(dia)
    if mes < 10:
      mes_ = '0'+str(mes)
    else:
      mes_ = str(mes)
    nome = str(ano_i)+str(mes_)+str(dia_)
    try:
      dados = get_data(nome)
      data.append(dados[0])
      data.append(dados[1])
      subprocess.run(['rm','/content/'+nome])
    except:
      print("Erro do dia: {}".format(dia))
  data = pd.DataFrame(data,columns=['Lat','Lon','Temp'])  
  nome = str(ano_i)+str(mes)+".csv"
  data.to_csv(nome)
```  
7. Explicação do código
  -> 1. A variavel ano_i vai receber o ano que a imagem pertence.  
  -> 2. O primei laço for é respectivo aos meses do ano.  
  -> 3. a variavel data vai receber as informações dos 2 pontos para cada dia do mes.  
  -> 4. A extração dos pontos é feita dentro do laço dia,que é respectivo para cada dia. Aqui, eu faço uma verificação simples para adicionar 0 ao nome, pois a formatação do link das imagens exige um 0 a esquerda de todo numero menor que 10.  
  -> 5. O nome é a junção do ano+mes+dia no link de download explicado anteriormente, por exemplo, 01/10/2010 == 01102010.  
  -> 6. o próximo passo é fazer o donwload e a extração dos pontos a partir da função get_data já explicada, e colocando dentro da variavel data (para o mes todo).  
  -> 7. Depois que todos os meses foram baixados, transformo a variavel data em um dataframe pandas e salvo apenas com o nome ano+mes.csv, esses dados vão ser unidos logo a frente.  

```
spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.3.1 /home/hadoop/Spark-GET-MQTT-Kafka-Data.py
``` 
