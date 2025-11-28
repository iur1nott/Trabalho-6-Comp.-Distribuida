# Trabalho 6 - Computação Distribuída: Comparação de Protocolos

## Introdução

Este trabalho compara quatro protocolos de comunicação amplamente utilizados em sistemas distribuídos: REST, SOAP, GraphQL e gRPC. Cada um possui características, vantagens e desvantagens específicas que os tornam mais adequados para diferentes cenários.

## 🔗 REST (Representational State Transfer)

### Origem
REST é um estilo arquitetural para aplicações cliente-servidor, proposto por Roy Fielding para orientar o design da Web. Sua ideia surgiu durante a evolução da Web, buscando definir um padrão para troca de recursos via HTTP.

### Características Principais
- Usa HTTP e seus verbos (GET, POST, PUT, DELETE etc.) para operações sobre recursos
- Segue restrições como: cliente-servidor desacoplado; stateless; interface uniforme; suporte a cache
- É "agnóstico de linguagem" — implementável em qualquer linguagem que suporte HTTP/JSON

### Vantagens
- Simplicidade e ampla familiaridade
- Flexibilidade e compatibilidade com diferentes ambientes
- Escalabilidade favorecida pelo stateless + HTTP + cache

### Desvantagens
- Over-fetching ou under-fetching de dados
- Cada recurso tipicamente exige endpoint próprio
- Statelessness exige que cada requisição traga todo contexto necessário

### Implementação REST

```python
# rest_server.py
from flask import Flask, send_file, jsonify
from music_data import MUSIC_CATALOG as TRACKS

app = Flask(__name__)

@app.route('/tracks', methods=['GET'])
def get_tracks():
    return jsonify(TRACKS)

@app.route('/stream/<int:track_id>', methods=['GET'])
def stream_track(track_id):
    track = next((t for t in TRACKS if t["id"] == track_id), None)
    if not track:
        return "Track não encontrada", 404
    return send_file(
        track["file"],
        mimetype='audio/mpeg',
        as_attachment=False,
        conditional=True
    )

if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

**Características da Implementação:**
- Endpoints específicos para cada recurso (`/tracks`, `/stream/<id>`)
- Respostas em JSON
- Stateless por padrão
- Uso direto de verbos HTTP

## 🧩 SOAP (Simple Object Access Protocol)

### Origem
SOAP é um protocolo de mensagens para web services, baseado em XML. É mais antigo na história de APIs, sendo utilizado amplamente em sistemas corporativos.

### Características Principais
- Comunica via mensagens XML estruturadas sobre HTTP ou outros protocolos
- Usa especificações formais (WSDL) para definir contratos da API
- Suporta extensões como segurança (WS-Security), transações, interoperabilidade

### Vantagens
- Contrato bem definido e formal
- Suporte a segurança robusta e padrões maturados
- Boa para integração entre sistemas diferentes e legados

### Desvantagens
- Verbosidade e complexidade (XML pesado)
- Tempo de desenvolvimento maior
- Menos flexível em cenários modernos

### Implementação SOAP

```python
# soap_server.py
from spyne import Application, rpc, ServiceBase, Integer, Unicode, Array, ComplexModel
from spyne.protocol.soap import Soap11
from spyne.server.wsgi import WsgiApplication
from wsgiref.simple_server import make_server
import base64
from music_data import MUSIC_CATALOG

class Track(ComplexModel):
    id = Integer
    title = Unicode
    artist = Unicode
    album = Unicode
    genre = Unicode
    duration_sec = Integer

class MusicService(ServiceBase):
    @rpc(_returns=Array(Track))
    def GetTrackList(ctx):
        return [Track(**track) for track in MUSIC_CATALOG]

    @rpc(Integer, _returns=Unicode)
    def StreamTrack(ctx, track_id):
        track = next((t for t in MUSIC_CATALOG if t["id"] == track_id), None)
        if not track or not track["file"]:
            return ""
        try:
            with open(track["file"], "rb") as f:
                return base64.b64encode(f.read()).decode('utf-8')
        except FileNotFoundError:
            return ""

application = Application(
    [MusicService],
    tns='spyne.examples.music',
    in_protocol=Soap11(validator='lxml'),
    out_protocol=Soap11()
)

if __name__ == '__main__':
    wsgi_app = WsgiApplication(application)
    server = make_server('0.0.0.0', 8000, wsgi_app)
    print("SOAP server rodando em http://0.0.0.0:8000/?wsdl")
    server.serve_forever()
```

**Características da Implementação:**
- Definição formal de tipos com `ComplexModel`
- Uso de decoradores `@rpc` para operações
- Serialização automática em XML
- Base64 para dados binários

## 📊 GraphQL

### Origem
GraphQL foi criado pelo Facebook e lançado publicamente em 2015. Surgiu para resolver problemas de APIs tradicionais no contexto de aplicações modernas.

### Características Principais
- Usa uma única endpoint para todo o serviço
- Cliente define o que quer buscar com consultas precisas
- Fortemente tipado com schema definido
- Permite flexibilidade evitando over-fetching e under-fetching

### Vantagens
- Flexibilidade e eficiência para o cliente
- Bom para aplicações com clientes variados e dados complexos
- Facilita evolução da API sem multiplicação de endpoints

### Desvantagens
- Curva de aprendizado mais alta
- Problemas de cache mais complexos
- Sobrecarga no servidor para resolver queries

### Implementação GraphQL

```python
# graphql_server.py
import graphene
from flask import Flask, request, jsonify, send_file
from music_data import MUSIC_CATALOG

class Track(graphene.ObjectType):
    id = graphene.ID()
    title = graphene.String()
    artist = graphene.String()
    album = graphene.String()
    genre = graphene.String()
    duration_sec = graphene.Int()
    stream_url = graphene.String()

class Query(graphene.ObjectType):
    tracks = graphene.List(Track)
    track = graphene.Field(Track, id=graphene.ID(required=True))

    def resolve_tracks(root, info):
        return [
            Track(
                id=t["id"],
                title=t["title"],
                artist=t["artist"],
                album=t["album"],
                genre=t["genre"],
                duration_sec=t["duration_sec"],
                stream_url=f"/stream/{t['id']}"
            )
            for t in MUSIC_CATALOG
        ]

    def resolve_track(root, info, id):
        id_int = int(id)
        track_data = next((t for t in MUSIC_CATALOG if t["id"] == id_int), None)
        if not track_data:
            return None
        return Track(
            id=track_data["id"],
            title=track_data["title"],
            artist=track_data["artist"],
            album=track_data["album"],
            genre=track_data["genre"],
            duration_sec=track_data["duration_sec"],
            stream_url=f"/stream/{track_data['id']}"
        )

schema = graphene.Schema(query=Query)
app = Flask(__name__)

@app.route('/graphql', methods=['GET', 'POST'])
def graphql_server():
    data = request.get_json()
    query = data.get('query')
    variables = data.get('variables')
    
    result = schema.execute(query, variables=variables)
    
    if result.errors:
        return jsonify({'errors': [str(e) for e in result.errors]}), 400
    
    return jsonify({'data': result.data})

if __name__ == '__main__':
    app.run(port=5001, debug=True)
```

**Características da Implementação:**
- Schema fortemente tipado com Graphene
- Único endpoint `/graphql` para todas as queries
- Resolvers dinâmicos que processam queries específicas
- Cliente controla quais campos retornar

## ⚡ gRPC

### Origem
gRPC é um framework de RPC de alto desempenho, criado pelo Google. É projetado para comunicação eficiente entre serviços em arquiteturas de microsserviços.

### Características Principais
- Usa HTTP/2 + Protocol Buffers (protobuf) para comunicação binária
- Baseado em chamadas de procedimento remoto (RPC)
- Tipagem forte com contratos via arquivos .proto

### Vantagens
- Alta performance e eficiência (baixa latência, alto throughput)
- Tipagem forte e contratos bem definidos
- Ideal para microserviços e comunicação backend-backend

### Desvantagens
- Overkill para aplicações simples
- Menos plug-and-play para front-end/browser
- Curva de configuração mais alta

### Implementação gRPC

```python
# grpc_server.py
import grpc
from concurrent import futures
import music_pb2
import music_pb2_grpc
from music_data import MUSIC_CATALOG

TRACKS_BY_ID = {track["id"]: track for track in MUSIC_CATALOG}

class MusicService(music_pb2_grpc.MusicServiceServicer):
    
    def GetTrackList(self, request, context):
        tracks = []
        for t in MUSIC_CATALOG:
            tracks.append(music_pb2.Track(
                id=t["id"],
                title=t["title"],
                artist=t["artist"],
                album=t["album"],
                genre=t["genre"],
                duration_sec=t["duration_sec"]
            ))
        return music_pb2.TrackListResponse(tracks=tracks)

    def StreamTrack(self, request, context):
        track_id = request.track_id
        track = TRACKS_BY_ID.get(track_id)
        
        if not track:
            context.abort(grpc.StatusCode.NOT_FOUND, "Track not found")
        
        mock_data = (
            f'{{"id":{track["id"]},"title":"{track["title"]}",'
            f'"artist":"{track["artist"]}"}}'
        ).encode('utf-8')
        
        yield music_pb2.TrackChunk(data=mock_data)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    music_pb2_grpc.add_MusicServiceServicer_to_server(MusicService(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    print("✅ Mock gRPC server rodando na porta 50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

**Características da Implementação:**
- Comunicação binária com Protocol Buffers
- Serviços definidos em arquivos .proto
- Streaming nativo com HTTP/2
- Stubs gerados automaticamente

## 📈 Comparação de Desempenho

<img width="1920" height="1091" alt="Figure_1" src="https://github.com/user-attachments/assets/1098494c-1f90-41d7-808f-a6e06bb77a04" />


### Métricas Comparativas

| **Protocolo** | **Latência (ms)**<br>*média* | **Latência (ms)**<br>*desvio padrão* | **Throughput (%)**<br>*média* | **Throughput (%)**<br>*desvio padrão* | **Uso de CPU (%)**<br>*média* | **Uso de CPU (%)**<br>*desvio padrão* |
|---------------|------------------------------|--------------------------------------|-------------------------------|---------------------------------------|-------------------------------|---------------------------------------|
| **GraphQL**   | 45.84                        | 4.93                                 | 76.99                         | 7.79                                  | 76.76                         | 25.96                                 |
| **REST**      | 26.80                        | 1.01                                 | 79.27                         | 8.86                                  | 45.40                         | 16.16                                 |
| **SOAP**      | 117.83                       | 34.10                                | 66.79                         | 2.31                                  | 90.21                         | 8.30                                  |
| **gRPC**      | **19.31**                    | 4.92                                 | **90.00**                     | 4.74                                  | 50.70                         | 29.51                                 |

### Análise de Performance

#### 1. **gRPC - Mais Eficiente**
**Justificativa:**
- HTTP/2 com multiplexação (múltiplas requisições na mesma conexão TCP)
- Serialização binária com Protocol Buffers (muito mais compacta)
- Menor overhead de rede e parsing mais rápido

#### 2. **REST - Segundo Lugar**
**Justificativa:**
- Protocolo stateless simples
- Overhead menor que GraphQL/SOAP
- Cache HTTP fácil de implementar

#### 3. **GraphQL - Terceiro Lugar**
**Justificativa:**
- Overhead de parsing de queries
- Risco de queries complexas que sobrecarregam o servidor
- Respostas tipicamente em JSON

#### 4. **SOAP - Menos Eficiente**
**Justificativa:**
- XML extremamente verboso (muito overhead)
- Parsing de XML computacionalmente caro
- WSDL e schemas complexos

### Detalhes Técnicos de Performance

```python
# Comparação de tamanho de dados
JSON: {"id": 1, "title": "Song", "artist": "Artist"}  # ~50 bytes
Protobuf: \x08\x01\x12\x04Song\x1a\x06Artist         # ~15 bytes (70% menor)
```

### Cenários de Carga Ótimos:
- **Alta concorrência**: gRPC > REST > GraphQL > SOAP
- **Largura de banda limitada**: gRPC > GraphQL ≈ REST > SOAP  
- **Processamento CPU**: gRPC > REST > GraphQL > SOAP

## 🎯 Conclusão

Cada protocolo possui seu nicho de aplicação ideal:

- **REST**: Melhor para APIs públicas, desenvolvimento rápido e ecossistemas web tradicionais
- **SOAP**: Adequado para sistemas corporativos legados que exigem contratos rigorosos
- **GraphQL**: Ideal para aplicações com clientes variados e necessidades de dados complexas
- **gRPC**: Excelente para comunicação interna entre microsserviços onde performance é crítica

A escolha depende dos requisitos específicos do projeto, considerando fatores como performance, complexidade, ecossistema e necessidades dos clientes.
