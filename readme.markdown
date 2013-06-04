# introdução

Este documento cobre coisas basicas de como escrever aplicações com [node.js](http://nodejs.org/)
utilizando [streams](http://nodejs.org/docs/latest/api/stream.html).

```
"Nós temos muitas maneiras de conectar programas como mangueiras de jardim e parafusos,
em um outro segmento onde se tornam necessarias para enviar menssagens em uma outra
direção. Esse meio é o do IO."
```

[Doug McIlroy. October 11, 1964](http://cm.bell-labs.com/who/dmr/mdmpipe.html)

![doug mcilroy](http://substack.net/images/mcilroy.png)

***

Streams chegaram até nós através dos
[primeiros dias de vida do unix](http://www.youtube.com/watch?v=tc4ROCJYbm0)
e provaram-se por muitas decadas como um meio seguro de compor grandes
sistemas de pequenas partes que
[fazem uma coisa bem](http://www.faqs.org/docs/artu/ch01s06.html).
No unix, streams são implementados pelo shell com `|` pipes.
No node, vem embutido no
[módulo de stream](http://nodejs.org/docs/latest/api/stream.html)
é usado pelas partes do núcleo do node e pode ser usado pelo usuario em seus módulo "user-space".
Similar ao unix, os módulos de stream no node tem como composição primaria o operador que chama-se
`.pipe()` e você utiliza em um mecanismo de contrapressão para uma livre escrita estrangulando
consumidores lentos.

Streams podem ajudar a
[separar suas preocupações](http://www.c2.com/cgi/wiki?SeparationOfConcerns)
porque eles restringem a implementação em uma area superficial em uma concistente
interface que pode ser muito bem
[reutilizada](http://www.faqs.org/docs/artu/ch01s06.html#id2877537).
Você pode conectar o fluxo de saida em uma entrada de uma outra lib e 
[usar libs](http://npmjs.org) que operam de modo abstrato em outros streams para
instituir o controle do fluxo de um alto-nivel.

Streams are an important component of
[small-program design](https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/)
and [unix philosophy](http://www.faqs.org/docs/artu/ch01s06.html)
but there are many other important abstractions worth considering.
Just remember that [technical debt](http://c2.com/cgi/wiki?TechnicalDebt)
is the enemy and to seek the best abstractions for the problem at hand.

Streams são um importante component de
[projeção para pequenos programas](https://michaelochurch.wordpress.com/2012/08/15/what-is-spaghetti-code/)
e podendo seguir a [filosofia do unix](http://www.faqs.org/docs/artu/ch01s06.html)
mas tendo outras abstrações importantes para serem consideradas.
Somente lembre-se que o [débito técnica](http://www.faqs.org/docs/artu/ch01s06.html)
é o inimigo para buscar as melhores abstrações que estão na sua mão. 

![brian kernighan](http://substack.net/images/kernighan.png)

***

# Por que você deve usar streams

I/O no node é assíncrono, então a interação com o disco e rede envolve
a passagem de funções como callbacks. Você pode ser tentado a escrever um código que serve
um arquivo do disco dessa maneira:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    fs.readFile(__dirname + '/data.txt', function (err, data) {
        if (err) {
            res.statusCode = 500;
            res.end(String(err));
        }
        else res.end(data);
    });
});
server.listen(8000);
```

Este código funciona mas é volumoso e joga todo o conteúdo de `data.txt` na
memória varias requisições antes de escrever o resultado nos clientes. Se
servindo varios usuarios concorrentes. A latência será alta fazendo com que os
usuarios esperem pelo termino da leitura do arquivo para o inicio do recebimento
do conteúdo.

Felizmente ambos os argumentos `(req, res)` são streams, signficando que pode ser escrito
de uma maneira muito melhor usando `fs.createReadStream()` em vez de
`fs.readFile()`:

``` js
var http = require('http');
var fs = require('fs');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.on('error', function (err) {
        res.statusCode = 500;
        res.end(String(err));
    });
    stream.pipe(res);
});
server.listen(8000);
```

Aqui `.pipe()` cuida de quem escuta os eventos `'data'` e `'end'` do
`fs.createReadStream()`. Este código não esta muito limpo, mas agora o arquivo `data.txt`
é escrito em um pedaço para os clientes no mesmo momento em que ele é recebido do disco.

Usando `.pipe()` tem outros benefícios também, como lidar com contrapressões automaticamente
então o node não amortecera pedaços na memória desnecessariamente quando o cliente remoto
estiver conecatado tornando essa ligação lenta e com uma latência maior.

Precisa de compressão? Temos módulos de streaming para isso também!

``` js
var http = require('http');
var fs = require('fs');
var oppressor = require('oppressor');

var server = http.createServer(function (req, res) {
    var stream = fs.createReadStream(__dirname + '/data.txt');
    stream.on('error', function (err) {
        res.statusCode = 500;
        res.end(String(err));
    });
    
    stream.pipe(oppressor(req)).pipe(res);
});
server.listen(8000);
```

Agora nosso arquivo esta comprimido para navegadores que suportam gzip ou esvaziamento 'deflate'! Nós podemos
somente deixar o [opressor](https://github.com/substack/oppressor) lidar com todo o conteúdo dos dados codificadas.

Uma vez que você aprende a api de stream, você pode apenas tirar encaixa os módulos de streaming
como peças de lego ou mangueiras de jardim em vez de ter que lembrar de como empurrar dados
através de vacilantes APIs personalizadas e de não-streaming.

Streams fazem da programação no node simples, elegante e combinável.

# básico

Streams are just
[EventEmitters](http://nodejs.org/docs/latest/api/events.html#events_class_events_eventemitter)
that have a
[.pipe()](http://nodejs.org/docs/latest/api/stream.html#stream_stream_pipe_destination_options)
function and expected to act in a certain way depending if the stream is
readable, writable, or both (duplex).

Streams são somente
[Emissores de Eventos](http://nodejs.org/docs/latest/api/events.html#events_class_events_eventemitter)
que tem um uma função
[.pipe()](http://nodejs.org/docs/latest/api/stream.html#stream_stream_pipe_destination_options)
e que esperam para agir em uma determinada cituação dependendo se o stream é
readable (podem ser apenas lidos), writable (podem ser apenas escritos), ou ambos (duplex).

Para criar um novo steam, apenas faça assim:

``` js
var Stream = require('stream');
var s = new Stream;
```

Este novo stream não tem nada ainda porque não se pode ler ou escrever.

## readable ( leitura )

Para fazer deste stream `s` um stream readable ( para leitura ), tudo o que precisamos fazer é
definir a proprieade `readable` como `true`:

``` js
s.readable = true
```

Streams legíveis emitem muitos eventos `'data'` e um unico evento `'end'`.
Seu stream não ira emitir nenhum evento `'data'` após o evento `'end'` ter sido emitido.

Este simples stream readable ( legível ) emite um evento `'data'` por secundo por 5 segundos,
logo em seguida ele termina. Os dados são canalizados até a stdout ( saida padrão ) para que possamos
assistir os resultados como eles acontecem.

``` js
var Stream = require('stream');

function createStream () {
    var s = new Stream;
    s.readable = true

    var times = 0;
    var iv = setInterval(function () {
        s.emit('data', times + '\n');
        if (++times === 5) {
            s.emit('end');
            clearInterval(iv);
        }
    }, 1000);
    
    return s;
}

createStream().pipe(process.stdout);
```

```
substack : ~ $ node rs.js
0
1
2
3
4
substack : ~ $ 
```

Neste exemplo os eventos `'data'` tem uma carga encadeada (`string`) como primeiro argumeto.
Buffers e strings são os tipos mais comuns de dados para o stream mas as vezes
é mais util emitir outros tipos de objetos.

Just make sure that the types you're emitting as data is compatible with the
types that the writable stream you're piping into expects.
Otherwise you can pipe into an intermediary conversion or parsing stream before
piping to your intended destination.

Apenas certifique-se que os tipos que você esta emitindo são compativeis com os
tipos que o fluxo de escrita que estão sendo canalizados são o esparado.

## writable (gravável)

Streams graváveis são fluxos que aceitam entrada. Para criar um stream gravável,
defina o atributo `writable` como `true` e defina também `write()`, `end()`, e
`destroy()`.

Esse fluxo que pode ser gravado conta todos os bytes de um fluxo de entrada e imprime o
resultado em um `end()` limpo. Se o stream é destruido não fara nada.

``` js
var Stream = require('stream');
var s = new Stream;
s.writable = true;

var bytes = 0;

s.write = function (buf) {
    bytes += buf.length;
};

s.end = function (buf) {
    if (arguments.length) s.write(buf);
    
    s.writable = false;
    console.log(bytes + ' bytes written');
};

s.destroy = function () {
    s.writable = false;
};
```

Se nós canalizamos um arquivo para este fluxo gravável:

``` js
var fs = require('fs');
fs.createReadStream('/etc/passwd').pipe(s);
```

```
$ node writable.js
2447 bytes written
```

One thing to watch out for is the convention in node to treat `end(buf)` as a
`write(buf)` then an `end()`. If you skip this it could lead to confusion
because people expect end to behave the way it does in core.

Uma coisa que devemos ficar atentos nesta convensão do node é que para tratar `end(buf)` como um
`write(buf)` em seguida, um `end()`. Se você pula estes passo, pode acabar criando uma confusão
porque as pessoas esperam o comportamento como se aplicado no núcleo e aqui você esta personalizando eles.

## contrapressão

Contrapressão é o mecanismo que os streams usam para certificar que readable
streams não emitem dados mais rápidos que writable possa consumir dados.

Note: a API que cuida da contrapressão serão modificadas no futuro
versão do node maiores que a (> 0.8). `pause()`, `resume()`, e `emit('drain')` são
programadas para demolição. O aviso foi exposto no escritório como planejamento local a meses.

Na ordem para executar a contrapressão corretamente para fluxos legíveis deveram
ser implementados `pause()` and `resume()`. Fluxos graváveis retornam `false` no
`.write()` quando é necessario que fluxo legível canalizado em uma velocidade branda e
emitindo `'drain'` onde estarão prontos para mais dados outra vez.

### writable stream (contrapressão de fluxos graváveis) 

When a writable stream wants a readable stream to slow down it should return
`false` in its `.write()` function. This causes the `pause()` to be called on
each readable stream source.

Quando um fluxo legível necessita que um fluxo gravável tenha uma redução de velocidade ele 
retornara `false` na função `.write()`. Isto causara em uma necessidade de chamar `pause()`
em cada fonte de fluxo legivel.

When the writable stream is ready to start receiving data again, it should emit
the `'drain'` event. Emitting `'drain'` causes the `resume()` function to be
called on each readable stream source.

Quando o fluxo gravável esta pronto para iniciar o recebimento de dados novamente, ele emitira
o evento `'drain'`. Emitindo `'drain'` causara o a necessidade de chamar a função `resume()`
para cada fonte de fluxo legivel.

### readable stream (contrapressão de fluxos legíveis) 

Quando `pause()` é chamado em um fluxo legível, significara que um fluxo escrito
estiver sofrendo uma caida (ato de ler) o fluxo escrita tera que sofrer uma baixa de velocidade
no modo como isso acontece. No fluxo legível quando chamado `pause()` sofre a parada de emição 
de dados sempre que possivel.

When the downstream is ready for more data, the readable stream's `resume()`
function will be called.

Quando uma caida esta em busca de mais dados para ler, o fluxo legível teve `resume()`
chamado.

## pipe (tubo)

`.piep()` é uma cóla de dados embaralhados de um fluxo legível para um fluxo
gravável e que é tratado com uma contrapressão. O api é somente isso:

```
src.pipe(dst) // fonte criando uma passagem para o destino
```

para um fluxo legível `src` e par um fluxo gravável `dst`. `.pipe()` retorna o
`dst` então se `dst` esta habil a ler um fluxo, você podera `.pipe()` encadear
as chamadas assim:

``` js
a.pipe(b).pipe(c).pipe(d)
```

que assemelha-se ao operador `|` (pipe 'tubo') do shell:

```
a | b | c | d
```

O use de `a.pipe(b).pipe(c).pipe(d)` é igual ao de:

```
a.pipe(b);
b.pipe(c);
c.pipe(d);
```

A implementação de stream no núcleo é somente um evento que emite com uma função `pipe`.
`pipe()` é bem curta. E você pode ler o
[código fonte](https://github.com/joyent/node/blob/master/lib/stream.js).

## termos 

Estes termos são uteis para falar sobre fluxos.

### through (através)

Fluxos atravessados são simplesmente filtros legíveis/graváveis que transformam entrada e
produzem saida.

### duplex

Fluxos duplex são legíveis/graváveis e ambos terminam para o fluxo empenhar um caminho
de mão dupla com interação, enviando de volta e em diante mensagens como um telefone. Uma
troca rpc é um bom exemplo de um fluxo duplex. A todo momento você irá ver algo assim:

``` js
a.pipe(b).pipe(a)
```

você provávelmenta estara lidando com um fluxo duplex.

## leia mais sobre 

* [documentação do módulo de stream que esta presente no núcleo](http://nodejs.org/docs/latest/api/stream.html#stream_stream)
* [notas sobre a api de stream](http://maxogden.com/node-streams)
* [porque streams são incríveis](http://blog.dump.ly/post/19819897856/why-node-js-streams-are-awesome)
* [nota sobre event-stream](http://thlorenz.com/#/blog/post/event-stream)

## O futuro 

Uma grande atualização foi planejada para o api de stream no node 0.9
O basico das apis como `.pipe()` continuaram as mesmas, internamente será diferente.
A nova api tera uma retrocompatibilidade com a api existente documentada por um longo tempo.

Você pode verificar o
repositório [readable-stream](https://github.com/isaacs/readable-stream)
para ver como as mudanças irão parecer.

***

# streams embutidas

Esses fluxos são contruidos no próprio node.

## process (processo)

### [process.stdin](http://nodejs.org/docs/latest/api/process.html#process_process_stdin)

Esse fluxo legível contem o padrão de entrada no sistema para o seu programa.

Ele é pausado por padrão mas da primeira vez ele refere-se a ele `.resume()` que faz
a chamada implicida no 
[next tick](http://nodejs.org/docs/latest/api/process.html#process_process_nexttick_callback) (próxima escala).

Se process.stdin é um tty (verifique
[`tty.isatty()`](http://nodejs.org/docs/latest/api/tty.html#tty_tty_isatty_fd))
segue uma entrada de eventos que é amortecida linearmente. Você pode desligar
o amortecimento linear chamando `process.stdin.setRawMode(true)` MAS os lidadores padrões para
chaves como `^C` e `^D` serão removidas.

### [process.stdout](http://nodejs.org/api/process.html#process_process_stdout)

Este fluxo gravável contem a saida padrão do sistema do seu programa.

`write(escreve)` nele se você precisa enviar dados para strout 

### [process.stderr](http://nodejs.org/api/process.html#process_process_stderr)

Este fluxo gravável contem saida padrão de erro do sistema para o seu programa.

`write(escreve)` para ele se você precisa enviar algum erro para stderr

## child_process.spawn()

## fs

### fs.createReadStream()

### fs.createWriteStream()

## net

### [net.connect()](http://nodejs.org/docs/latest/api/net.html#net_net_connect_options_connectionlistener)

Esta funão retorna a [fluxo duplex] que conecta-se atravês do protocolo `tcp` a um host remoto.

Você pode iniciar escrevendo em um fluxo de modo correto e a escrite sera amortecida
até `'connect'` for acionado.

### net.createServer()

## http

### http.request()

### http.createServer()

## zlib

### zlib.createGzip()

### zlib.createGunzip()

### zlib.createDeflate()

### zlib.createInflate()

***

# control streams

## [through](https://github.com/dominictarr/through)

## [from](https://github.com/dominictarr/from)

## [pause-stream](https://github.com/dominictarr/pause-stream)

## [concat-stream](https://github.com/maxogden/node-concat-stream)

concat-stream amortecem conteudos de um fluxo em um unico buffer.
`contact(cb)` recebe somente um callback `cb(body)` com o `body` amortecido
onde um fluxo é finalizado.

Por exemplo neste programa, o método `contact` aciona a função de callback com o corpo que é um `string`
`"beep boop"` uma vez que o `cs.end()` é chamado.
O programa recebe o corpo e transforma todos os caracteres em maisculos, imprimindo `BEEP BOOP.`

``` js
var concat = require('concat-stream');

var cs = concat(function (body) {
    console.log(body.toUpperCase());
});
cs.write('beep ');
cs.write('boop.');
cs.end();
```

```
$ node concat.js
BEEP BOOP.
```

Aqui um example de uso do concat-stream ele ira analisar a entrada que é uma url codificada
que é um dado do formulario e responde com uma versão em JSON em formato de texto dos parametros do formulario: 

``` js
var http = require('http');
var qs = require('querystring');
var concat = require('concat-stream');

var server = http.createServer(function (req, res) {
    req.pipe(concat(function (body) {
        var params = qs.parse(body);
        res.end(JSON.stringify(params) + '\n');
    }));
});
server.listen(5005);
```

```
$ curl -X POST -d 'beep=boop&dinosaur=trex' http://localhost:5005
{"beep":"boop","dinosaur":"trex"}
```

## [duplex](https://github.com/dominictarr/duplex)

## [duplexer](https://github.com/Raynos/duplexer)

## [emit-stream](https://github.com/substack/emit-stream)

## [invert-stream](https://github.com/dominictarr/invert-stream)

## [map-stream](https://github.com/dominictarr/map-stream)

## [remote-events](https://github.com/dominictarr/remote-events)

## [buffer-stream](https://github.com/Raynos/buffer-stream)

## [event-stream](https://github.com/dominictarr/event-stream)

## [auth-stream](https://github.com/Raynos/auth-stream)

***

# meta streams

## [mux-demux](https://github.com/dominictarr/mux-demux)

## [stream-router](https://github.com/Raynos/stream-router)

## [multi-channel-mdm](https://github.com/Raynos/multi-channel-mdm)

***

# state streams

## [crdt](https://github.com/dominictarr/crdt)

## [delta-stream](https://github.com/Raynos/delta-stream)

## [scuttlebutt](https://github.com/dominictarr/scuttlebutt)

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) pode ser utilizados para
um estado de sincronização de pessoa-para-pessoa (p2p) com uma malha topológica onde os nós
só podem estar conectados através de intermediários e não havera nó com uma versão oficial
de todo o dado.

O tipo de rede com distribuição de pessoa-para-pessoa 
[scuttlebutt](https://github.com/dominictarr/scuttlebutt) provem uma especialidade
util quando os nós em diferentes lados da rede beiram a necessidade de compartilhar e
atualizar o mesmo estado. Um exemplo deste tipo de rede podem ser clientes de navegação
que enviam mensagens através de um servidor http para outro é processa isso no bastidor
onde o navagador não tem uma conexão direta com um terceiro. Outro caso de uso seriam
os sistemas que abrangem redes internas utilizando IPv4 que são escassos.

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) uses a
[gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol)
to pass messages between connected nodes so that state across all the nodes will
[eventually converge](https://en.wikipedia.org/wiki/Eventual_consistency)
on the same value everywhere.

[scuttlebutt](https://github.com/dominictarr/scuttlebutt) usa um 
[gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol)
para passar mensagens entre nós conectados de modo que em todos os nós irão 
[eventualmente convergir](https://en.wikipedia.org/wiki/Eventual_consistency)
no mesmo valor em todas as partes.

Usando a interface `scuttlebutt/model`, nós podemos criar varios nós e canaliza-los
uns aos outros para criar qualquer tipo de rede que quisermos:

``` js
var Model = require('scuttlebutt/model');
var am = new Model;
var as = am.createStream();

var bm = new Model;
var bs = bm.createStream();

var cm = new Model;
var cs = cm.createStream();

var dm = new Model;
var ds = dm.createStream();

var em = new Model;
var es = em.createStream();

as.pipe(bs).pipe(as);
bs.pipe(cs).pipe(bs);
bs.pipe(ds).pipe(bs);
ds.pipe(es).pipe(ds);

em.on('update', function (key, value, source) {
    console.log(key + ' => ' + value + ' from ' + source);
});

am.set('x', 555);
```

A rede foi criada é forma um gráfico não-direcional que parece com isso:

```
a <-> b <-> c
      ^
      |
      v
      d <-> e
```

Nota-se que os nós `a` e `e` não estão diretamente conectados, mas quando executamos isse script:

```
$ node model.js
x => 555 from 1347857300518
```

o valor definido do nó `a` procura um caminho até o nó `e` que procura por `b` e
`d`. Aqui todos os nós estão no mesmo processo mas porque o [scuttlebutt](https://github.com/dominictarr/scuttlebutt)
usa uma simples interface de streaming, os nós são colocado em um processo ou servidos e
conectados com qualquer transporte de fluxo que pode lidar com dados do tipo `string`.

Depois você pode criar um exemplo mais realista conectando atráves de uma rede e
incrementando uma variavel de contador.

Aqui um servidor que define um contador com com valor inicial de 0 na variavel `count` e incrementa `count++`
a cada 320 milliseconds, imprimindo todas as atualizações do contador:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
m.set('count', '0');
m.on('update', function (key, value) {
    console.log(key + ' = ' + m.get('count'));
});

var server = net.createServer(function (stream) {
    stream.pipe(m.createStream()).pipe(stream);
});
server.listen(8888);

setInterval(function () {
    m.set('count', Number(m.get('count')) + 1);
}, 320);
```

Agora você pode criar um cliente que conecta-se como servidor, atualiza o contador em um
intervalo, e imprime todas as atualizações recebidas:

``` js
var Model = require('scuttlebutt/model');
var net = require('net');

var m = new Model;
var s = m.createStream();

s.pipe(net.connect(8888, 'localhost')).pipe(s);

m.on('update', function cb (key) {
    // wait until we've gotten at least one count value from the network
    if (key !== 'count') return;
    m.removeListener('update', cb);
    
    setInterval(function () {
        m.set('count', Number(m.get('count')) + 1);
    }, 100);
});

m.on('update', function (key, value) {
    console.log(key + ' = ' + value);
});
```

O cliente é um pouco mais complicado sabendo que ele tem que esperar por uma atualização de
alguem para começar a atulizar o próprio contador or descobrir que o dele é zero.

Uma vez que você tenha o servidor e alguns clientes executando você vera uma execução sequencial similar a esta:

```
count = 183
count = 184
count = 185
count = 186
count = 187
count = 188
count = 189
```

Ocasionalmente nos muitos nós você vera uma sequencia de valores repetidos assim:

```
count = 147
count = 148
count = 149
count = 149
count = 150
count = 151
```

Estes valores são devido a
[scuttlebutt's](https://github.com/dominictarr/scuttlebutt)
historico baseado no algoritimo de resolução de conflito que esta trabalhando duro para garantir os estados ao redor do sistema em todos os nós eventualmente consistente.

Nota-se que o servidor neste exempli é somente um outro nó com os mesmos
privilégios que os clientes conectados a ele. Os tempos "cliente" e "servidor" não
afetam como o estado de sincronização provem, somente quem iniciou a
conexão. Protocólos como este são chamados de protocólos simétricos.
Veja [dnode](https://github.com/substack/dnode) para outros exemplos de
um protocólo simétrico.

## [append-only](http://github.com/Raynos/append-only)

***

# http streams

## [request](https://github.com/mikeal/request)

## [oppressor](https://github.com/substack/oppressor)

## [response-stream](https://github.com/substack/response-stream)

***

# io streams

## [reconnect](https://github.com/dominictarr/reconnect)

## [kv](https://github.com/dominictarr/kv)

## [discovery-network](https://github.com/Raynos/discovery-network)

***

# parser streams

## [tar](https://github.com/creationix/node-tar)

## [trumpet](https://github.com/substack/node-trumpet)

## [JSONStream](https://github.com/dominictarr/JSONStream)

Use este módulo para analisar e traformar em texto puro dados de um fluxo.

Se você precisar passar uma grande coleção de json através de uma conexão lenta ou você
te um objeto json que ele popula lentamente um módulo fazendo uma analise incremental
dos dados quando eles chegam.

## [json-scrape](https://github.com/substack/json-scrape)

## [stream-serializer](https://github.com/dominictarr/stream-serializer)

***

# browser streams

## [shoe](https://github.com/substack/shoe)

## [domnode](https://github.com/maxogden/domnode)

## [sorta](https://github.com/substack/sorta)

## [graph-stream](https://github.com/substack/graph-stream)

## [arrow-keys](https://github.com/Raynos/arrow-keys)

## [attribute](https://github.com/Raynos/attribute)

## [data-bind](https://github.com/travis4all/data-bind)

***

# html streams

## [hyperstream](https://github.com/substack/hyperstream)


# audio streams

## [baudio](https://github.com/substack/baudio)

# rpc streams

## [dnode](https://github.com/substack/dnode)

[dnode](https://github.com/substack/dnode) faz uma chamada remota de funções
através de qualquer tipo de fluxo.

Aqui um basico servidor utilizando o dnode:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(s.replace(/[aeiou]{2,}/, 'oo').toUpperCase())
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

Em seguida você pode interceptar um cliente que esta fazendo uma chamada para o servidor utilizando a função `.transform()`:: 

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (s) {
        console.log('beep => ' + s);
        d.end();
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

Acionando o servidor, quando você executar o cliente provavelmente você vera:

```
$ node client.js
beep => BOOP
```

O cliente envia `'beep'` para o servidor utilizando a função `transform()` e
o servidor avisa o cliente já com o resultado, puro!

A interface de fluxo do dnode fornce aqui um fluxo duplex onde ambos o cliente e servidor
canalizando uns aos outros (`c.pipe(d).pipe(c)`) com requisições e respostas dos
dois lados.

A loucura do dnode incia-se quando você começa passando argumentos a uma função
que tem os callbacks arrancados. Aqui uma versão atualizada do servidor anterior com um
callback multi-estágio passando dança:

``` js
var dnode = require('dnode');
var net = require('net');

var server = net.createServer(function (c) {
    var d = dnode({
        transform : function (s, cb) {
            cb(function (n, fn) {
                var oo = Array(n+1).join('o');
                fn(s.replace(/[aeiou]{2,}/, oo).toUpperCase());
            });
        }
    });
    c.pipe(d).pipe(c);
});

server.listen(5004);
```

Aqui o cliente atualizado:

``` js
var dnode = require('dnode');
var net = require('net');

var d = dnode();
d.on('remote', function (remote) {
    remote.transform('beep', function (cb) {
        cb(10, function (s) {
            console.log('beep:10 => ' + s);
            d.end();
        });
    });
});

var c = net.connect(5004);
c.pipe(d).pipe(c);
```

Depois você gira o servidor, onde nós executamos o cliente que agora tem:

```
$ node client.js
beep:10 => BOOOOOOOOOOP
```

Ele simplesmente funciona!™

A ideia basica é essa onde você somente coloca funções em objetos e as chama do outro
lado de um fluxo e funções são arrancadas fora com uma outra finalização a fazer voltando
ao estado inicial. A melhor coisa é essa onde você passa funções para uma funcão que dispara
funções, essas funções são disparadas do *outro* lado!

Esta abordagem para disparar argumentos que são funções recursivamente será chamada adorávelmente
de "todas as tartarugas vão abaixo" que vai afinando. Os valores retornados de todas as nossas funções
são ignorados e são enumeradas em um objeto que é enviado no estilo json.

São tartarugas por todo o caminho abaixo!

![turtles all the way](http://substack.net/images/all_the_way_down.png)

Dnode trabalha no node or nos navegadores por meio de fluxos é totalmente facíl de
chamar funções definidas em qualquer lugar e especialmente util onde emparelham com 
[mux-demux](https://github.com/dominictarr/mux-demux) com multiplex em um fluxo rpc
para controlar um volumos fluxo de dados.

## [rpc-stream](https://github.com/dominictarr/rpc-stream)

***

# test streams

## [tap](https://github.com/isaacs/node-tap)

## [stream-spec](https://github.com/dominictarr/stream-spec)

***

# power combos

## distributed partition-tolerant chat

O módulo [acrescentar apenas](http://github.com/Raynos/append-only) pode nos dar uma
conveniente lista de itens acrescentados apenas com uma regra personalizada feito em cima do
[scuttlebutt](https://github.com/dominictarr/scuttlebutt)
que faz do trabalho de escrever algo realmente simples com uma eventualidade consistente, chat distribuido
pode replicar com outros nós e a fazer com que partições da rede sobreviva.

TODO: the rest

## faça o seu proprio socket.io

Nós podemos construir algo no estilo do socket.io com uma api de emição de eventos através de fluxos
usando muitas das libs mencionadas anteriormente neste documento.

Primeiro você usa o [tênis](http://github.com/substack/shoe) 
para criar um novo lidador de websocket do lado do servidor e um
[emissor de fluxo](https://github.com/substack/emit-stream)
para transformar um emissor de evento em um fluxo que emite objetos.
O fluxo de objeto pode ser alimentado assim
[JSONStream](https://github.com/dominictarr/JSONStream)
para serializer os objetos e de um fluxo serializado pode canalizar
para um navegador remoto.


``` js
var EventEmitter = require('events').EventEmitter;
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var sock = shoe(function (stream) {
    var ev = new EventEmitter;
    emitStream(ev)
        .pipe(JSONStream.stringify())
        .pipe(stream)
    ;
    ...
});
```

Dentro do callback do tênis nós podemos emitir eventos para a função `ev`. Até aqui tudo bem
somente emite diferentes tipos para eventos em intervalos:

``` js
var intervals = [];

intervals.push(setInterval(function () {
    ev.emit('upper', 'abc');
}, 500));

intervals.push(setInterval(function () {
    ev.emit('lower', 'def');
}, 300));

stream.on('end', function () {
    intervals.forEach(clearInterval);
});
```

Finalmente a instancia de tênis somente tem que estar vinculada a um servidor http:

``` js
var http = require('http');
var server = http.createServer(require('ecstatic')(__dirname));
server.listen(8080);

sock.install(server, '/sock');
```

Entretanto do lado do navegador para somente analisar coisas o fluxo do tênis esta em json e
passando o objeto resultando do fluxo para `eventStream()`. Somente `eventStream()` retorna um
emissor de eventos isso emite eventos do lado do servidor:

``` js
var shoe = require('shoe');
var emitStream = require('emit-stream');
var JSONStream = require('JSONStream');

var parser = JSONStream.parse([true]);
var stream = parser.pipe(shoe('/sock')).pipe(parser);
var ev = emitStream(stream);

ev.on('lower', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toLowerCase();
    document.body.appendChild(div);
});

ev.on('upper', function (msg) {
    var div = document.createElement('div');
    div.textContent = msg.toUpperCase();
    document.body.appendChild(div);
});
```

Use [browserify](https://github.com/substack/node-browserify) to build this
browser source code so that you can `require()` all these nifty modules
browser-side:

User o [browserify](https://github.com/substack/node-browserify) para construir
este código para o navegador então você podera usar `require()` para todos esses módulos
bacanas do lado do navegador:

```
$ browserify main.js -o bundle.js
```

Somente quando você colocar `<script src="/bundle.js"></script>` no html e executar em um
navagador para ver os eventos mostrando os fluxos desse lado e mostrando as coisas funcionando. 

Com essa abordagem de fluxo você pode depender de mais componentes reutilizaveis sendo 
necessario saber como os fluxos conversam. Em vez de mensagens roteadas através de um sistema evento
global no estilo do socket.io, você pode focar em mais em espadaçar a sua aplicação
em pequenas unidades de funcionalidade que fazem uma coisa bem.

Para cada instancia você pode trivialmente trocar JSONStram deste exemplo por
[stream-serializer](https://github.com/dominictarr/stream-serializer)
para pegar uma serialização diferente com uma configuração deiferente de compensações.
Você poderia fugir das camadas de cima do tênis para lidar com
[reconexões](https://github.com/dominictarr/reconnect) ou batimentos cardíacos
usando uma interface simples para os fluxos.
Você pode até mesmo adicionar um fluxo em cada espaço nomeado de eventos encadeados com
[eventemitter2](https://npmjs.org/package/eventemitter2) no lugar
do emissor de evento do núcleo do node.

Se você precisar de fluxos diferentes para agir de diferentes formas seria igualmente
mais simples executar o fluxo do tênis em um exemplo através do mux-demux para
criar canais separados para cada tipo diferente de fluxo que precisar.

Como os requerimentos do seu sistema envolvem através do tempo, você pode trocar
para cada pedaço necessario do fluxo com muitos ou nenhum risco
isso implica mais em uma abordagem opinativa de um framework.

## fluxos html para navegador e o servidor

Nós podemos usar muitos módulos de fluxo reutilizando a mesma lógica de processamentod o html para o
cliente e o servidor! Essa abordagem é indexavel, SEO-friendly, e nos dá
atulizações em tempo real.

Nosso processamento tem linhas de json como entrada e retornando `strings` de html
como saida. Texto, é uma interface universal!

render.js:

``` js
var through = require('through');
var hyperglue = require('hyperglue');
var fs = require('fs');
var html = fs.readFileSync(__dirname + '/static/row.html');

module.exports = function () {
    return through(function (line) {
        try { var row = JSON.parse(line) }
        catch (err) { return this.emit('error', err) }
        
        this.queue(hyperglue(html, {
            '.who': row.who,
            '.message': row.message
        }).outerHTML);
    });
};
```

We can use [brfs](http://github.com/substack/brfs) to inline the
`fs.readFileSync()` call for browser code
and [hyperglue](https://github.com/substack/hyperglue) to update html based on
css selectors. You don't need to use hyperglue necessarily here; anything that
can return a string with html in it will work.

Nós podemos usar [brfs](http://github.com/substack/brfs) para chamada em linha
`fs.readFileSync` para o código do navegador
e [hyperglue](https://github.com/substack/hyperglue) para atualizar o html baseado em
seletores css. Você não precisa usar a hipder-cóla necessariamente aqui; de qualquer maneira
pode-se retornar uma `string` com html que funcionara.

O `row.html` usado somente para arrancar isso:

row.html:

``` html
<div class="row">
  <div class="who"></div>
  <div class="message"></div>
</div>
```

O servidor usara somente [slice-file](https://github.com/substack/slice-file) para
manter tudo de modo simples. [slice-file](https://github.com/substack/slice-file) é
mais pequeno que uma api glorificada de `tail/tail -f` mas as insterfaces são bem mapeadas
para bancos de dados com resultados regulares mais as mudanças alimentadas como couchdb.

server.js:

``` js
var http = require('http');
var fs = require('fs');
var hyperstream = require('hyperstream');
var ecstatic = require('ecstatic')(__dirname + '/static');

var sliceFile = require('slice-file');
var sf = sliceFile(__dirname + '/data.txt');

var render = require('./render');

var server = http.createServer(function (req, res) {
    if (req.url === '/') {
        var hs = hyperstream({
            '#rows': sf.slice(-5).pipe(render())
        });
        hs.pipe(res);
        fs.createReadStream(__dirname + '/static/index.html').pipe(hs);
    }
    else ecstatic(req, res)
});
server.listen(8000);

var shoe = require('shoe');
var sock = shoe(function (stream) {
    sf.follow(-1,0).pipe(stream);
});
sock.install(server, '/sock');
```

A primeira parte do servidor lida com a rota `/` e fluxos das últimas 5 linhas
do `data.txt` na div `#rows`.

A segunda parte do servidor lida com atualizações em tempo real para `#rows` usando
[tênis](http://github.com/substack/shoe), um simples polyfill para websocket. 

O proximo passo sera escrever alguns código para nevegador que popule as atualizações
do [tênis](http://github.com/substack/shoe) na dic `#rows`:

``` js
var through = require('through');
var render = require('./render');

var shoe = require('shoe');
var stream = shoe('/sock');

var rows = document.querySelector('#rows');
stream.pipe(render()).pipe(through(function (html) {
    rows.innerHTML += html;
}));
```

Just compile with [browserify](http://browserify.org) and
[brfs](http://github.com/substack/brfs):

Apenas compile com [browserify](http://browserify.org) e 
[brfs](http://github.com/substack/brfs):

```
$ browserify -t brfs browser.js > static/bundle.js
```

E é isso! agora nós podemos popular `data.txt` com os dados:

```
$ echo '{"who":"substack","message":"beep boop."}' >> data.txt
$ echo '{"who":"zoltar","message":"COWER PUNY HUMANS"}' >> data.txt
```

gire o servidor:

```
$ node server.js
```

e navegue até `localhost:8000` onde você ira ver o nosso conteudo. Se nós adicionarmos
mais conteudo:

```
$ echo '{"who":"substack","message":"oh hello."}' >> data.txt
$ echo '{"who":"zoltar","message":"HEAR ME!"}' >> data.txt
```

quando a pagina atualizar automaticamente em tempo real, hooray!

Agora estamos usando a mesma lógica de processamento em ambos... cliente e o servidor
para servir conteudo do tipo SEO-friendly, indexavel e com conteudo em tempo real. Hooray!
