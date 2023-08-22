# EklKibanaExampleWebApi

Bu projede Serilog ile elasticsearch'e log atmak ve kibana ile gösterimini yapmak üzerine bir geliştirme yapılmıştır. 

#### Geliştirme adımları; 

Projenin oluşturulması;

```cmd
dotnet new webapi –name EklKibanaExampleWebApi
```
İlgili Paketlerin Projeye eklenmesi;

```cmd

dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Exceptions
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.Debug
dotnet add package Serilog.Sinks.Elasticsearch

```
Paketleri projeye ekledikten sonra program.cs de gerekli configurasyonlerı yapıyoruz.

```c#

void configureLogging(){
    var environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT");

    var configuration = new ConfigurationBuilder()
       .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
       .AddJsonFile(
        $"appsettings.{environment}.json", optional: true)
       .Build();

    Log.Logger = new LoggerConfiguration()
        .Enrich.FromLogContext()
        .Enrich.WithExceptionDetails()
        .WriteTo.Debug()
        .WriteTo.Console()
        .WriteTo.Elasticsearch(ConfigureElasticSink(configuration, environment ?? string.Empty))
        .Enrich.WithProperty("Environment", environment??string.Empty)
        .ReadFrom.Configuration(configuration)
        .CreateLogger();
}

ElasticsearchSinkOptions ConfigureElasticSink(IConfigurationRoot configuration, string environment)
{
    return new ElasticsearchSinkOptions(new Uri(configuration["ElasticConfiguration:Uri"]))
    {
        AutoRegisterTemplate =true,
        IndexFormat=$"{Assembly.GetExecutingAssembly().GetName()?.Name?.ToString().ToLower().Replace(".","-")}-{environment.ToLower()}-{DateTime.UtcNow:yyyy-MM}",
        NumberOfReplicas = 1,
        NumberOfShards = 2
    };

}

```

Bu konfigurasyon ile indeximiz otomatik olarak oluşacaktır. Kibana üzerinden gerekli düzenlemeleri aşağıda görsel olarak ekledim.


appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Information",
        "System": "Warning"
      }
    }
  },
  "ElasticConfiguration": {
    "Uri": "http://elasticsearch:9200"
  },
  "AllowedHosts": "*"
}
```

Ardından build'den önce

```c#
configureLogging();

builder.Host.UseSerilog();

```

ekliyoruz.


Projeye sağ tıklayıp Add> Add Docker Support deyip projemizin Dockerfile'ının oluşturuyoruz.

Sonra Elasticsearc, kibana ve projemizi docker'da ayağa kaldıracak şekilde docker-compose.yml dosyamızı ekleyip aşağıdaki gibi düzenliyoruz.
yml dosyası projenizin gereksinimleri doğrultusunda düzenlenmeli. 

```docker-compose.yml

version: '3.1'

services:
  elasticsearch:
    container_name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:7.13.4
    environment:
      - "discovery.type=single-node"
      - "bootstrap.memory_lock=true"
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"

  kibana:
    container_name: kibana
    image: docker.elastic.co/kibana/kibana:7.13.4
    environment:
      - "ELASTICSEARCH_URL=http://elasticsearch:9200"
      - "ELASTICSEARCH_HOSTS=http://elasticsearch:9200"
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  EklKibanaExampleWebApi:
    container_name: EklKibanaExampleWebApi
    build:
      context: .
      dockerfile: Dockerfile   

    image: eklkibanaexamplewebapi   
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://0.0.0.0:80
    ports:
      - "6060:80"
      - "6061:443"
    

```

```
docker-compose up
```

komutu ile servislerimizi docker'da ayağa kaldırıyoruz.

![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/7b292878-faae-4722-91b9-321753566f53)


ElasticSearch:
![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/43ed8789-c439-4f9d-b44d-7cc12469544a)


Kibana:

![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/2aa424a4-02d5-4267-84c1-84708d3b8878)


WebApi:

![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/6880e889-01f7-4140-8935-0ae766d1d425)


Kibana'da index'imizi ekleyelim; 
![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/56df4c9b-bfe8-447f-a753-537cb0e69ac4)


![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/30bb9bbf-2b70-44f1-84c1-671d761f2618)


![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/1da7d465-4485-4f44-bc34-c057357d12c1)


![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/8e3a2710-e707-45e6-9a03-757444dc7b15)

İndex'imizi oluşturduktan sonra ilgili loglarımızı görüntüleyebiliyoruz. 

![image](https://github.com/coderentr/EklKibanaExampleWebApi/assets/33000530/a8cd517a-393b-403e-aaab-5caaae164c50)


